# LLM Wiki Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 백엔드 엔지니어링 학습 위키의 디렉토리 구조, CLAUDE.md 스키마, 그리고 /ingest·/query·/lint 커스텀 커맨드 스킬을 생성한다.

**Architecture:** raw/(원본 소스) + wiki/(LLM 생성 페이지) 이중 구조. CLAUDE.md가 위키 운영 규칙 전체를 정의하고, .claude/commands/ 아래 세 개의 커스텀 커맨드가 ingest/query/lint 워크플로우를 담당한다.

**Tech Stack:** Markdown, Obsidian, Claude Code custom slash commands

---

## File Map

| 작업 | 파일 |
|------|------|
| Create | `raw/articles/.gitkeep` |
| Create | `raw/books/.gitkeep` |
| Create | `raw/videos/.gitkeep` |
| Create | `raw/assets/.gitkeep` |
| Create | `wiki/db/.gitkeep` |
| Create | `wiki/kafka/.gitkeep` |
| Create | `wiki/network/.gitkeep` |
| Create | `wiki/os/.gitkeep` |
| Create | `wiki/distributed/.gitkeep` |
| Create | `wiki/patterns/.gitkeep` |
| Create | `wiki/spring/.gitkeep` |
| Create | `wiki/sources/.gitkeep` |
| Create | `wiki/index.md` |
| Create | `wiki/log.md` |
| Create | `wiki/overview.md` |
| Create | `CLAUDE.md` |
| Create | `.claude/commands/ingest.md` |
| Create | `.claude/commands/query.md` |
| Create | `.claude/commands/lint.md` |

---

## Task 1: 디렉토리 구조 생성

**Files:**
- Create: `raw/articles/.gitkeep`, `raw/books/.gitkeep`, `raw/videos/.gitkeep`, `raw/assets/.gitkeep`
- Create: `wiki/db/.gitkeep`, `wiki/kafka/.gitkeep`, `wiki/network/.gitkeep`, `wiki/os/.gitkeep`, `wiki/distributed/.gitkeep`, `wiki/patterns/.gitkeep`, `wiki/spring/.gitkeep`, `wiki/sources/.gitkeep`

- [ ] **Step 1: raw/ 하위 디렉토리 생성**

```bash
mkdir -p raw/articles raw/books raw/videos raw/assets
touch raw/articles/.gitkeep raw/books/.gitkeep raw/videos/.gitkeep raw/assets/.gitkeep
```

- [ ] **Step 2: wiki/ 하위 디렉토리 생성**

```bash
mkdir -p wiki/db wiki/kafka wiki/network wiki/os wiki/distributed wiki/patterns wiki/spring wiki/sources
touch wiki/db/.gitkeep wiki/kafka/.gitkeep wiki/network/.gitkeep wiki/os/.gitkeep wiki/distributed/.gitkeep wiki/patterns/.gitkeep wiki/spring/.gitkeep wiki/sources/.gitkeep
```

- [ ] **Step 3: 구조 확인**

```bash
find raw wiki -type d | sort
```

Expected output:
```
raw
raw/articles
raw/assets
raw/books
raw/videos
wiki
wiki/db
wiki/distributed
wiki/kafka
wiki/network
wiki/os
wiki/patterns
wiki/sources
wiki/spring
```

- [ ] **Step 4: Commit**

```bash
git add raw/ wiki/
git commit -m "chore: scaffold directory structure"
```

---

## Task 2: wiki/index.md 생성

**Files:**
- Create: `wiki/index.md`

- [ ] **Step 1: index.md 생성**

파일 내용:

```markdown
# Wiki Index

_마지막 업데이트: -_

## DB
<!-- 페이지 추가 시 여기에 기록 -->

## Kafka
<!-- 페이지 추가 시 여기에 기록 -->

## Network
<!-- 페이지 추가 시 여기에 기록 -->

## OS
<!-- 페이지 추가 시 여기에 기록 -->

## Distributed
<!-- 페이지 추가 시 여기에 기록 -->

## Patterns
<!-- 페이지 추가 시 여기에 기록 -->

## Spring
<!-- 페이지 추가 시 여기에 기록 -->

## Sources
<!-- ingest된 소스 목록 -->
```

- [ ] **Step 2: Commit**

```bash
git add wiki/index.md
git commit -m "docs: add wiki/index.md template"
```

---

## Task 3: wiki/log.md 생성

**Files:**
- Create: `wiki/log.md`

- [ ] **Step 1: log.md 생성**

파일 내용:

