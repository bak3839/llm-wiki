# LLM Wiki 설계 문서

**날짜:** 2026-04-20  
**도메인:** 백엔드 엔지니어링 학습 위키  
**대상:** DB, Kafka, Network, OS, Distributed Systems, Patterns, Spring

---

## 개요

Obsidian + Claude Code 기반의 개인 학습 위키. `raw/`에 원본 소스를 추가하고 ingest 워크플로우를 실행하면 LLM이 자동으로 위키를 생성·업데이트한다. 사용자는 소싱과 탐색을 담당하고, LLM은 요약·크로스레퍼런스·파일링 전체를 담당한다.

---

## 디렉토리 구조

```
llm-wiki/
├── CLAUDE.md               # 위키 스키마 & 워크플로우 정의
├── raw/                    # 원본 소스 (읽기 전용, LLM이 수정 안 함)
│   ├── articles/           # 웹 클리핑 아티클
│   ├── books/              # PDF, 책 노트
│   └── videos/             # 영상 강의 노트
└── wiki/
    ├── index.md            # 모든 페이지 카탈로그
    ├── log.md              # ingest/query 이력 (append-only)
    ├── overview.md         # 위키 전체 요약 & 현황
    ├── db/                 # 트랜잭션, 격리수준, 인덱싱, 복제 등
    ├── kafka/              # 파티션, 컨슈머 그룹, 오프셋, 브로커 등
    ├── network/            # TCP/IP, HTTP, 로드밸런싱 등
    ├── os/                 # 프로세스, 스레드, 메모리 관리 등
    ├── distributed/        # CAP 이론, 분산 트랜잭션, 합의 알고리즘 등
    ├── patterns/           # CQRS, Event Sourcing, MSA 패턴 등
    ├── spring/             # Spring Boot, DI, AOP, JPA 연동 등
    └── sources/            # 소스별 요약 페이지 (ingest 시 자동 생성)
```

---

## Ingest 워크플로우

트리거: 사용자가 `raw/`에 파일 추가 후 `ingest [파일명]` 입력

1. 소스 파일 읽기
2. 핵심 내용 추출 & `wiki/sources/` 아래 요약 페이지 생성
3. 관련 카테고리 디렉토리 하위 페이지 생성 또는 업데이트
4. `wiki/index.md` 업데이트 (새 페이지 추가, 기존 페이지 요약 갱신)
5. `wiki/log.md`에 이력 항목 추가: `## [YYYY-MM-DD] ingest | 소스 제목`

---

## 페이지 포맷

모든 위키 페이지는 아래 frontmatter를 포함한다:

```yaml
---
title: 페이지 제목
category: db | kafka | network | os | distributed | patterns | spring
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

## CLAUDE.md 역할

CLAUDE.md는 LLM에게 위키 운영 방법을 알려주는 스키마 파일이다. 포함 내용:

- 디렉토리 구조 및 각 디렉토리의 용도
- ingest 워크플로우 단계별 지침
- 페이지 frontmatter 형식
- 크로스레퍼런스 규칙
- index.md, log.md 업데이트 규칙
- 카테고리별 페이지 작성 가이드라인

---

## 핵심 파일

| 파일 | 역할 |
|------|------|
| `CLAUDE.md` | LLM 운영 스키마 (위키의 헌법) |
| `wiki/index.md` | 모든 페이지 카탈로그, 링크 + 한 줄 요약 |
| `wiki/log.md` | append-only 이력 파일 |
| `wiki/overview.md` | 위키 전체 상태 요약 |
