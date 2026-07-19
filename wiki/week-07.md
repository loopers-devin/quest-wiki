---
week: 7
title: 이벤트 기반 아키텍처 — ApplicationEvent · Kafka · 선착순 쿠폰
status: ready
---

# Week 7 — 이벤트 기반 아키텍처 (ApplicationEvent · Kafka · 선착순 쿠폰)

## 과제 개요

이벤트 기반 아키텍처의 **Why → How → Scale** 을 한 주에 관통한다. Spring `ApplicationEvent`로 **경계를 나누는 감각**을 익히고, Kafka로 **이벤트 파이프라인**(`commerce-api` → Kafka → `commerce-collector`)을 구축한 뒤, **선착순 쿠폰 발급**에 실전 적용한다. Transactional Outbox Pattern으로 At Least Once 발행을 보장하고, Consumer는 `event_handled` 기반 멱등 처리 + `version`/`updated_at` 기준 최신 이벤트만 반영한다.

- **Must-Have**: Event vs Command, ApplicationEvent, Kafka Producer/Consumer, Transactional Outbox Pattern, Kafka 기반 선착순 쿠폰
- **Nice-to-Have**: Consumer Group 분리, Consumer 배치 처리, DLQ

---

## 💻 구현 과제 채점 테이블

> **판정 원칙**: ★(필수) 항목 중 하나라도 빠지면 **FAIL**. ◯(일반)은 종합 판단. △(보너스)는 Rising Talent 신호.

### 🧾 ① ApplicationEvent (Step 1)

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| AE1 | 주문–결제 플로우에서 부가 로직(로깅/알림 등)을 `ApplicationEvent`로 분리 | ★ | `ApplicationEventPublisher.publishEvent(...)` |
| AE2 | 좋아요–집계 분리 (집계 실패와 무관하게 좋아요는 성공) | ★ | LikeService에서 이벤트 발행 후 별도 리스너 |
| AE3 | 유저 행동 로깅(조회/클릭/좋아요/주문)이 이벤트 리스너로 처리됨 | ★ | `@EventListener` 로깅 핸들러 |
| AE4 | 트랜잭션 결과와의 상관관계에 맞춘 `@TransactionalEventListener(phase=...)` 사용 | ★ | `AFTER_COMMIT` / `AFTER_ROLLBACK` 등 명시적 phase |
| AE5 | 이벤트 vs 명령(Command) 구분에 대한 판단 흔적 (이벤트명이 과거형·도메인 사실 표현) | ◯ | `OrderPlacedEvent`, `LikeAddedEvent` 등 네이밍 |

### 🎾 ② Kafka Producer / Consumer 기본 파이프라인 (Step 2)

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| K1 | `commerce-api` → Kafka → `commerce-collector` 구조 존재 | ★ | 별도 모듈 or 별도 Consumer 앱 |
| K2 | Producer 설정에 `acks=all` | ★ | `application.yml` producer 설정 |
| K3 | Producer 설정에 `enable.idempotence=true` | ★ | `application.yml` producer 설정 |
| K4 | PartitionKey를 도메인 기준으로 지정 (productId / orderId / couponId 등) | ★ | `send(topic, key, value)` |
| K5 | Consumer는 **manual Ack** 처리 (`AckMode.MANUAL_IMMEDIATE` 등) | ★ | `Acknowledgment.acknowledge()` 호출 |
| K6 | 토픽이 도메인 관심사별로 분리됨 (catalog / order / coupon 등) | ◯ | 최소 2개 이상 토픽 존재 |

### 📤 ③ Transactional Outbox Pattern (Step 2)

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| OB1 | Outbox 테이블 존재 (`event_id`, `aggregate_id`, `payload`, `status`, `created_at` 등) | ★ | DDL / Entity |
| OB2 | 도메인 트랜잭션과 **동일 트랜잭션**으로 Outbox 저장 | ★ | 서비스 메서드가 `@Transactional`로 도메인 + Outbox 함께 저장 |
| OB3 | 별도 Publisher(스케줄러/폴러/CDC)로 Outbox → Kafka 발행 | ★ | `@Scheduled` 폴러 or Debezium 등 |
| OB4 | 발행 성공 시 Outbox row 상태 전이 (PENDING → SENT) 또는 삭제 | ★ | 상태 컬럼 업데이트 로직 |
| OB5 | At Least Once 보장 근거 (실패 시 재발행, 발행 후 커밋 순서) | ◯ | 코드/문서 근거 |

### 📊 ④ Metrics 집계 Consumer (Step 2)

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| MC1 | Consumer가 이벤트를 수집해 `product_metrics`에 upsert | ★ | JPA `save` + unique key or native upsert |
| MC2 | 좋아요 수 / 판매량 / 조회 수 중 **최소 2종** 집계 | ★ | 이벤트 타입별 핸들러 |
| MC3 | `version` 또는 `updated_at` 기준으로 오래된 이벤트는 무시 | ★ | 비교 후 skip 로직 |

### 🔁 ⑤ 멱등 처리 (Step 2)

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| ID1 | `event_handled(event_id PK)` (DB 또는 Redis) 테이블/스토어 존재 | ★ | DDL / Redis key 스키마 |
| ID2 | Consumer 진입 시 `event_id` 존재 여부 체크 후 skip | ★ | 핸들러 첫 라인에서 확인 |
| ID3 | 처리 완료 후 `event_handled` 기록 (같은 트랜잭션 or 안전한 순서) | ★ | 처리 로직 + insert |
| ID4 | 이벤트 핸들링 테이블과 로그 테이블을 **분리**한 판단 흔적 | ◯ | PR/블로그/주석 서술 |