```markdown
# Wiki Log

<!-- 형식: ## [YYYY-MM-DD] ingest|query|lint | 제목 -->
<!-- 예시: ## [2026-04-20] ingest | 아티클 제목 -->
<!-- 빠른 조회: grep "^## \[" wiki/log.md | tail -5 -->
```

- [ ] **Step 2: Commit**

```bash
git add wiki/log.md
git commit -m "docs: add wiki/log.md template"
```

---

## Task 4: wiki/overview.md 생성

**Files:**
- Create: `wiki/overview.md`

- [ ] **Step 1: overview.md 생성**

파일 내용:

```markdown
# Wiki Overview

_마지막 업데이트: -_

## 현황

| 카테고리 | 페이지 수 | 소스 수 |
|----------|-----------|---------|
| DB | 0 | 0 |
| Kafka | 0 | 0 |
| Network | 0 | 0 |
| OS | 0 | 0 |
| Distributed | 0 | 0 |
| Patterns | 0 | 0 |
| Spring | 0 | 0 |
| **합계** | **0** | **0** |

## 최근 추가 항목

_아직 없음_

## 주요 개념

_소스가 ingest되면 자동으로 채워집니다._
```

- [ ] **Step 2: Commit**

```bash
git add wiki/overview.md
git commit -m "docs: add wiki/overview.md template"
```

---

## Task 5: CLAUDE.md 생성

**Files:**
- Create: `CLAUDE.md`

- [ ] **Step 1: CLAUDE.md 생성**

파일 내용:

```markdown
# LLM Wiki — 운영 스키마

이 저장소는 백엔드 엔지니어링 학습을 위한 개인 위키다.
`raw/`에 원본 소스를 추가하고 `/ingest`를 실행하면 LLM이 위키를 자동으로 생성·업데이트한다.

## 디렉토리 구조

```
llm-wiki/
├── CLAUDE.md                   # 이 파일
├── .claude/commands/           # 커스텀 슬래시 커맨드
│   ├── ingest.md               # /ingest
│   ├── query.md                # /query
│   └── lint.md                 # /lint
├── raw/                        # 원본 소스 — 절대 수정 금지
│   ├── articles/
│   ├── books/
│   ├── videos/
│   └── assets/
└── wiki/                       # LLM이 생성·유지하는 위키
    ├── index.md
    ├── log.md
    ├── overview.md
    ├── db/
    ├── kafka/
    ├── network/
    ├── os/
    ├── distributed/
    ├── patterns/
    ├── spring/
    └── sources/
```

## 불변 규칙

- `raw/` 디렉토리는 절대 수정하지 않는다. 읽기만 허용.
- `wiki/log.md`는 append-only다. 기존 항목을 수정하지 않는다.

## 스킬 사용법

- `/ingest <파일경로>` — raw/ 하위 소스를 위키에 통합
- `/query <질문>` — 위키 기반으로 질문에 답하고 결과를 페이지로 저장
- `/lint` — 위키 건강 상태 점검

## 페이지 Frontmatter 형식

모든 wiki/ 하위 페이지는 아래 frontmatter를 포함해야 한다:

```yaml
---
title: 페이지 제목
category: db | kafka | network | os | distributed | patterns | spring | source
tags: [태그1, 태그2]
sources: [raw/articles/파일명.md]
updated: YYYY-MM-DD
---
```

## 크로스레퍼런스 규칙

- 다른 위키 페이지를 언급할 때는 항상 `[[페이지명]]` 형식으로 링크한다.
- 다른 카테고리의 관련 개념은 페이지 하단 "관련 항목" 섹션에 명시한다.
- 소스 간 모순되는 내용 발견 시 해당 페이지에 표시한다:
  ```
  > ⚠️ 주의: [소스명]에서 상충하는 내용 발견 — [내용 요약]
  ```

## index.md 업데이트 규칙

- 새 페이지 생성 시 해당 카테고리 섹션에 추가: `- [[페이지명]] — 한 줄 요약`
- 파일 상단 "마지막 업데이트" 날짜를 갱신한다.

## log.md 업데이트 규칙

항목 형식:

```
## [YYYY-MM-DD] ingest | 소스 제목
- 생성: [[페이지1]], [[페이지2]]
- 업데이트: [[페이지3]]

## [YYYY-MM-DD] query | 질문 요약
- 답변 저장: [[페이지명]] (저장하지 않은 경우 "저장 안 함")

## [YYYY-MM-DD] lint | 이슈 N건
- 고아 페이지 N개, 모순 N건, frontmatter 오류 N건
```

## overview.md 업데이트 규칙

- `/ingest` 실행 후 카테고리별 페이지 수, 소스 수, 최근 추가 항목을 업데이트한다.
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add CLAUDE.md wiki schema"
```

