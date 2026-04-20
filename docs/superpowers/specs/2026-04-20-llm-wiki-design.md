# LLM Wiki 설계 문서

**날짜:** 2026-04-20  
**도메인:** 백엔드 엔지니어링 학습 위키  
**대상:** DB, Kafka, Network, OS, Distributed Systems, Patterns, Spring

---

## 개요

Obsidian + Claude Code 기반의 개인 학습 위키. `raw/`에 원본 소스를 추가하고 `/ingest` 스킬을 실행하면 LLM이 자동으로 위키를 생성·업데이트한다. 사용자는 소싱과 탐색을 담당하고, LLM은 요약·크로스레퍼런스·파일링 전체를 담당한다.

세 가지 핵심 워크플로우는 Claude Code 커스텀 슬래시 커맨드(스킬)로 구현한다:
- `/ingest` — 소스 문서를 위키에 통합
- `/query` — 위키 기반으로 질문에 답하고 답변을 위키에 저장
- `/lint` — 위키 건강 상태 점검

---

## 디렉토리 구조

```
llm-wiki/
├── CLAUDE.md                   # 위키 스키마 & 전체 규칙 정의
├── .claude/
│   └── commands/
│       ├── ingest.md           # /ingest 스킬 정의
│       ├── query.md            # /query 스킬 정의
│       └── lint.md             # /lint 스킬 정의
├── raw/                        # 원본 소스 (읽기 전용, LLM이 수정 안 함)
│   ├── articles/               # 웹 클리핑 아티클
│   ├── books/                  # PDF, 책 노트
│   ├── videos/                 # 영상 강의 노트
│   └── assets/                 # 이미지 등 첨부 파일
└── wiki/
    ├── index.md                # 모든 페이지 카탈로그
    ├── log.md                  # 이력 파일 (append-only)
    ├── overview.md             # 위키 전체 요약 & 현황
    ├── db/                     # 트랜잭션, 격리수준, 인덱싱, 복제 등
    ├── kafka/                  # 파티션, 컨슈머 그룹, 오프셋, 브로커 등
    ├── network/                # TCP/IP, HTTP, 로드밸런싱 등
    ├── os/                     # 프로세스, 스레드, 메모리 관리 등
    ├── distributed/            # CAP 이론, 분산 트랜잭션, 합의 알고리즘 등
    ├── patterns/               # CQRS, Event Sourcing, MSA 패턴 등
    ├── spring/                 # Spring Boot, DI, AOP, JPA 연동 등
    └── sources/                # 소스별 요약 페이지 (ingest 시 자동 생성)
```

---

## 스킬 정의

### `/ingest [파일경로]`

소스 파일을 읽어 위키에 통합한다.

**단계:**
1. `raw/` 하위 소스 파일 읽기
2. 핵심 내용 추출
3. `wiki/sources/[소스명].md` 요약 페이지 생성
4. 관련 카테고리 디렉토리(`wiki/db/`, `wiki/kafka/` 등) 하위 페이지 생성 또는 업데이트
5. `wiki/index.md` 업데이트
6. `wiki/overview.md` 업데이트 (새 소스 반영)
7. `wiki/log.md`에 이력 추가: `## [YYYY-MM-DD] ingest | 소스 제목`

### `/query [질문]`

위키를 기반으로 질문에 답하고, 가치 있는 답변은 위키 페이지로 저장한다.

**단계:**
1. `wiki/index.md` 읽어 관련 페이지 파악
2. 관련 페이지들 읽기
3. 답변 생성 (형식: 마크다운 설명, 비교표, 요약 페이지 등 질문에 맞게 선택)
4. 답변이 재사용 가치가 있으면 `wiki/` 적절한 카테고리 아래 페이지로 저장
5. `wiki/log.md`에 이력 추가: `## [YYYY-MM-DD] query | 질문 요약`

### `/lint`

위키 전체를 점검하고 개선이 필요한 항목을 보고한다.

