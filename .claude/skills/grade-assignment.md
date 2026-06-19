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
- `[채점 URL]`: 생략 시 현재 cmux 브라우저에 열려 있는 채점 페이지 사용.

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

### Step 0: cmux 브라우저 준비

```bash
CMUX=/Applications/cmux.app/Contents/Resources/bin/cmux
$CMUX identify --json
```

이미 browser surface가 있으면 그 ref를 사용한다. 없으면:

```bash
$CMUX --json browser open "https://www.loopers.im/grading"
# 반환된 surface_ref (예: surface:6) 를 이하 단계에서 사용
```

---

### Step 1: 채점 페이지 이동

```bash
$CMUX browser <surface> goto "<채점-URL>"
$CMUX browser <surface> wait --load-state complete --timeout-ms 20000
$CMUX browser <surface> get url
```

URL이 로그인 페이지로 바뀌었다면 → 사용자에게 로그인 요청.

---

### Step 2: 학생 제출물 링크 수집

```bash
$CMUX browser <surface> eval "JSON.stringify({
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
$CMUX browser <surface> snapshot --interactive
# "피드백 작성 및 수정하기" 버튼 ref 확인 (보통 e8~e30 사이)
$CMUX --json browser <surface> click <btn-ref> --snapshot-after
$CMUX browser <surface> wait --text "종합 피드백" --timeout-ms 5000
```

---

### Step 6: 기존 피드백 읽기

```bash
$CMUX browser <surface> eval "document.querySelector('textarea').value"
```

---

### Step 7: 피드백 텍스트 업데이트

기존 텍스트에서 `🤖` 결과 줄이 있으면 해당 줄을 교체하고, 없으면 맨 끝에 추가한다.

**새 결과 형식:**
```
🤖 구현 평가 결과: PASS
🤖 라이팅 평가 결과: PASS
```

FAIL인 경우 `PASS` 대신 `FAIL`.

기존 피드백이 있을 때:
```
<기존 내용>

🤖 구현 평가 결과: PASS
🤖 라이팅 평가 결과: PASS
```

기존 피드백이 없을 때:
```
🤖 구현 평가 결과: PASS
🤖 라이팅 평가 결과: PASS
```

텍스트 구성 후 snapshot으로 textbox ref 확인하고 입력:
```bash
$CMUX browser <surface> snapshot --interactive
# textbox ref 확인 (보통 e56 근처)
$CMUX browser <surface> fill <textbox-ref> "<전체 텍스트>"
```

또는 CSS selector로:
```bash
$CMUX browser <surface> fill "textarea" "<전체 텍스트>"
```

---

### Step 8: 토글 설정

**Rising Talent(RT) 토글은 건드리지 않는다.** RT는 사람이 직접 판단한다.

통과 여부 토글만 설정한다:

| ID | 항목 | 조건 |
|----|------|------|
| `#assignment-pass` | 구현 과제 통과 | 구현 PASS → true |
| `#wil-pass` | 라이팅 과제 통과 | 라이팅 PASS → true |

```bash
# 현재 상태 확인
$CMUX browser <surface> eval "JSON.stringify(Array.from(document.querySelectorAll('[role=\"switch\"]')).map(s => ({id: s.id, checked: s.getAttribute('aria-checked')})))"

# 결과에 따라 통과 토글만 클릭 (RT 토글은 무시)
$CMUX browser <surface> click "#assignment-pass"   # 필요시만
$CMUX browser <surface> click "#wil-pass"          # 필요시만
```

---

### Step 9: 저장

```bash
$CMUX --json browser <surface> click "저장" --snapshot-after
# 또는 snapshot의 저장 버튼 ref
$CMUX browser <surface> wait --text "저장" --timeout-ms 5000
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
| 로그인 페이지로 이동됨 | 사용자에게 로그인 후 재실행 요청 |
| GitHub 링크 없음 | 사용자에게 직접 URL 요청 |
| 블로그 링크 없음 | 라이팅 과제 미제출 처리 (FAIL) |
| textarea가 마크다운 에디터라 fill 안 됨 | `eval "document.querySelector('textarea').value = '...'"`로 직접 주입 후 change 이벤트 dispatch |
| 저장 실패 | 사용자에게 알림, 직접 저장 요청 |
