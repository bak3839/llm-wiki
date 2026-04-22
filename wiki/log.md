# Wiki Log

<!-- 형식: ## [YYYY-MM-DD] ingest|query|lint | 제목 -->
<!-- 예시: ## [2026-04-20] ingest | 아티클 제목 -->
<!-- 빠른 조회: grep "^## \[" wiki/log.md | tail -5 -->

## [2026-04-21] ingest | [Chapter 4] 카프카 컨슈머: 카프카에서 데이터 읽기
- 생성: [[컨슈머-그룹]], [[리밸런스]], [[오프셋-커밋]], [[컨슈머-설정]], [[폴링-루프]], [[kafka_chapter_4]]
- 업데이트: (없음)

## [2026-04-21] query | auto.offset.reset latest와 earliest 작동 방식 비교
- 답변 저장: [[auto-offset-reset]]

## [2026-04-21] lint | 이슈 1건
- 고아 페이지 1개, 크로스레퍼런스 누락 0개, frontmatter 오류 0개, 모순 0건, index 누락 0개, 오래된 내용 0개

## [2026-04-21] lint | 이슈 2건
- 고아 페이지 1개, 크로스레퍼런스 누락 0개, frontmatter 오류 0개, 모순 0건, index 누락 0개, 오래된 내용 1개 (overview.md Kafka 페이지 수 오류: 4 → 6, auto-offset-reset 최근 추가 누락)

## [2026-04-21] lint | 이슈 6건 → 수정 완료
- 링크 불일치 5개 (한글 링크명↔영문 파일명): aliases 추가로 해결, 루트 빈 파일 컨슈머-그룹.md 삭제
- 고아 페이지 1개: [[wiki/sources/kafka_chapter_4|kafka_chapter_4]] (미수정)

## [2026-04-21] lint | index.md·overview.md 링크 명시적 경로로 수정
- index.md, overview.md의 모든 링크를 [[wiki/<카테고리>/<파일명>|표시명]] 형식으로 변경
- ingest.md, query.md 스킬에 명시적 경로 링크 규칙 추가
