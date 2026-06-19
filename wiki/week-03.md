---
week: 3
title: 도메인 모델링 & 레이어드 아키텍처
status: ready
---

# Week 3 — 도메인 모델링 & 레이어드 아키텍처

## 과제 개요

Product, Brand, Like, Order 4개 도메인을 가진 커머스 시스템을, **도메인 모델링 + 레이어드 아키텍처 + DIP** 를 적용해 구현한다. Application Layer는 얇게, 도메인 객체 조합 위주로 흐름을 작성하고, 핵심 도메인 로직은 단위 테스트로 검증한다.

---

## 💻 구현 과제 채점 테이블

> **판정 원칙**: ★(필수) 항목 중 하나라도 빠지면 **FAIL**. ◯(일반) 항목은 종합 판단(과반 충족 권장). △(보너스)는 Rising Talent 후보 신호.

### 🏷 Product / Brand 도메인

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| P1 | 상품 객체가 브랜드 정보 + 좋아요 수를 포함(또는 조합 결과로 노출) | ★ | `Product` 엔티티/응답 모델에 brand, likeCount 필드 |
| P2 | 상품 정렬 조건(`latest`, `price_asc`, `likes_desc`) 조회 설계 | ◯ | 정렬 enum/파라미터 + Repository 메서드 시그니처 |
| P3 | 상품은 재고를 가지고 주문 시 차감 가능 | ★ | `Product.deductStock(qty)` 같은 도메인 메서드 |
| P4 | 재고 음수 방지가 도메인 레벨에서 처리됨 | ★ | 차감 메서드 내부 가드 + 예외 (서비스 if문은 감점) |

### 👍 Like 도메인

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| L1 | 좋아요가 유저-상품 관계로 **별도 도메인** 분리 | ★ | `/domain/like` 패키지 + `Like` 엔티티 |
| L2 | 상품 상세/목록에서 좋아요 수 함께 제공 | ◯ | Facade/Service에서 like count 조회 조합 |
| L3 | 좋아요 등록/취소 흐름의 단위 테스트 존재 | ★ | `LikeServiceTest` 또는 도메인 테스트 |

### 🛒 Order 도메인

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| O1 | 주문이 여러 상품 + 수량을 표현 | ★ | `Order` + `OrderLine`/`OrderItem` |
| O2 | 주문 시 재고 차감 수행(결제는 후주차) | ★ | OrderService에서 Product.deductStock 호출 |
| O3 | 유저 부재 / 상품 부재 / 재고 부족 예외 흐름 설계 | ★ | 명시적 예외 타입 또는 명시적 처리 |
| O4 | 정상 주문 / 예외 주문 모두 단위 테스트 커버 | ★ | OrderServiceTest 양/음 케이스 |

### 🧩 도메인 서비스

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| D1 | 도메인 간 협력 로직이 Domain Service에 위치 | ◯ | `/domain/.../*DomainService` 또는 유사 |
| D2 | 상품 상세에서 Product+Brand 조합을 도메인 서비스에서 처리 | △ | ProductInfoService 등 (없어도 PASS 가능) |
| D3 | 복합 유스케이스는 Application Layer, 도메인 로직은 위임 | ★ | Facade/UseCase가 얇고 도메인에 위임 |
| D4 | 도메인 서비스는 무상태, 도메인 객체 협력 중심 | ◯ | 필드 없음 / 외부 상태 의존 없음 |

### 🧱 소프트웨어 아키텍처 & 설계

| # | 항목 | 필수 | 자동 채점 힌트 |
|---|------|------|--------------|
| A1 | 의존 방향: Application → Domain ← Infrastructure | ★ | import 검사 (domain이 infra/app을 모르면 OK) |
| A2 | Application Layer가 도메인 객체 조합 orchestration | ★ | Facade/UseCase 내부에 비즈니스 분기 ≒ 0 |
| A3 | 핵심 비즈니스 로직은 Entity/VO/Domain Service에 위치 | ★ | 서비스에 if/계산식 몰리면 감점 |
| A4 | Repository Interface는 Domain, 구현체는 Infrastructure | ★ | `/domain/.../*Repository.kt` + `/infrastructure/...Impl` |
| A5 | 패키지는 계층 + 도메인 기준 (`/domain/order`, `/application/like` 등) | ★ | 디렉토리 구조 확인 |
| A6 | 외부 의존성 분리, Fake/Stub으로 단위 테스트 가능 | ★ | 테스트에서 H2/JPA 통째로 안 띄움 |

---

## 🤖 자동 채점 룰 (스킬용)

### 구현 PASS 조건
- **★(필수) 항목이 모두 충족**되면 PASS
- ★ 항목 중 하나라도 미충족 → **FAIL**
- ◯(일반) 항목은 70% 이상 충족 권장 (미달 시 종합 감점 사유로 기록만)

### 구현 FAIL 조건
- ★ 중 하나라도 미충족
- 도메인 레이어 없이 서비스에 비즈니스 로직 집중
- 단위 테스트 전무 또는 OrderService 테스트 누락

### Rising Talent 신호 (사람 판단용 — 토글은 클릭하지 않음)
- 체크리스트 거의 완벽 + 설계 의도가 코드에서 명확히 드러남
- D2(Product+Brand 조합을 도메인 서비스에서 처리) 같은 △ 항목 충족
- 복합 유스케이스 흐름이 깔끔하게 분리됨
- VO 도입이나 도메인 이벤트 등 한 단계 더 나아간 설계

---

## ✍️ 라이팅 과제 채점

### 제출 형식 (두 가지 중 택1 또는 둘 다)

1. **GitHub Issue (Tech Note)** — 다음 4개 포맷 중 선택
   - 📐 Design Doc (설계 의사결정 중심)
   - 🪞 Retrospective (회고 · 트러블슈팅)
   - ⚔️ Challenge Story (도전 → 해결 서사)
   - 📊 Benchmark Report (A vs B 측정)
2. **Blog** — 주차 학습/판단 정리, TL;DR 포함

### PASS 조건 (기본)
- 블로그 또는 Issue 링크 제출 + 접근 가능 + **100자 초과** → PASS

### FAIL 조건 (엄격 적용)
- 미제출 (링크 없음)
- 링크 접근 불가 (404, 권한, 비공개 등)
- 내용이 사실상 없음 (100자 이하, 빈 글)

### 주차별 추가 기준
- (없음 — 기본 규칙만 적용)

### Rising Talent 신호 (사람 판단용)
- "무엇을 했다"가 아니라 **"왜 그렇게 판단했는가"**가 드러남
- 코드 비교 / 흐름도 / 리팩토링 전후 등 구체적 근거 제시
- 추천 주제 중 하나를 깊이 있게 다룸:
  - 상품이 좋아요 수를 직접 관리해야 할까?
  - 상품 상세에서 브랜드를 함께 제공하려면 누가 조합해야 할까?
  - VO 도입 시점과 이유?
  - Order/Product/User의 책임 분배?
  - Repository Interface를 Domain Layer에 두는 이유?
  - 도메인 → Application으로 옮긴 이유?
  - 테스트 가능한 구조를 위한 우선 고려 사항?
