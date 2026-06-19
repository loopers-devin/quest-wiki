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

### Step 6: 기존 피드백 읽기 + **즉시 백업** (필수 가드)

> ⚠️ **이 단계를 건너뛰면 절대 Step 7로 진행하지 않는다.** 과거에 이 단계 누락으로 멘토 피드백이 영구 손실된 사고가 있었음.

#### 6-1. 기존 textarea 값 읽기

```bash
ORIGINAL=$($CMUX agent-browser <surface> eval "document.querySelector('textarea')?.value ?? ''")
```

#### 6-2. **타임스탬프 백업 파일에 즉시 저장**

```bash
BACKUP_DIR="$HOME/.claude/projects/-Users-devin-loopers-devin-quest/grading-backups"
mkdir -p "$BACKUP_DIR"
TS=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="$BACKUP_DIR/week${WEEK}_${STUDENT_NAME}_${TS}.txt"
printf '%s' "$ORIGINAL" > "$BACKUP_FILE"
# 검증: 백업 파일이 실제로 생성되었는지 확인
if [ ! -f "$BACKUP_FILE" ]; then
  echo "FATAL: 백업 생성 실패 — 채점 중단"
  exit 1
fi
echo "✅ 백업 저장: $BACKUP_FILE ($(wc -c < "$BACKUP_FILE")자)"
```

> **규칙**:
> - 백업 파일이 디스크에 존재함을 확인하기 전에는 절대 Step 7으로 가지 않는다.
> - 백업 경로를 결과 보고(Step 10)에 항상 포함한다.
> - 같은 학생을 여러 번 채점해도 타임스탬프가 다르므로 누적 보관된다.

---

### Step 7: 피드백 텍스트 업데이트 (append-only)

> ⚠️ **절대 금지**: `fill`로 textarea를 통째 교체하는 행위.
> **유일하게 허용되는 동작**: 원본 텍스트의 **마지막에만** `🤖` 평가 블록을 덧붙이는 것.

#### 7-1. 새 🤖 블록 구성

새로 추가할 평가 블록 (PASS/FAIL 케이스별):

**PASS / PASS**
```
🤖 구현 평가 결과: PASS
🤖 라이팅 평가 결과: PASS
```

**구현 FAIL — 사유 함께 기록**
```
🤖 구현 평가 결과: FAIL
   - ★ Repository Interface가 Domain Layer에 위치하지 않음 (A4)
   - ★ OrderService 단위 테스트 누락 (O4)
🤖 라이팅 평가 결과: PASS
```

**둘 다 FAIL**
```
🤖 구현 평가 결과: FAIL
   - ★ 도메인 모델링 없음 (P1, L1, O1)
🤖 라이팅 평가 결과: FAIL
   - 블로그 링크 미제출
```

> **사유 작성 규칙**
> - FAIL일 때만 `   - ` 들여쓰기로 사유 1~3줄.
> - 가능하면 위키 항목 코드(`★ A4`, `O4` 등)와 함께.
> - PASS 항목엔 사유 없음.

#### 7-2. 최종 텍스트 = **원본 그대로 + 구분선 + 새 🤖 블록**

```
<원본 텍스트 그대로 — 한 글자도 수정 금지>

---
🤖 구현 평가 결과: ...
🤖 라이팅 평가 결과: ...
```

- 원본 끝에 이미 이전 🤖 블록이 있어도 **건드리지 않고** 그 뒤에 새 블록을 추가한다. 누적되는 한이 있어도 손실 위험은 피한다.
- 원본이 완전히 비어있는 경우에만 구분선(`---`) 없이 🤖 블록만 입력한다.

#### 7-3. 입력 (백업 확인 후에만 실행)

```bash
# 백업 파일 존재 재확인 — 없으면 중단
[ -f "$BACKUP_FILE" ] || { echo "FATAL: 백업 없음"; exit 1; }

# 원본을 변수에 다시 로드 (혹시 페이지가 다른 학생으로 바뀌었을 가능성 차단)
CURRENT=$($CMUX agent-browser <surface> eval "document.querySelector('textarea')?.value ?? ''")
if [ "$CURRENT" != "$ORIGINAL" ]; then
  echo "FATAL: 페이지 textarea가 백업 시점과 달라짐 — 채점 중단"
  exit 1
fi

# 최종 텍스트 = 원본 + (원본 비어있지 않으면) "\n\n---\n" + 새 🤖 블록
if [ -n "$ORIGINAL" ]; then
  FINAL="${ORIGINAL}

---
${ROBOT_BLOCK}"
else
  FINAL="${ROBOT_BLOCK}"
fi

# fill로 입력 (이때 FINAL은 원본을 100% 포함하므로 손실 없음)
$CMUX agent-browser <surface> fill "textarea" "$FINAL"
```

> ⚠️ **재발 방지 체크리스트** (코드 작성 시 매번 확인):
> 1. 백업 파일이 디스크에 실제로 만들어졌는가?
> 2. `FINAL` 변수에 `$ORIGINAL`이 prefix로 들어가 있는가?
> 3. 페이지가 백업 시점과 같은 학생인가?
> 위 셋 중 하나라도 아니면 절대 fill 호출하지 않는다.

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