---

## Task 6: /ingest 커스텀 커맨드 생성

**Files:**
- Create: `.claude/commands/ingest.md`

- [ ] **Step 1: .claude/commands/ 디렉토리 생성**

```bash
mkdir -p .claude/commands
```

- [ ] **Step 2: ingest.md 생성**

파일 내용:

```markdown
# /ingest

소스 파일을 읽어 위키에 통합한다. 인자로 받은 파일 경로(`$ARGUMENTS`)가 대상이다.

## 단계

1. **소스 읽기**
   - `$ARGUMENTS` 경로의 파일을 읽는다.
   - 파일이 존재하지 않으면 "파일을 찾을 수 없습니다: $ARGUMENTS" 오류를 출력하고 중단한다.

2. **핵심 내용 추출**
   - 주요 개념, 정의, 원리, 예시를 파악한다.
   - 이 소스가 주로 속하는 카테고리를 결정한다 (db/kafka/network/os/distributed/patterns/spring).

3. **sources/ 페이지 생성**
   - `wiki/sources/<파일명>.md`를 생성한다 (확장자는 .md로 통일).
   - frontmatter: `category: source`, `sources: [$ARGUMENTS]`, `updated: <오늘 날짜>`
   - 내용: 소스 제목, 출처, 핵심 요약 (300~500자), 주요 개념 목록

4. **카테고리 페이지 생성·업데이트**
   - 소스에서 등장하는 각 핵심 개념마다 `wiki/<category>/<개념명>.md`를 확인한다.
   - 페이지가 없으면 새로 생성한다. 있으면 새 소스의 내용을 반영해 업데이트한다.
   - 각 페이지에 frontmatter를 포함한다 (CLAUDE.md 형식 참고).
   - 관련 페이지는 `[[페이지명]]`으로 상호 링크한다.
   - 기존 내용과 모순되면 `> ⚠️ 주의:` 블록으로 표시한다.

5. **index.md 업데이트**
   - 새로 생성된 페이지를 해당 카테고리 섹션에 추가한다: `- [[페이지명]] — 한 줄 요약`
   - 파일 상단 "마지막 업데이트" 날짜를 갱신한다.

6. **overview.md 업데이트**
   - 카테고리별 페이지 수 및 소스 수 표를 갱신한다.
   - "최근 추가 항목"에 이번 ingest에서 생성·업데이트된 페이지를 기록한다.
   - 파일 상단 "마지막 업데이트" 날짜를 갱신한다.

7. **log.md에 이력 추가**
   - 파일 하단에 아래 형식으로 추가한다:
   ```
   ## [YYYY-MM-DD] ingest | <소스 제목>
   - 생성: [[페이지1]], [[페이지2]], ...
   - 업데이트: [[페이지3]], ...
   ```

## 완료 후 출력

ingest가 끝나면 다음을 출력한다:
- 생성된 페이지 목록
- 업데이트된 페이지 목록
- 총 터치된 파일 수
```

- [ ] **Step 3: 파일 존재 확인**

```bash
cat .claude/commands/ingest.md
```

Expected: 파일 내용이 출력됨

- [ ] **Step 4: Commit**

```bash
git add .claude/commands/ingest.md
git commit -m "feat: add /ingest custom command"
```

---

## Task 7: /query 커스텀 커맨드 생성

**Files:**
- Create: `.claude/commands/query.md`

- [ ] **Step 1: query.md 생성**

파일 내용:

```markdown
# /query

위키를 기반으로 질문에 답한다. 가치 있는 답변은 위키 페이지로 저장한다.
인자로 받은 질문(`$ARGUMENTS`)에 대해 답한다.

## 단계

1. **index.md 읽기**
   - `wiki/index.md`를 읽어 관련 카테고리와 페이지를 파악한다.

2. **관련 페이지 읽기**
   - 질문과 관련된 페이지들을 읽는다.
   - 필요한 경우 sources/ 페이지도 참고한다.

3. **답변 생성**
   - 질문 유형에 따라 형식을 선택한다:
     - 개념 설명 → 마크다운 설명 + 예시
     - 비교 질문 → 비교표 (마크다운 테이블)
     - 흐름/순서 질문 → 단계별 목록
   - 답변에 근거로 사용된 위키 페이지를 `[[페이지명]]` 형식으로 인용한다.

4. **위키 저장 여부 결정**
   - 답변이 재사용 가치가 있으면 (비교 분석, 종합 설명 등) 위키 페이지로 저장한다.
   - 저장 경로: `wiki/<관련 카테고리>/<답변 제목>.md`
   - frontmatter의 `category`는 가장 관련성 높은 카테고리로 설정한다.
   - 저장 후 `wiki/index.md`에 해당 페이지를 추가한다.

5. **log.md에 이력 추가**
   ```
   ## [YYYY-MM-DD] query | <질문 요약>
   - 답변 저장: [[페이지명]] (저장하지 않은 경우 "저장 안 함")
   ```
```

