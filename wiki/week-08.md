---
week: 8
title: Redis 대기열 — Sorted Set · 입장 토큰 · 실시간 순번 조회
status: ready
---

# Week 8 — Redis 대기열 (Sorted Set · 입장 토큰 · 실시간 순번 조회)

## 과제 개요

트래픽이 폭증하는 순간에도 시스템을 보호하면서, **유저에게 공정한 대기 경험**을 제공하는 구조를 설계·구현한다. Redis Sorted Set 기반 대기열로 **진입 순서를 보장**하고 중복 진입을 막고, 스케줄러가 처리량 기준(DB 커넥션 풀·평균 처리 시간)에 맞춰 **입장 토큰(TTL)** 을 발급한다. 유저는 Polling으로 **자기 순번과 예상 대기 시간**을 조회할 수 있으며, 토큰 보유자만 주문 API에 진입한다. 주문 API 이후 흐름(이벤트·Kafka·Metrics)은 R7 구조를 그대로 활용한다.

- **Must-Have**: Redis Sorted Set 대기열, 입장 토큰 발급/검증(TTL), 스케줄러 기반 순차 입장, Polling 순번 조회
- **Nice-to-Have**: SSE 실시간 Push, Polling 주기 동적 조절, Thundering Herd 완화(Jitter), Redis 장애 시 Graceful Degradation

---

## 💻 구현 과제 채점 테이블

> **판정 원칙**: ★(필수) 항목 중 하나라도 빠지면 **FAIL**. ◯(일반)은 종합 판단. △(보너스)는 Rising Talent 신호.

### 🚪 ① 대기열 진입 (Step 1)

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| Q1 | 대기열 진입 API 존재 (`POST /queue/enter` 등) | ★ | Controller |
| Q2 | Redis **Sorted Set** 기반으로 진입 순서 보장 (score = enqueue timestamp) | ★ | `ZADD queue:{key} score userId` |
| Q3 | userId 기반 **중복 진입 방지** (이미 대기 중이면 신규 등록 X) | ★ | `ZSCORE` 체크 or `ZADD NX` |
| Q4 | 전체 대기 인원 조회 (`ZCARD` 등) | ★ | 관리자 API 또는 순번 조회 응답에 포함 |

### 🎫 ② 입장 토큰 & 스케줄러 (Step 2)

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| T1 | 스케줄러가 주기적으로 대기열에서 **N명씩** 꺼내 입장 토큰 발급 | ★ | `@Scheduled` + `ZPOPMIN`/`ZRANGE`+`ZREM` |
| T2 | 입장 토큰에 **TTL** 부여 (e.g. 5분) | ★ | `SET token:{...} ... EX 300` |
| T3 | 주문 API 진입 시 토큰 검증 (헤더/쿠키에서 토큰 추출 후 존재/유효 확인) | ★ | 인터셉터/필터/`@PreAuthorize` |
| T4 | 주문 완료 후 토큰 삭제 (재사용 방지) | ★ | `DEL token:{...}` |
| T5 | 스케줄러 배치 크기 산정 **근거 문서화** (DB 커넥션 풀, 평균 처리 시간 등) | ★ | PR/블로그/README 서술 |
| T6 | 토큰 발급 시 대기열에서 해당 유저 제거 (Sorted Set에서 pop) | ◯ | `ZPOPMIN` 또는 pop 후 `ZREM` |

### 📡 ③ 실시간 순번 조회 (Step 3)

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| P1 | 순번 조회 API 존재 (`GET /queue/position` 등) | ★ | Controller |
| P2 | 현재 순번 반환 (`ZRANK`) | ★ | `ZRANK queue:{key} userId` |
| P3 | 예상 대기 시간 계산 로직 (순번 / 배치 크기 × 배치 주기 등) | ★ | 서비스 계산 코드 |
| P4 | Polling 기반 응답 구조 (순번 + 예상 대기 시간 + 토큰 발급 여부) | ★ | 응답 DTO에 3종 포함 |
| P5 | 토큰이 발급된 유저는 순번 조회 응답에 **토큰 포함**해 반환 | ★ | 응답 DTO의 `token` 필드 |

### 🧪 ④ 검증

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| V1 | 동시 진입 테스트 (대기열 순서 정확성 검증) | ★ | 통합/부하 테스트 코드 |
| V2 | 토큰 만료 테스트 (TTL 초과 시 무효화) | ★ | `Awaitility` / `Thread.sleep` 테스트 |
| V3 | 처리량 초과 테스트 (배치 크기 초과 요청에도 시스템 안정) | ★ | 부하 테스트 결과 |

