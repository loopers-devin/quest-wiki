---
week: 9
title: Redis ZSET 기반 실시간 랭킹 — Kafka Consumer · Ranking API
status: ready
---

# Week 9 — Redis ZSET 실시간 랭킹 (Kafka Consumer · Ranking API)

## 과제 개요

R7에서 구축한 Kafka 이벤트 파이프라인을 이어받아, `product_metrics` 적재와 함께 **Redis ZSET 기반 실시간 랭킹 파이프라인**을 구축한다. Consumer가 조회/좋아요/주문 이벤트를 컨슘해 일간 키(`ranking:all:{yyyyMMdd}`, TTL 2일)에 **가중치 × Score**로 점수를 누적하고, "오늘의 인기상품" API를 통해 페이지 조회와 상품 상세의 순위 정보를 제공한다.

- **Must-Have**: Redis ZSET, 실시간 랭킹 집계, Ranking API (페이지 조회 + 상품 상세 순위)
- **Nice-to-Have**: Kafka 배치 리스너, 실시간 Weight 조절, 시간 단위 랭킹, 콜드 스타트 완화(Score Carry-Over 스케줄러)

---

## 💻 구현 과제 채점 테이블

> **판정 원칙**: ★(필수) 항목 중 하나라도 빠지면 **FAIL**. ◯(일반)은 종합 판단. △(보너스)는 Rising Talent 신호.

### 📈 ① Ranking Consumer (Kafka → Redis ZSET)

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| RC1 | Kafka Consumer가 조회/좋아요/주문 이벤트를 컨슘해 Redis ZSET에 반영 | ★ | `@KafkaListener` + `ZINCRBY` |
| RC2 | 키 전략이 일간 기준 (`ranking:all:{yyyyMMdd}`) | ★ | 키 네이밍 유틸/상수 |
| RC3 | 키 TTL **2일** 설정 (초기 생성 시 `EXPIRE`) | ★ | `redisTemplate.expire(...)` |
| RC4 | 이벤트 종류별 **Weight** 및 **Score**가 명시적으로 정의됨 (상수/설정) | ★ | Enum/설정 클래스 or `application.yml` |
| RC5 | 주문 이벤트의 Score가 단순 count가 아닌 `price * amount` 등 도메인 기반 계산 | ★ | 계산 로직 존재 |
| RC6 | 날짜별 키 계산 유틸/함수 존재 (오늘/어제/특정 일자) | ★ | `keyOf(date)` 형태 |

### 🏆 ② Ranking API

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| RA1 | 랭킹 Page 조회 API (`GET /api/v1/rankings?date=yyyyMMdd&size=&page=`) | ★ | Controller |
| RA2 | `ZREVRANGE` 등으로 상위 N개 조회 후 페이징 처리 | ★ | Redis 조회 코드 |
| RA3 | 상품 ID만이 아닌 **상품 정보 aggregation** (이름, 가격, 이미지 등) | ★ | Product 조회 후 결합 |
| RA4 | 상품 상세 API 응답에 해당 상품의 **순위 정보 포함** (`ZREVRANK`) | ★ | 상세 응답 DTO에 `rank` 필드 |
| RA5 | 랭킹에 없는 상품은 순위 필드 `null` 반환 | ★ | null 처리 로직 |

### 🧪 ③ 검증

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| V1 | 이벤트 발행 → ZSET 반영 → API 조회까지 **E2E 흐름** 테스트 | ★ | 통합 테스트 |
| V2 | 일자가 바뀌어도 이전 날짜 랭킹 조회 정상 동작 (date 파라미터) | ★ | 어제 키 조회 테스트 |
| V3 | 가중치가 의도대로 반영됨 (e.g. 주문 1건 > 좋아요 3건) | ★ | 시나리오 테스트 |

