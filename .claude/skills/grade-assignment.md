---
name: grade-assignment
description: Loopers quest 부트캠프 과제 채점 스킬. 주차 번호를 받아 wiki/week-NN.md 기준에 따라 cmux 브라우저로 채점 페이지를 열고, GitHub PR/블로그를 분석해 PASS/FAIL을 판단한 뒤 피드백 하단에 결과를 자동 입력하고 토글을 설정합니다.
---

# 과제 채점 스킬 (grade-assignment)

## 사용법

```
/grade-assignment <주차> [채점 URL]
```

예시:
```
/grade-assignment 3 https://www.loopers.im/grading?lectureId=8&roundId=33&studentId=668
```

- `<주차>`: 1~10 사이 정수. 누락 시 사용자에게 묻는다.
- `[채점 URL]`: 생략 시 현재 cmux `agent-browser`에 열려 있는 채점 페이지 사용.
- **랜딩 URL**: `https://www.loopers.im/grading?lectureId=8&roundId=0` — 학생/라운드 미지정 시 진입 페이지.
- **자동화 전제**: cmux의 `agent-browser` surface를 사용한다. 일반 `browser`가 아니다.

채점 기준은 이 스킬에 하드코딩되어 있지 않다. 호출 시점에 `wiki/week-NN.md`를 읽어 그 안에 정의된 PASS/FAIL 조건을 적용한다.

---

## 실행 절차

### Step -1: 인자 파싱 + 위키 로드

1. 첫 인자가 1~10 범위의 정수면 주차로 사용. 아니면 `AskUserQuestion`으로 주차 확인.
2. 위키 파일 경로: `<repo-root>/wiki/week-NN.md` (NN은 2자리, 예: `week-03.md`).
   - `<repo-root>`는 현재 작업 디렉토리에서 위로 올라가며 `wiki/` 폴더를 찾아 결정. 보통 `/Users/devin/loopers-devin/quest/`.
3. 위키 파일이 없으면 → 사용자에게 알리고 종료.
4. frontmatter의 `status`가 `draft`면 → 사용자에게 알리고 진행 여부 확인.
5. 위키 본문에서 다음을 추출해 채점에 사용한다:
   - `## 구현 과제` 섹션의 PASS 조건(★ 핵심 항목 포함), FAIL 조건, Rising Talent 기준
   - `## 라이팅 과제` 섹션의 주차별 추가 기준 (없으면 기본 규칙 적용)

### Step 0: cmux agent-browser 준비

> **반드시 `agent-browser`를 사용한다.** 일반 `browser`가 아닌 agent 전용 surface여야 한다.

```bash
CMUX=/Applications/cmux.app/Contents/Resources/bin/cmux
$CMUX identify --json
```

이미 agent-browser surface가 있으면 그 ref를 사용한다. 없으면 채점 랜딩 URL로 신규 오픈:

```bash
$CMUX --json agent-browser open "https://www.loopers.im/grading?lectureId=8&roundId=0"
# 반환된 surface_ref (예: surface:6) 를 이하 단계에서 사용
```

> **기본 진입 URL**: `https://www.loopers.im/grading?lectureId=8&roundId=0`
> - 채점 대상 학생별 페이지로 이동하려면 `roundId`, `studentId`가 채워진 URL로 `goto` 한다.

---

### Step 1: 채점 페이지 이동

```bash
$CMUX agent-browser <surface> goto "<채점-URL>"
$CMUX agent-browser <surface> wait --load-state complete --timeout-ms 20000
$CMUX agent-browser <surface> get url
```

URL이 로그인 페이지로 바뀌었다면 → 사용자에게 로그인 요청.

---

### Step 2: 학생 제출물 링크 수집

```bash
$CMUX agent-browser <surface> eval "JSON.stringify({
  github: document.querySelector('a[href*=\"github.com\"]')?.href,
  blog: Array.from(document.querySelectorAll('a[href]')).find(a => !a.href.includes('github.com') && a.href.startsWith('http') && !a.href.includes('loopers.im'))?.href,
  student: document.querySelector('h3.font-bold')?.textContent?.trim()
})"
```

