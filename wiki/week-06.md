---
week: 6
title: 외부 시스템 연동 — Resilience (Timeout · Fallback · CircuitBreaker)
status: ready
---

# Week 6 — 외부 시스템 연동 & Resilience (Timeout · Fallback · CircuitBreaker)

## 과제 개요

외부 결제(PG) 시스템 장애·지연을 전제로 한 **Resilience 설계**를 학습·적용한다. 로컬 `pg-simulator` 모듈을 활용해 비동기 결제(요청 60% 성공, 처리 지연 1~5s, 처리 결과 성공 70% / 한도초과 20% / 잘못된카드 10%) 흐름을 다루고, **타임아웃 · 폴백 · 서킷브레이커 · 콜백/상태 확인 API 복구**를 통해 외부 장애가 내부 시스템 전체를 무너뜨리지 않도록 방어한다.

- **Must-Have**: Fallback / Timeout / CircuitBreaker
- **Nice-to-Have**: Retryer

---

## 💻 구현 과제 채점 테이블

> **판정 원칙**: ★(필수) 항목 중 하나라도 빠지면 **FAIL**. ◯(일반)은 종합 판단. △(보너스)는 Rising Talent 신호.

### 🔌 ① PG 연동 API

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| P1 | `POST /api/v1/payments` 엔드포인트 존재 (orderId, cardType, cardNo 입력) | ★ | Controller / API 문서 |
| P2 | RestTemplate / FeignClient / WebClient 로 `pg-simulator` 호출 | ★ | HttpClient 설정 / `@FeignClient` |
| P3 | PG 요청 payload에 `orderId, cardType, cardNo, amount, callbackUrl` 포함 | ★ | 요청 DTO |
| P4 | PG 콜백 수신 엔드포인트 존재 | ★ | 예: `POST /api/v1/examples/callback` |

### ⏱ ② Timeout

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| T1 | Connect / Read timeout이 명시적으로 설정됨 | ★ | `RestTemplateBuilder.setConnectTimeout(...)` / `feign.client.config.*.readTimeout` |
| T2 | 타임아웃 값이 PG 요청 지연(100~500ms)에 대해 합리적 근거를 가짐 | ★ | PR/블로그에 근거 서술 or 코드 주석 |
| T3 | 타임아웃 발생 시 500 전파가 아니라 예외 처리 → 폴백/재조회 흐름으로 유도 | ★ | try-catch, `@ControllerAdvice`, fallback 메서드 |

### 🛡 ③ Fallback

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| F1 | PG 호출 실패 시 내부 API는 정상 응답 (5xx 대량 배출 X) | ★ | fallback 메서드 존재 |
| F2 | 결제 상태를 `PENDING / FAILED / SUCCESS` 등 상태값으로 남겨 조회·복구 가능 | ★ | Payment status enum + 저장 로직 |
| F3 | Fallback 시 주문 상태 전이가 정합적 (자동 롤백 · 보류 · 결제 대기 중 하나로 명확히 결정) | ★ | Order 상태 전이 코드 |

### 🔌⚡ ④ CircuitBreaker

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| CB1 | Resilience4j 등 CircuitBreaker 라이브러리 도입 | ★ | `build.gradle` 의존성 + `@CircuitBreaker` |
| CB2 | PG 호출부에 CircuitBreaker 어노테이션/래핑 적용 | ★ | `@CircuitBreaker(name="pg", fallbackMethod=...)` |
| CB3 | Fallback 메서드가 정의되고 서비스가 정상 응답을 반환 | ★ | fallback 시그니처 존재 |
| CB4 | `slidingWindow`, `failureRateThreshold`, `waitDurationInOpenState` 등 설정을 기본값이 아니라 의도적으로 조정 | ◯ | `application.yml` resilience4j 설정 |

### 📞 ⑤ 콜백 + 상태 확인 API 복구

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| R1 | PG 콜백 수신 시 결제 상태 업데이트 로직 존재 | ★ | 콜백 handler → Payment 저장 |
| R2 | 결제 상태 확인 API 호출 로직 존재 (`GET /api/v1/payments/{txId}` 또는 `?orderId=`) | ★ | PG 조회 클라이언트 메서드 |
| R3 | 스케줄러 · 배치 · 관리자 API 중 **하나 이상**으로 미완료 결제 복구 가능 | ★ | `@Scheduled` / 관리자 endpoint |
| R4 | 타임아웃/서킷 오픈으로 요청이 실패해도 상태 확인 API로 결제건이 결국 시스템에 반영됨 | ★ | R1~R3가 실제로 연결된 흐름 |

