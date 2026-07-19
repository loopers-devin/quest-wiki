---
week: 10
title: Spring Batch — 주간·월간 랭킹 Materialized View
status: ready
---

# Week 10 — Spring Batch 주간·월간 랭킹 (Materialized View)

## 과제 개요

R9에서 적재한 `product_metrics` 일간 집계 데이터를 기반으로, **Spring Batch**를 활용해 대규모 데이터를 집계하는 조회 전용 구조를 설계·구현한다. Chunk-Oriented Processing으로 하루치 메트릭을 읽어 **주간·월간 Materialized View**(`mv_product_rank_weekly`, `mv_product_rank_monthly`)에 TOP 100 랭킹을 저장하고, 기존 Ranking API를 확장해 **일간/주간/월간** 랭킹을 기간 파라미터로 제공한다.

- **Must-Have**: Spring Batch Job, Chunk-Oriented Batch Processing, Materialized View (Statistics)
- **Nice-to-Have**: Partition/Multi-thread Step, Job 파라미터 유효성/멱등성, 실패 재실행 전략, 부분 재집계

---

## 💻 구현 과제 채점 테이블

> **판정 원칙**: ★(필수) 항목 중 하나라도 빠지면 **FAIL**. ◯(일반)은 종합 판단. △(보너스)는 Rising Talent 신호.

### 🧱 ① Spring Batch Job

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| B1 | Spring Batch 의존성 및 Job 설정 존재 (`@EnableBatchProcessing` or Boot 3.x 자동설정) | ★ | `build.gradle` + Job 빈 |
| B2 | Job이 **파라미터 기반**으로 동작 (기준일자·기간 등을 `JobParameters`로 수신) | ★ | `JobParametersBuilder` / `@Value("#{jobParameters[...]}")` |
| B3 | **Chunk-Oriented** 구성 — `ItemReader` / `ItemProcessor` / `ItemWriter` 존재 | ★ | `StepBuilder.chunk(N)` |
| B4 | 대상 테이블은 `product_metrics` (일간 집계 데이터) | ★ | Reader의 쿼리/엔티티 |
| B5 | Chunk 크기가 명시적으로 설정됨 (임의 상수 아님, 근거 있는 값) | ◯ | 설정값 + 주석/문서 |
| B6 | Job 실행 이력이 Spring Batch 메타 테이블에 정상 기록 (`BATCH_JOB_EXECUTION` 등) | ◯ | 메타 테이블 존재 |

### 🗂 ② Materialized View 설계

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| MV1 | `mv_product_rank_weekly` 테이블 존재 (주간 TOP 100) | ★ | DDL / Entity |
| MV2 | `mv_product_rank_monthly` 테이블 존재 (월간 TOP 100) | ★ | DDL / Entity |
| MV3 | MV 스키마에 최소 `product_id`, `rank`, `score`, `period_start`, `period_end` 포함 | ★ | 컬럼 정의 |
| MV4 | Writer가 MV에 정확히 **TOP 100**만 적재 (초과분 제외) | ★ | Writer 로직 or SQL LIMIT |
| MV5 | 재실행 시 동일 기간 데이터는 upsert 또는 삭제 후 재적재로 **멱등** 처리 | ★ | `MERGE` / delete-then-insert |

### 🧩 ③ Ranking API 확장

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| RA1 | 기존 `GET /api/v1/rankings` 가 **기간 파라미터**(daily/weekly/monthly)를 수신 | ★ | `?period=` 또는 별도 경로 |
| RA2 | 일간은 R9의 Redis ZSET, 주간·월간은 MV 조회로 라우팅 | ★ | 서비스 분기 로직 |
| RA3 | 주간·월간 랭킹 응답이 상품 정보 aggregation 포함 (이름/가격/이미지 등) | ★ | 상품 조회 join/batch fetch |
| RA4 | 페이지네이션(size, page) 정상 동작 | ★ | LIMIT/OFFSET or Pageable |

### 🧪 ④ 검증

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| V1 | Job 파라미터를 바꿔 여러 기간을 실행 가능 (통합 테스트 or 수동 실행 결과) | ★ | `JobLauncherTestUtils` / 실행 로그 |
| V2 | 동일 기간 재실행 시 MV가 중복 적재되지 않음 (MV5 검증) | ★ | 재실행 후 count 동일 |
| V3 | API로 daily/weekly/monthly 조회 시 각기 다른 소스에서 정상 응답 | ★ | E2E 테스트 |

