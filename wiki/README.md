# Loopers 부트캠프 채점 위키

루퍼스 부트캠프 10주차 과제의 PASS/FAIL 채점 기준을 주차별로 정리한 위키입니다.
`/grade-assignment <주차> [URL]` 스킬이 이 위키를 읽어 채점에 사용합니다.

## 구조

- `week-NN.md` — 주차별 구현/라이팅 과제 기준
- 각 파일은 frontmatter(`week`, `title`, `status`)와 두 섹션(구현/라이팅)으로 구성

## 주차 인덱스

| 주차 | 주제 | 상태 |
|------|------|------|
| [Week 1](week-01.md) | TBD | draft |
| [Week 2](week-02.md) | TBD | draft |
| [Week 3](week-03.md) | 도메인 모델링 & 레이어드 아키텍처 | ready |
| [Week 4](week-04.md) | 트랜잭션 · 동시성 제어 · 쿠폰 도메인 | ready |
| [Week 5](week-05.md) | TBD | draft |
| [Week 6](week-06.md) | TBD | draft |
| [Week 7](week-07.md) | TBD | draft |
| [Week 8](week-08.md) | TBD | draft |
| [Week 9](week-09.md) | TBD | draft |
| [Week 10](week-10.md) | TBD | draft |

## 기본 규칙

### 구현 과제
- ★ 표시는 **핵심 필수 항목**. 하나라도 빠지면 FAIL.
- 핵심 외 항목은 종합 판단.
- Rising Talent 토글은 사람이 직접 판단 (스킬은 건드리지 않음).

### 라이팅 과제 (기본값)
- 블로그 링크 제출 + 접근 가능 + 100자 초과 → **PASS**
- 미제출 / 링크 접근 불가 / 100자 이하 → **FAIL**
- 주차별 추가 기준이 있으면 그것을 우선 적용.

## 기준 추가/수정 방법

1. 해당 `week-NN.md` 열기
2. `## 구현 과제` 섹션의 PASS/FAIL 조건 작성
3. 주차별 추가 라이팅 기준이 있으면 `### 주차별 추가 기준`에 작성
4. frontmatter의 `status: draft` → `status: ready`로 변경
5. 커밋 & 푸시