- **github**: 구현 과제 GitHub PR/repo URL
- **blog**: 라이팅 과제 블로그 URL
- **student**: 학생 이름

---

### Step 3: GitHub PR 분석 (구현 과제)

GitHub PR URL인 경우:
```bash
gh pr view <PR번호> --repo <owner/repo> --json title,body,files,commits,additions,deletions
gh pr diff <PR번호> --repo <owner/repo> 2>/dev/null | head -500
```

GitHub repo URL인 경우:
```bash
gh repo view <owner/repo> --json description,defaultBranchRef
gh api /repos/<owner/repo>/contents --jq '.[].name'
```

분석 후 **Step -1에서 로드한 위키 기준**에 따라 PASS/FAIL 및 Rising Talent 여부를 판단한다.

판단 원칙:
- 위키의 ★ 항목 중 하나라도 미충족 → FAIL
- 위키의 FAIL 조건에 해당 → FAIL
- 그 외 항목은 종합 판단
- Rising Talent는 판단만 하고 토글은 절대 클릭하지 않는다 (사람이 직접)

---

### Step 4: 블로그 글 분석 (라이팅 과제)

블로그 URL을 WebFetch로 가져와 내용을 분석한다.

판정 규칙:
1. 위키의 `### 주차별 추가 기준`에 내용이 있으면 그 기준 우선 적용
2. 없으면 기본 규칙 적용:
   - PASS: 링크 있음 + 접근 가능 + 100자 초과
   - FAIL: 미제출 / 접근 불가 / 100자 이하

---

### Step 5: 피드백 모달 열기

```bash
$CMUX agent-browser <surface> snapshot --interactive
# "피드백 작성 및 수정하기" 버튼 ref 확인 (보통 e8~e30 사이)
$CMUX --json agent-browser <surface> click <btn-ref> --snapshot-after
$CMUX agent-browser <surface> wait --text "종합 피드백" --timeout-ms 5000
```

---

### Step 6: 기존 피드백 읽기

```bash
$CMUX agent-browser <surface> eval "document.querySelector('textarea').value"
```

---

### Step 7: 피드백 텍스트 업데이트

**원칙: 기존 피드백 내용은 모두 보존하고, 🤖 평가 블록은 항상 텍스트 최하단에 위치한다.**

#### 7-1. 기존 피드백 파싱

Step 6에서 읽은 텍스트를 두 부분으로 나눈다:
- **사람 작성 영역**: 기존 텍스트에서 마지막 `🤖 구현 평가 결과:` 줄 이후의 모든 🤖 블록을 제외한 앞부분
- **이전 🤖 블록**: 무시(덮어쓰기). 단, 사람이 🤖 사이에 추가 멘트를 끼워 넣은 경우는 보존하기 어려우므로 가능하면 🤖 블록은 항상 가장 마지막에 모여 있다고 가정한다.

> 정규식 가이드: `/(?:\n+)?(?:---\s*\n)?(?:🤖[^\n]*\n?)+\s*$/` 에 매치되는 꼬리 부분을 잘라낸다.

#### 7-2. 새 🤖 블록 구성

**PASS / PASS (이상 없음)**
```
---
🤖 구현 평가 결과: PASS
🤖 라이팅 평가 결과: PASS
```

**구현 FAIL — 사유 함께 기록**
```
---
🤖 구현 평가 결과: FAIL
   - ★ Repository Interface가 Domain Layer에 위치하지 않음 (A4)
   - ★ OrderService 단위 테스트 누락 (O4)
🤖 라이팅 평가 결과: PASS
```

**라이팅 FAIL**
```
---
🤖 구현 평가 결과: PASS
🤖 라이팅 평가 결과: FAIL
   - 블로그 링크 미제출
```