### 🔁 ⑥ Retryer (Nice-to-Have)

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| RT1 | Retry 정책 적용 (Spring Retry / Resilience4j Retry / Feign Retryer) | △ | `@Retry`, `RetryTemplate` |
| RT2 | 재시도가 멱등성 관점에서 안전 (orderId / 클라이언트 txId 기반 중복 방지) | △ | Idempotency-Key or DB unique 제약 |

### 🤖 AI 활용 (보너스)

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| AI1 | `analyze-external-integration` 등 Skill로 외부 연동 설계 리스크를 리뷰한 흔적 | △ | PR/블로그에 트랜잭션 경계·상태 불일치·장애 시나리오 검토 |

---

## 🤖 자동 채점 룰 (스킬용)

### 구현 PASS 조건
- ★ 항목 모두 충족 → PASS
- Must-Have 3종이 모두 살아 있어야 함:
  - Timeout(T1~T3), Fallback(F1~F3), CircuitBreaker(CB1~CB3)
- 콜백 + 상태 확인 API 복구(R1~R4)까지 구성됨

### 구현 FAIL 조건
- 타임아웃 미설정 (T1) — 기본값에 의존
- PG 호출 실패 시 내부 5xx 배출 (F1) — fallback 없음
- CircuitBreaker 미도입 (CB1)
- 콜백(R1) · 상태 확인 API 호출(R2) 둘 다 없음 → 결과 복구 경로 부재
- 결제 상태 저장 자체가 없음 (F2) → 폴백/복구 자체가 불가능

### Rising Talent 신호 (사람 판단용)
- CB4가 정교하게 튜닝됨 (sliding window, failure rate threshold, half-open 전이 근거 명시)
- RT2 멱등성 완비 (Idempotency-Key, DB unique 제약, 재시도 시 중복 결제 방지)
- 결제 상태 재확인 **스케줄러 + 관리자 수동 트리거 API** 병행
- 외부 호출을 **트랜잭션 밖으로 분리**한 흔적 (트랜잭션 경계 관리)
- pg-simulator의 지연/실패 시나리오를 **통합 테스트**로 커버
- "성공했지만 응답 유실" 케이스에 대한 복구 로직이 명확히 존재

---

## ✍️ 라이팅 과제 채점

### 제출 형식 (두 가지 중 택1 또는 둘 다)

1. **GitHub Issue (Tech Note)** — 4개 포맷 중 선택
   - 📐 Design Doc / 🪞 Retrospective / ⚔️ Challenge Story / 📊 Benchmark Report
2. **Blog** — 주차 학습/판단 정리, TL;DR 포함

> **참고**: 6주차는 장애·복구 서사가 많아 🪞 Retrospective / ⚔️ Challenge Story 포맷이 잘 어울린다.

### PASS 조건 (기본)
- 블로그 또는 Issue 링크 제출 + 접근 가능 + **100자 초과** → PASS

### FAIL 조건 (엄격 적용)
- 미제출 (링크 없음)
- 링크 접근 불가 (404, 권한, 비공개 등)
- 내용이 사실상 없음 (100자 이하, 빈 글)

### 주차별 추가 기준
- (없음 — 기본 규칙만 적용)

### Rising Talent 신호 (사람 판단용)
- Timeout 값 / CircuitBreaker 임계치 선택 **근거**가 서술됨 (숫자만 나열 X)
- "결제 성공했는데 응답 유실" 케이스에서의 정합성 처리 서술
- 폴백 정책 (주문 롤백 vs 보류 vs Pending 유지)의 트레이드오프 서술
- 추천 주제 중 깊이 있게 다룬 경우:
  - PG 응답이 느려서 서킷브레이커가 열렸다
  - 응답이 안 와서 실패 처리했는데 PG에선 결제가 됐다
  - 주문 상태는 Pending 인데 사용자는 결제 안내를 받았다
  - PG 장애 하나로 주문 전체가 멈춰버렸다
  - 결제가 실패하면 주문을 무조건 롤백해야 할까?
  - 재시도 횟수는 몇 번이 적절했을까?
  - 폴백 처리를 어떻게 했지?