- [ ] **Step 2: Commit**

```bash
git add .claude/commands/query.md
git commit -m "feat: add /query custom command"
```

---

## Task 8: /lint 커스텀 커맨드 생성

**Files:**
- Create: `.claude/commands/lint.md`

- [ ] **Step 1: lint.md 생성**

파일 내용:

```markdown
# /lint

위키 전체를 점검하고 개선이 필요한 항목을 보고한다.

## 점검 단계

1. **index.md 읽기**
   - 등록된 모든 페이지 목록을 확보한다.

2. **전체 페이지 순회**
   - `wiki/` 하위 모든 .md 파일을 읽는다 (index.md, log.md, overview.md 제외).

3. **점검 항목별 확인**

   **a. 고아 페이지** — 다른 페이지에서 `[[이 페이지]]`로 링크되지 않는 페이지
   
   **b. 크로스레퍼런스 누락** — 아웃바운드 `[[링크]]`가 하나도 없는 페이지
   
   **c. frontmatter 오류** — title/category/tags/sources/updated 중 하나라도 누락된 페이지
   
   **d. 모순 항목** — 같은 개념에 대해 서로 다른 설명을 하는 페이지 쌍
   
   **e. index.md 누락** — `wiki/` 하위에 존재하지만 `wiki/index.md`에 등록되지 않은 페이지
   
   **f. 오래된 내용** — 최근 ingest된 소스와 상충하는 내용을 가진 페이지

4. **보고서 출력**

   다음 형식으로 결과를 출력한다:

   ```
   ## Lint 결과 — YYYY-MM-DD

   ### 고아 페이지 (N개)
   - [[페이지명]] — 인바운드 링크 없음

   ### 크로스레퍼런스 누락 (N개)
   - [[페이지명]] — 아웃바운드 링크 없음

   ### Frontmatter 오류 (N개)
   - [[페이지명]] — 누락 필드: updated, tags

   ### 모순 항목 (N건)
   - [[페이지A]] vs [[페이지B]] — [모순 내용 요약]

   ### Index 누락 (N개)
   - [[페이지명]] — index.md에 미등록

   ### 오래된 내용 (N개)
   - [[페이지명]] — [업데이트 필요 이유]

   총 이슈: N건
   ```

5. **log.md에 이력 추가**
   ```
   ## [YYYY-MM-DD] lint | 이슈 N건
   - 고아 페이지 N개, 크로스레퍼런스 누락 N개, frontmatter 오류 N개, 모순 N건, index 누락 N개, 오래된 내용 N개
   ```

## 수정 제안

각 이슈 항목에 대해 수정 방법을 제안한다. 사용자가 수정을 요청하면 해당 페이지를 직접 수정한다.
```

- [ ] **Step 2: Commit**

```bash
git add .claude/commands/lint.md
git commit -m "feat: add /lint custom command"
```

---

## Task 9: 환영합니다!.md 정리 및 최종 push

**Files:**
- Delete: `환영합니다!.md` (Obsidian 기본 파일, 불필요)

- [ ] **Step 1: 기본 환영 파일 삭제**

```bash
git rm "환영합니다!.md"
```

- [ ] **Step 2: 전체 구조 최종 확인**

```bash
find . -not -path './.git/*' -not -path './.obsidian/*' -not -name '.DS_Store' | sort
```

Expected output (주요 파일):
```
.
./CLAUDE.md
./.claude/commands/ingest.md
./.claude/commands/query.md
./.claude/commands/lint.md
./docs/superpowers/plans/2026-04-20-llm-wiki-foundation.md
./docs/superpowers/specs/2026-04-20-llm-wiki-design.md
./llm-wiki.md
./raw/articles/.gitkeep
./raw/assets/.gitkeep
./raw/books/.gitkeep
./raw/videos/.gitkeep
./wiki/db/.gitkeep
./wiki/distributed/.gitkeep
./wiki/index.md
./wiki/kafka/.gitkeep
./wiki/log.md
./wiki/network/.gitkeep
./wiki/os/.gitkeep
./wiki/overview.md
./wiki/patterns/.gitkeep
./wiki/sources/.gitkeep
./wiki/spring/.gitkeep
```

- [ ] **Step 3: Commit 및 push**

```bash
git commit -m "chore: remove default obsidian welcome file"
git push origin main
```
