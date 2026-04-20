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
- 고아 페이지 N개, 크로스레퍼런스 누락 N개, frontmatter 오류 N개, 모순 N건, index 누락 N개, 오래된 내용 N개
```

## overview.md 업데이트 규칙

- `/ingest` 실행 후 카테고리별 페이지 수, 소스 수, 최근 추가 항목을 업데이트한다.