### 🎫 ⑥ Kafka 기반 선착순 쿠폰 발급 (Step 3)

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| CP1 | 쿠폰 발급 요청 API가 **Kafka 발행만** 하고 즉시 반환 (비동기) | ★ | Controller → Producer, DB 직접 발급 X |
| CP2 | Consumer에서 실제 쿠폰 발급 처리 | ★ | `@KafkaListener` 발급 핸들러 |
| CP3 | 발급 수량 제한 동시성 제어 (수량 초과 X) | ★ | DB unique/락 · Redis atomic · 원자적 감소 등 |
| CP4 | 중복 발급 방지 (같은 유저 · 같은 요청) | ★ | userId+couponId unique or 멱등 키 |
| CP5 | 발급 결과 확인 구조 존재 (polling API or 콜백/알림) | ★ | `GET /api/v1/coupons/issue-requests/{id}` 등 |
| CP6 | 동시성 테스트로 수량 초과 미발생 검증 | ★ | 통합/부하 테스트 코드 |

### 🎁 ⑦ Nice-to-Have

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| NH1 | Consumer Group을 관심사별로 분리 (집계용 / 알림용 / 감사로그용 등) | △ | 서로 다른 `group.id` |
| NH2 | Consumer 배치 처리 (`batchListener=true`, `max.poll.records`) | △ | `ConcurrentKafkaListenerContainerFactory.setBatchListener(true)` |
| NH3 | DLQ 구성 (재시도 실패 이벤트 격리) | △ | `DeadLetterPublishingRecoverer` / `.DLT` 토픽 |

### 🤖 AI 활용 (보너스)

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| AI1 | AI Skill로 이벤트 경계·Outbox·멱등 설계 리스크를 리뷰한 흔적 | △ | PR/블로그에 트랜잭션 경계·중복 처리·순서 보장 검토 |

---

## 🤖 자동 채점 룰 (스킬용)

### 구현 PASS 조건
- ★ 항목 모두 충족 → PASS
- Must-Have 5종이 모두 살아 있어야 함:
  - ApplicationEvent (AE1~AE4)
  - Kafka Producer/Consumer 기본 (K1~K5)
  - Transactional Outbox (OB1~OB4)
  - Metrics 집계 Consumer (MC1~MC3) + 멱등 처리 (ID1~ID3)
  - 선착순 쿠폰 (CP1~CP6)

### 구현 FAIL 조건
- ApplicationEvent 분리 없음 (AE1~AE3) — 부가 로직이 여전히 주 트랜잭션에 붙어 있음
- `@TransactionalEventListener` phase 미사용 (AE4) — 커밋 전후 관계 무시
- Producer 설정 누락 (K2·K3) — `acks=all` 또는 `idempotence=true` 미설정
- manual Ack 미사용 (K5) — auto commit에 의존
- Transactional Outbox 미구현 (OB1·OB2) — 도메인과 별개 트랜잭션으로 Kafka 직접 발행
- 멱등 처리 없음 (ID1~ID3) — 중복 소비 시 집계·발급 중복 발생
- 쿠폰 발급 API가 동기 처리 (CP1) — Kafka 발행 없이 DB 직접 발급
- 쿠폰 수량 제한 미구현 (CP3) 또는 동시성 테스트 없음 (CP6) — 수량 초과 발급 가능

### Rising Talent 신호 (사람 판단용)
- 이벤트 vs 명령 구분(AE5)에 대한 판단 근거가 코드·글로 명확히 드러남
- Outbox 발행 실패/중복 발행 시나리오까지 테스트로 커버 (OB5)
- `version`/`updated_at` 기반 최신 이벤트 반영이 실제 out-of-order 시나리오로 검증됨 (MC3)
- 이벤트 핸들링 테이블과 로그 테이블을 분리한 이유 서술 (ID4)
- 선착순 쿠폰의 수량 제한을 DB / Redis / Kafka 파티셔닝 중 어느 계층에서 처리할지 트레이드오프 서술
- Consumer Group 분리(NH1) · 배치(NH2) · DLQ(NH3) 중 2개 이상 구현
- Redis 기반 발급과 Kafka 기반 발급의 실제 벤치마크
- "성공했지만 응답 유실" · "consumer가 중복 소비" 등 실패 시나리오 복구 로직

---

## ✍️ 라이팅 과제 채점

### 제출 형식 (두 가지 중 택1 또는 둘 다)

1. **GitHub Issue (Tech Note)** — 4개 포맷 중 선택
   - 📐 Design Doc / 🪞 Retrospective / ⚔️ Challenge Story / 📊 Benchmark Report
2. **Blog** — 주차 학습/판단 정리, TL;DR 포함

> **참고**: 7주차는 설계 판단(경계 나누기)이 핵심이라 📐 Design Doc / 📊 Benchmark Report 포맷이 잘 어울린다.

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
  - ApplicationEvent만으로 충분한 경계 vs Kafka가 필요한 경계, 그 기준은?
  - 트랜잭션 안에 다 넣을 수 있지만, 굳이 나누는 이유
  - 좋아요는 동기, 집계는 비동기 — 상품의 좋아요 수가 바로 반영되어야 할까?
  - Outbox Pattern 없이 Kafka만 쓰면 어떤 일이 벌어질까?
  - 선착순 쿠폰을 Redis로 처리하는 것과 Kafka로 처리하는 것의 차이
  - 100장 한정 쿠폰에 1만 명이 동시에 요청하면?
  - 멱등 처리를 DB로 할 때와 Redis로 할 때의 트레이드오프
- 이벤트명이 도메인 사실을 표현하는지(과거형 · 부작용 없음)에 대한 서술
- Outbox 폴러 지연 vs CDC(Debezium) 트레이드오프 서술
- Consumer의 out-of-order 처리(version/updated_at)를 실제 사례로 설명
