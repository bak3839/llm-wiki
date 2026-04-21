---
title: "[Chapter 4] 카프카 컨슈머: 카프카에서 데이터 읽기"
category: source
tags: [kafka, consumer, consumer-group, rebalance, offset, poll]
sources: [raw/articles/kafka_chapter_4.md]
updated: 2026-04-21
---

# [Chapter 4] 카프카 컨슈머: 카프카에서 데이터 읽기

## 출처

`raw/articles/kafka_chapter_4.md`

## 핵심 요약

카프카에서 데이터를 읽는 애플리케이션은 `KafkaConsumer`를 사용한다. 컨슈머는 컨슈머 그룹 단위로 동작하며, 같은 그룹 내 컨슈머들은 토픽의 파티션을 나눠서 소비한다. 파티션 수를 초과하는 컨슈머는 유휴 상태가 된다.

리밸런스는 컨슈머 그룹 구성 변화(컨슈머 추가/제거) 시 파티션을 재할당하는 프로세스로, 조급한 리밸런스(전체 중단 후 재할당)와 협력적 리밸런스(점진적 재할당)로 구분된다. Kafka 3.1부터는 협력적 리밸런스가 기본값이다.

오프셋 커밋은 `__consumer_offsets` 토픽에 기록되며, 자동 커밋(5초 기본)과 수동 커밋(`commitSync`, `commitAsync`)으로 관리한다. 오프셋의 의미는 "여기까지 읽었다"가 아니라 "여기서부터 읽어라"이다.

## 주요 개념 목록

- [[컨슈머-그룹]] — 동일 토픽을 파티션 단위로 분산 소비
- [[리밸런스]] — 조급한/협력적 리밸런스, 그룹 코디네이터, 정적 멤버십
- [[오프셋-커밋]] — 자동/수동 커밋, commitSync/commitAsync, `__consumer_offsets`
- [[컨슈머-설정]] — fetch 관련, 세션/하트비트, 파티션 할당 전략 등 17개 설정
- [[폴링-루프]] — poll() 내부 동작, prefetch, 스레드 안정성, seek, wakeup
- [[auto-offset-reset]] — latest vs earliest 비교 (query 파생 페이지)