**둘 다 FAIL**
```
---
🤖 구현 평가 결과: FAIL
   - ★ 도메인 모델링 없음 (P1, L1, O1)
   - ★ 단위 테스트 전무
🤖 라이팅 평가 결과: FAIL
   - 링크 접근 불가 (404)
```

> **사유 작성 규칙**
> - FAIL일 때만 `   - ` 들여쓰기로 사유를 한 줄씩 나열한다.
> - 가능하면 위키에서 어긴 항목 코드(`★ A4`, `O4` 등)를 함께 표기해 추적 가능하게 한다.
> - 1~3줄 이내로 간결하게. 장문 분석은 Step 10 결과 보고에만 담는다.
> - PASS인 항목은 사유를 적지 않는다.

#### 7-3. 최종 텍스트 = 사람 영역 + 빈 줄 + 새 🤖 블록

```
<기존 사람 작성 피드백 그대로>

---
🤖 구현 평가 결과: ...
🤖 라이팅 평가 결과: ...
```

기존 사람 영역이 비어있으면 구분선(`---`)도 생략하고 🤖 블록만 쓴다.

#### 7-4. 입력

```bash
$CMUX agent-browser <surface> snapshot --interactive
# textbox ref 확인 (보통 e56 근처)
$CMUX agent-browser <surface> fill <textbox-ref> "<최종 텍스트>"
```

또는 CSS selector로:
```bash
$CMUX agent-browser <surface> fill "textarea" "<최종 텍스트>"
```

> ⚠️ `fill`은 기존 값을 덮어쓰므로 반드시 Step 7-3의 **최종 텍스트 전체**를 넘긴다. 부분만 넘기면 사람 작성 영역이 사라진다.

---

### Step 8: 토글 설정

> **평가 항목 영역에는 총 4개의 토글이 있다. 이 중 "통과" 2개만 다루고, "라이징 탤런트(RT)" 2개는 절대 건드리지 않는다.**

| # | 라벨 | 다루는가? | 동작 |
|---|------|---------|------|
| 1 | 구현 과제 통과 | ✅ | 구현 PASS면 ON, FAIL이면 OFF로 맞춘다 |
| 2 | 구현 과제 라이징 탤런트 | ❌ | **건드리지 않음** (사람이 판단) |
| 3 | 라이팅 과제 통과 | ✅ | 라이팅 PASS면 ON, FAIL이면 OFF로 맞춘다 |
| 4 | 라이팅 과제 라이징 탤런트 | ❌ | **건드리지 않음** (사람이 판단) |

#### 8-1. 라벨 기반으로 토글 식별

ID가 빌드마다 바뀔 수 있으므로 **DOM 텍스트 라벨로 토글을 찾는다**. RT 토글을 잘못 클릭하지 않도록 라벨에 "라이징"이 포함되면 즉시 제외한다.

```bash
$CMUX agent-browser <surface> eval "
JSON.stringify(
  Array.from(document.querySelectorAll('[role=\"switch\"]')).map(sw => {
    // 가장 가까운 라벨 텍스트를 찾는다 (label, 인접 텍스트, aria-label 순)
    const label =
      sw.closest('label')?.textContent?.trim() ||
      sw.parentElement?.textContent?.trim() ||
      sw.getAttribute('aria-label') ||
      '';
    return {
      id: sw.id || null,
      label,
      checked: sw.getAttribute('aria-checked') === 'true',
      isRT: label.includes('라이징'),
      kind: label.includes('구현') ? 'impl' : label.includes('라이팅') ? 'wil' : 'unknown',
    };
  })
)
"
```

기대 결과:
```json
[
  {"id":"...","label":"구현 과제 통과","checked":false,"isRT":false,"kind":"impl"},
  {"id":"...","label":"구현 과제 라이징 탤런트","checked":false,"isRT":true,"kind":"impl"},
  {"id":"...","label":"라이팅 과제 통과","checked":false,"isRT":false,"kind":"wil"},
  {"id":"...","label":"라이팅 과제 라이징 탤런트","checked":false,"isRT":true,"kind":"wil"}
]
```