### 🎁 ⑤ Nice-to-Have

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| NH1 | Partition Step 또는 Multi-threaded Step으로 병렬 처리 | △ | `PartitionHandler` / `TaskExecutor` |
| NH2 | 실패 시 재실행 전략 (Skip / Retry / Restart) 명시적 설정 | △ | `.faultTolerant().retryLimit(...)` |
| NH3 | 부분 재집계(특정 상품·특정 일자만) 가능한 파라미터 설계 | △ | `JobParameters` 확장 |
| NH4 | MV 스키마에 인덱스 설계 (조회 패턴 기반) | △ | DDL 인덱스 정의 |

### 🤖 AI 활용 (보너스)

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| AI1 | AI Skill로 배치 설계 리스크(멱등·chunk 크기·MV 스키마) 리뷰 흔적 | △ | PR/블로그 서술 |

---

## 🤖 자동 채점 룰 (스킬용)

### 구현 PASS 조건
- ★ 항목 모두 충족 → PASS
- Must-Have 3종이 모두 살아 있어야 함:
  - Spring Batch Job (B1~B4)
  - Materialized View (MV1~MV5)
  - Ranking API 확장 (RA1~RA4)
  - 검증 (V1~V3)

### 구현 FAIL 조건
- Spring Batch 미도입 (B1) — 단순 스케줄러/서비스 메서드로 처리
- Chunk-Oriented 구성 아님 (B3) — Tasklet 하나로 전체 처리하며 대량 데이터 실패 위험
- Job 파라미터 미사용 (B2) — 하드코딩된 일자로만 동작
- MV 테이블 부재 (MV1·MV2) — 조회 전용 구조 없이 매번 원본 집계
- MV 재실행 시 중복 적재 (MV5) — 멱등성 없음
- API가 기간 확장을 지원하지 않음 (RA1) — daily만 제공
- 주간·월간 조회 시 상품 정보 aggregation 없음 (RA3)

### Rising Talent 신호 (사람 판단용)
- Chunk 크기(B5) 산정에 실제 처리 시간/메모리 실측 근거
- Partition/Multi-thread Step(NH1)으로 배치 처리 시간 개선 실측
- 실패 재실행 전략(NH2)이 Skip/Retry/Restart 시나리오별로 나뉨
- 부분 재집계(NH3)로 데이터 오류 복구 시나리오 대응
- MV 인덱스(NH4)를 조회 패턴 기반으로 설계 (커버링 인덱스 등)
- MV vs 실시간 집계 트레이드오프 서술 (지연 허용치 · 저장 비용)
- 대량 데이터 실측 결과(수백만 rows) 기반 서술
- 자정 배치 · 시간대 · DST 등 시간 처리 엣지케이스 고려

---

## ✍️ 라이팅 과제 채점

### 제출 형식 (두 가지 중 택1 또는 둘 다)

1. **GitHub Issue (Tech Note)** — 4개 포맷 중 선택
   - 📐 Design Doc / 🪞 Retrospective / ⚔️ Challenge Story / 📊 Benchmark Report
2. **Blog** — 주차 학습/판단 정리, TL;DR 포함

> **참고**: 10주차는 부트캠프 마지막 회차로, **10주 여정 회고** 성격의 🪞 Retrospective 포맷이 잘 어울린다. 배치 성능 실측을 다룬다면 📊 Benchmark Report도 좋다.

### PASS 조건 (기본)
- 블로그 또는 Issue 링크 제출 + 접근 가능 + **100자 초과** → PASS

### FAIL 조건 (엄격 적용)
- 미제출 (링크 없음)
- 링크 접근 불가 (404, 권한, 비공개 등)
- 내용이 사실상 없음 (100자 이하, 빈 글)

### 주차별 추가 기준
- (없음 — 기본 규칙만 적용)

### Rising Talent 신호 (사람 판단용)
- **10주 여정 회고**로서의 완성도:
  - 1~10주차 흐름이 단순 나열이 아니라 **연결된 서사**로 정리됨
  - 사고방식이 바뀐 **전환점** 명시 (예: 4주 트랜잭션/락, 7주 이벤트 분리)
  - **Trade-off 판단** 1~2개 — 왜 그 선택, 대안, 지금 다시 한다면
  - 실전 연결 포인트 (캐시 무효화, Kafka 집계, Resilience4j 설정 등)
- 배치 관련 실측/판단 서술:
  - Chunk 크기 선정 근거와 실측 결과
  - Partition/Multi-thread 도입 전후 처리 시간 비교
  - MV vs 실시간 집계 트레이드오프 서술
  - 재실행 멱등성을 어떻게 보장했는가