**점검 항목:**
- 고아 페이지 (인바운드 링크 없는 페이지)
- 아웃바운드 링크가 없는 페이지 (크로스레퍼런스 누락)
- 소스는 있지만 위키 페이지가 없는 개념
- 페이지 간 모순되는 내용
- 최신 소스에 의해 낡은 내용
- frontmatter 누락 또는 형식 오류

**출력:** 문제 목록 + 각 항목 수정 제안. `wiki/log.md`에 이력 추가: `## [YYYY-MM-DD] lint | 이슈 N건`

---

## 페이지 포맷

모든 위키 페이지는 아래 frontmatter를 포함한다:

```yaml
---
title: 페이지 제목
category: db | kafka | network | os | distributed | patterns | spring | source
tags: [태그1, 태그2, ...]
sources: [raw/articles/xxx.md]
updated: YYYY-MM-DD
---
```

---

## 크로스레퍼런스 규칙

- 다른 위키 페이지 언급 시 `[[페이지명]]` 형식으로 링크
- 다른 카테고리의 관련 개념은 페이지 하단 "관련 항목" 섹션에 명시
- 모순되거나 상충하는 내용 발견 시 해당 페이지에 표시:
  ```
  > ⚠️ 주의: [소스명]에서 상충하는 내용 발견 — [내용 요약]
  ```

---

## index.md 포맷

카테고리별로 구성하며, 각 항목은 링크 + 한 줄 요약 형식:

```markdown
# Wiki Index

_마지막 업데이트: YYYY-MM-DD | 총 N개 페이지_

## DB
- [[트랜잭션]] — ACID 속성과 격리 수준 정리
- [[인덱스]] — B-Tree, Hash 인덱스 구조와 사용 전략

## Kafka
- [[카프카 파티션]] — 파티션 구조, 리더/팔로워, 오프셋 관리

## Sources
- [[sources/article-xxx]] — 아티클 제목 (YYYY-MM-DD)
```

---

## log.md 포맷

```markdown
# Wiki Log

## [YYYY-MM-DD] ingest | 소스 제목
- 생성: [[페이지1]], [[페이지2]]
- 업데이트: [[페이지3]]

## [YYYY-MM-DD] query | 질문 요약
- 답변 저장: [[페이지명]] (또는 "저장 안 함")

## [YYYY-MM-DD] lint | 이슈 N건
- 고아 페이지 N개, 모순 N건, frontmatter 오류 N건
```

---

## overview.md 업데이트 시점

- `/ingest` 실행 시마다 자동 업데이트
- 내용: 카테고리별 페이지 수, 총 소스 수, 최근 추가 항목, 주요 개념 목록

---

## CLAUDE.md 역할

CLAUDE.md는 LLM에게 위키 운영 방법을 알려주는 스키마 파일이다. 포함 내용:

- 디렉토리 구조 및 각 디렉토리 용도
- 스킬 사용법 요약 (`/ingest`, `/query`, `/lint`)
- 페이지 frontmatter 형식
- 크로스레퍼런스 규칙
- `index.md`, `log.md`, `overview.md` 업데이트 규칙
- 카테고리별 페이지 작성 가이드라인
- `raw/` 디렉토리는 절대 수정하지 않는다는 불변 규칙

---

## 핵심 파일 요약

| 파일 | 역할 | 업데이트 시점 |
|------|------|--------------|
| `CLAUDE.md` | LLM 운영 스키마 | 수동 (사용자) |
| `.claude/commands/ingest.md` | ingest 스킬 | 수동 (사용자) |
| `.claude/commands/query.md` | query 스킬 | 수동 (사용자) |
| `.claude/commands/lint.md` | lint 스킬 | 수동 (사용자) |
| `wiki/index.md` | 전체 페이지 카탈로그 | ingest 시 |
| `wiki/log.md` | append-only 이력 | ingest/query/lint 시 |
| `wiki/overview.md` | 위키 현황 요약 | ingest 시 |