#### 8-2. 대상 토글 결정

- `isRT === true` 인 토글은 **즉시 제외**. 어떤 경우에도 클릭하지 않는다.
- `kind === 'impl' && !isRT` → 구현 통과 토글 (목표 상태 = 구현 PASS 여부)
- `kind === 'wil' && !isRT` → 라이팅 통과 토글 (목표 상태 = 라이팅 PASS 여부)
- 현재 `checked`와 목표 상태가 **다를 때만** 클릭한다.

#### 8-3. 클릭

`id`가 있으면 ID로, 없으면 라벨 텍스트로 클릭한다.

```bash
# ID가 있을 때 (안전한 selector)
$CMUX agent-browser <surface> click "#<해당 토글 id>"

# ID가 없을 때 — label 텍스트 기반 click
$CMUX agent-browser <surface> click "text=구현 과제 통과 >> [role=switch]"
$CMUX agent-browser <surface> click "text=라이팅 과제 통과 >> [role=switch]"
```

> ⚠️ **절대로 "라이징 탤런트"가 포함된 라벨의 토글을 selector로 잡지 않는다.** 디버깅 중에도 RT 토글은 그대로 둔다.

#### 8-4. 검증

토글 클릭 후 다시 한 번 상태를 읽어 목표와 일치하는지 확인한다.

```bash
$CMUX agent-browser <surface> eval "
Array.from(document.querySelectorAll('[role=\"switch\"]'))
  .filter(sw => !(sw.closest('label')?.textContent || sw.parentElement?.textContent || '').includes('라이징'))
  .map(sw => ({
    label: (sw.closest('label')?.textContent || sw.parentElement?.textContent || '').trim(),
    checked: sw.getAttribute('aria-checked')
  }))
"
```

---

### Step 9: 저장

```bash
$CMUX --json agent-browser <surface> click "저장" --snapshot-after
# 또는 snapshot의 저장 버튼 ref
$CMUX agent-browser <surface> wait --text "저장" --timeout-ms 5000
```

---

### Step 10: 결과 보고

```
✅ 채점 완료: [학생 이름] (Week N)

구현 과제
  - 통과: PASS / FAIL
  - Rising Talent (참고): YES / NO
  핵심 근거: ...

라이팅 과제
  - 통과: PASS / FAIL
  핵심 근거: ...

적용 기준: wiki/week-NN.md
```

---

## 채점 기준

채점 기준은 본 스킬 파일이 아니라 **각 주차별 위키 파일**에 정의되어 있다.

- 위치: `<repo-root>/wiki/week-NN.md`
- 인덱스: `<repo-root>/wiki/README.md`
- 새 라운드 기준 추가/수정은 위키 파일을 직접 편집하면 된다 (이 스킬 파일은 수정 불필요).

---

## 에러 처리

| 상황 | 대응 |
|------|------|
| 주차 인자 누락 | `AskUserQuestion`으로 주차 확인 |
| 위키 파일 없음 | 사용자에게 알리고 종료 |
| 위키 status가 draft | 사용자에게 알리고 진행 여부 확인 |
| agent-browser surface 없음 | `agent-browser open <랜딩 URL>`로 생성 후 진행 |
| 로그인 페이지로 이동됨 | 사용자에게 로그인 후 재실행 요청 |
| GitHub 링크 없음 | 사용자에게 직접 URL 요청 |
| 블로그 링크 없음 | 라이팅 과제 미제출 처리 (FAIL) |
| textarea가 마크다운 에디터라 fill 안 됨 | `eval "document.querySelector('textarea').value = '...'"`로 직접 주입 후 change 이벤트 dispatch |
| 저장 실패 | 사용자에게 알림, 직접 저장 요청 |
