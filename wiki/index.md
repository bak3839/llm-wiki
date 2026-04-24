# Wiki Index

_마지막 업데이트: 2026-04-24 (Chapter 6)_

## DB
<!-- 페이지 추가 시 여기에 기록 -->

## Kafka
- [[wiki/kafka/consumer-group|컨슈머-그룹]] — 동일 토픽을 파티션 단위로 분산 소비하는 컨슈머 그룹 개념
- [[wiki/kafka/rebalance|리밸런스]] — 조급한/협력적 리밸런스, 그룹 코디네이터, 파티션 재할당
- [[wiki/kafka/offset-commit|오프셋-커밋]] — 자동/수동 커밋 전략 및 `__consumer_offsets`
- [[wiki/kafka/consumer-config|컨슈머-설정]] — KafkaConsumer 주요 설정 레퍼런스 (17개 설정)
- [[wiki/kafka/poll-loop|폴링-루프]] — poll() 내부 동작, prefetch, seek, wakeup, 독립 실행 컨슈머
- [[wiki/kafka/auto-offset-reset|auto-offset-reset]] — latest vs earliest 비교: 적용 조건, 시작 위치, 선택 기준
- [[wiki/kafka/follower-fetch|팔로워-페치]] — 가장 가까운 레플리카 fetch, ReplicaSelector, out-of-range 처리 (KIP-392)
- [[wiki/kafka/cluster-membership|클러스터-멤버십]] — ZooKeeper ephemeral 노드를 통한 브로커 등록·이탈 감지
- [[wiki/kafka/controller|컨트롤러]] — 파티션 리더 선출, epoch, 좀비 컨트롤러, KRaft
- [[wiki/kafka/replication|복제]] — 리더/팔로워 레플리카, ISR, high watermark, 선호 리더
- [[wiki/kafka/request-handling|요청-처리]] — acceptor/processor/IO 스레드, acks, zero-copy, Fetch Session Cache

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
- [[wiki/sources/kafka_chapter_4|kafka_chapter_4]] — [Chapter 4] 카프카 컨슈머: 카프카에서 데이터 읽기
- [[wiki/sources/KIP-392-allow-consumers-to-fetch-from-closest-replica|KIP-392]] — 컨슈머가 가장 가까운 레플리카에서 fetch할 수 있도록 허용
- [[wiki/sources/kafka-chapter-6-internals|kafka-chapter-6]] — [Chapter 6] 카프카 내부 메커니즘