### 🎁 ④ Nice-to-Have

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| NH1 | Kafka **배치 리스너**로 정제 후 ZSET 반영 (단건 처리 최소화) | △ | `setBatchListener(true)` + in-memory aggregate |
| NH2 | 실시간 Weight 조절 (설정 변경 → 재기동 없이 반영) | △ | ConfigMap/Redis/DB 기반 설정 |
| NH3 | 시간 단위 랭킹 (`ranking:hourly:{yyyyMMddHH}`) 구현 | △ | 별도 키 스킴 |
| NH4 | 콜드 스타트 완화 — 23:50 스케줄러로 다음날 랭킹판 사전 생성 (Score Carry-Over) | △ | `@Scheduled` + `ZUNIONSTORE` w/ weight |

### 🤖 AI 활용 (보너스)

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| AI1 | AI Skill로 랭킹 지표(가중치·TTL·롱테일·콜드스타트) 리스크를 리뷰한 흔적 | △ | PR/블로그 서술 |

---

## 🤖 자동 채점 룰 (스킬용)

### 구현 PASS 조건
- ★ 항목 모두 충족 → PASS
- Must-Have 3종이 모두 살아 있어야 함:
  - Ranking Consumer (RC1~RC6)
  - Ranking API (RA1~RA5)
  - 검증 (V1~V3)

### 구현 FAIL 조건
- Kafka Consumer가 ZSET에 반영하지 않음 (RC1) — 배치/스케줄러로 DB에서 다시 계산
- 키 전략이 일간이 아니거나 TTL 미설정 (RC2·RC3) — 누적 랭킹만 유지
- Weight/Score가 이벤트 타입 무관하게 균일 (RC4·RC5) — "좋아요 = 주문" 취급
- Ranking API가 없거나 상품 정보 aggregation 없음 (RA1·RA3)
- 상품 상세에 순위 정보 미포함 (RA4) — 상세와 랭킹이 분리된 채로만 존재
- E2E · 가중치 반영 검증 테스트 부재 (V1·V3)

### Rising Talent 신호 (사람 판단용)
- Weight/Score 설계에 **근거**(비즈니스 지표·정규화·log 스케일)가 서술됨
- 배치 리스너(NH1)로 ZSET/DB 연산 스루풋 개선을 실측
- 콜드 스타트 스케줄러(NH4)로 자정 랭킹 급락 문제 해결
- 시간 단위 랭킹(NH3)에서 슬라이딩 윈도우 or `ZUNIONSTORE` 활용
- 상위 N개만 유지하는 메모리 최적화 (`ZREMRANGEBYRANK`) 서술
- ZSET 조회를 매 요청마다 vs 주기적 캐싱 — 트레이드오프 실측
- 상품 정보 aggregation을 위해 N+1 방지 (batch fetch / 캐시)
- 시간의 양자화(quantization)와 랭킹 안정성에 대한 서술

---

## ✍️ 라이팅 과제 채점

### 제출 형식 (두 가지 중 택1 또는 둘 다)

1. **GitHub Issue (Tech Note)** — 4개 포맷 중 선택
   - 📐 Design Doc / 🪞 Retrospective / ⚔️ Challenge Story / 📊 Benchmark Report
2. **Blog** — 주차 학습/판단 정리, TL;DR 포함

> **참고**: 9주차는 지표 설계·성능 실측이 핵심이라 📐 Design Doc / 📊 Benchmark Report 포맷이 잘 어울린다.

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
  - 누적 랭킹만 유지하면 왜 롱테일 문제가 발생할까?
  - 시간의 양자화 — 왜 필요한가?
  - 콜드스타트(0점에서 시작) 문제를 어떻게 풀 수 있을까?
  - 우리의 랭킹 지표는 이렇게 구성돼요 — 진짜 인기있는 상품은 이런 것
  - 실시간 랭킹, 이렇게 풀면 쉽다
  - 상품이 10만 개일 때 ZSET 메모리는 얼마나 쓸까? 상위 N개만 유지하면?
  - Top-N을 매번 ZREVRANGE로 조회하는 것 vs 주기적 캐싱 — 트레이드오프
- 가중치 산정에 **비즈니스 근거**(전환율, 매출 기여도) 서술
- 자정 랭킹 급락(콜드 스타트) 실제 그래프/재현
- ZSET 메모리 실측 (redis-cli MEMORY USAGE) 기반 서술
