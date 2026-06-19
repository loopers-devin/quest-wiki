---
week: 3
title: 도메인 모델링 & 레이어드 아키텍처
status: ready
---

# Week 3 — 도메인 모델링 & 레이어드 아키텍처

## 구현 과제

**과제 개요**: Product, Brand, Like, Order 도메인을 가진 커머스 시스템을 레이어드 아키텍처로 구현.

### PASS 조건 (★ 핵심 항목 모두 충족 필수)

**도메인 모델링**
- ★ Product, Brand, Like, Order 도메인 객체 존재 (Entity 또는 VO)
- ★ 도메인 레이어에 비즈니스 규칙 캡슐화 (단순 getter/setter 위주 아님)
- ★ Repository Interface가 Domain Layer에 위치, 구현체는 Infrastructure Layer

**아키텍처**
- ★ 레이어드 구조: `interfaces` / `application` / `domain` / `infrastructure`
- ★ DIP 적용: Application → Domain ← Infrastructure 의존 방향
- Application Layer는 도메인 객체를 조합하는 얇은 레이어로 구성

**기능**
- 상품 재고 차감이 도메인 레벨에서 처리됨
- 재고 음수 방지 처리 존재
- Like가 별도 도메인으로 분리됨
- 주문 시 재고 부족 예외 처리 존재

**테스트**
- ★ 핵심 도메인 로직에 단위 테스트 존재
- 외부 의존성 분리 (Fake/Stub 활용)
- 정상/예외 케이스 모두 커버

### FAIL 조건
- ★ 핵심 항목 중 하나라도 미충족
- 도메인 레이어 없이 서비스에 비즈니스 로직 집중
- 단위 테스트 전무

### Rising Talent (선택)
- 체크리스트 거의 완벽 + 설계 의도가 코드에서 명확히 드러남
- 특히 **도메인 서비스 분리**, **복합 유스케이스 흐름 설계**가 인상적인 경우

---

## 라이팅 과제

**과제 개요**: 도메인/아키텍처에 대한 학습 정리.

### PASS 조건 (기본)
- 블로그 링크 제출 + 접근 가능 + 100자 초과 → PASS

### FAIL 조건 (엄격 적용)
- 미제출 (링크 없음)
- 링크 접근 불가 (404 등)
- 내용이 사실상 없음 (100자 이하, 빈 글)

### 주차별 추가 기준 (선택)
- (없음 — 기본 규칙만 적용)