### 🎁 ⑤ Nice-to-Have

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| NH1 | SSE 기반 실시간 순번 Push | △ | `SseEmitter` / `Flux<ServerSentEvent>` |
| NH2 | Polling 주기 동적 조절 (순번 구간별로 클라이언트에 `retryAfter` 안내) | △ | 응답 헤더/필드에 폴링 간격 |
| NH3 | Thundering Herd 완화 (토큰 발급 간격 분산 / Jitter) | △ | 스케줄러 로직에 랜덤 지연 |
| NH4 | Graceful Degradation (Redis 장애 시 Fallback — bypass 또는 안내 페이지) | △ | try-catch + fallback 흐름 |

### 🤖 AI 활용 (보너스)

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| AI1 | AI Skill로 대기열 설계 리스크(중복 진입/토큰 재사용/스케줄러 배치)를 리뷰한 흔적 | △ | PR/블로그 검토 |

---

## 🤖 자동 채점 룰 (스킬용)

### 구현 PASS 조건
- ★ 항목 모두 충족 → PASS
- Must-Have 4종이 모두 살아 있어야 함:
  - Redis Sorted Set 대기열 (Q1~Q4)
  - 입장 토큰 & 스케줄러 (T1~T5)
  - Polling 순번 조회 (P1~P5)
  - 검증 (V1~V3)

### 구현 FAIL 조건
- Sorted Set 미사용 (Q2) — List/Set/DB로 순서 보장 시도
- 중복 진입 방지 없음 (Q3) — 같은 userId가 여러 번 대기열에 쌓임
- 토큰 TTL 없음 (T2) — 만료 없이 영구 유효
- 주문 API에서 토큰 검증 없음 (T3) — 대기열이 사실상 관문 역할 X
- 스케줄러 배치 크기 근거 없음 (T5) — 임의 상수로 처리량 산정
- 순번 조회 API 없음 (P1~P2) — 유저가 자기 순번을 알 수 없음
- 동시 진입 · 토큰 만료 검증 테스트 부재 (V1~V2)

### Rising Talent 신호 (사람 판단용)
- 배치 크기(T5) 산정에 실제 부하 측정 결과가 반영됨 (커넥션 풀·p99 latency 근거)
- Thundering Herd(NH3)를 재현하고 Jitter로 완화한 실측 결과
- SSE(NH1)와 Polling의 트레이드오프를 실제로 비교
- Redis 장애 시나리오(NH4) — bypass 모드 · 서킷브레이커 · 안내 페이지 등 실제 fallback 흐름
- 토큰 재사용 방지(T4) 외에 재발급/취소 시나리오까지 다룸
- Polling 주기(NH2)를 순번 구간별로 조절해 서버 부하 실측 개선
- 대기열 → 토큰 → 주문 → R7 이벤트 파이프라인의 **end-to-end 흐름** 문서화
- 공정성(FIFO) vs 우선순위(VIP) 트레이드오프 서술

---

## ✍️ 라이팅 과제 채점

### 제출 형식 (두 가지 중 택1 또는 둘 다)

1. **GitHub Issue (Tech Note)** — 4개 포맷 중 선택
   - 📐 Design Doc / 🪞 Retrospective / ⚔️ Challenge Story / 📊 Benchmark Report
2. **Blog** — 주차 학습/판단 정리, TL;DR 포함

> **참고**: 8주차는 처리량 산정/부하 실측이 핵심이라 📊 Benchmark Report / 📐 Design Doc 포맷이 잘 어울린다.

### PASS 조건 (기본)
- 블로그 또는 Issue 링크 제출 + 접근 가능 + **100자 초과** → PASS

### FAIL 조건 (엄격 적용)
- 미제출 (링크 없음)
- 링크 접근 불가 (404, 권한, 비공개 등)
- 내용이 사실상 없음 (100자 이하, 빈 글)

### 주차별 추가 기준
- (없음 — 기본 규칙만 적용)

### Rising Talent 신호 (사람 판단용)
- 추천 주제 중 깊이 있게 다룬 경우:
  - Rate Limiting으로 거부하는 것과 대기열로 줄 세우는 것, 어떤 상황에서 어떤 전략이 맞을까?
  - 스케줄러 배치 크기를 어떻게 산정했는가? 그 근거는?
  - Thundering Herd를 직접 겪었다면, 어떻게 완화했는가?
  - Redis가 죽으면 우리 서비스는 어떻게 되어야 하는가?
  - Polling vs SSE — 왜 그 방식을 선택했는가?
  - 토큰 TTL을 몇 분으로 설정했고, 그 기준은 무엇인가?
- 배치 크기 산정에 **숫자 근거**(커넥션 풀 · p99 · 목표 TPS)가 있는 서술
- 폴링 주기와 서버 부하의 관계를 실측/그래프로 정리
- 대기열의 공정성(FIFO) 관점에서 예외 케이스(재접속, 세션 유실, VIP) 논의
