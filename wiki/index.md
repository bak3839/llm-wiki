# Wiki Index

_마지막 업데이트: 2026-05-17 (Distributed Locks with Redis ingest)_

## DB
- [[wiki/db/innodb-locking|InnoDB-락-유형]] — S/X·인텐션·레코드·갭·넥스트키·인서트인텐션·AUTO-INC 락 총정리
- [[wiki/db/transaction-isolation-levels|트랜잭션-격리-수준]] — READ UNCOMMITTED/COMMITTED/REPEATABLE READ/SERIALIZABLE 비교, semi-consistent read
- [[wiki/db/repeatable-read-phantom-locking-reads|REPEATABLE-READ-팬텀-잠금읽기]] — 잠금 읽기(FOR UPDATE/FOR SHARE)가 MVCC 스냅샷을 우회해 팬텀 리드를 유발하는 이유

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
- [[wiki/kafka/zookeeper-vs-kraft|ZooKeeper-vs-KRaft]] — ZooKeeper 방식과 KRaft 방식의 메타데이터 관리·컨트롤러 선출·운영 복잡성 비교
- [[wiki/kafka/reliability|신뢰성]] — 신뢰성 보장, acks, 복제 팩터 트레이드오프, 언클린 리더 선출, min.insync.replicas, 페이지 캐시
- [[wiki/kafka/kafka-vs-mq|Kafka-vs-MQ]] — Kafka와 전통적 메시지 큐(RabbitMQ 등) 설계 철학·기능 비교
- [[wiki/kafka/consumer-failure-and-recovery|컨슈머-장애-복구]] — 컨슈머 사망 감지, 리밸런스, 재기동 시 오프셋 처리 흐름
- [[wiki/kafka/producer-reliability|프로듀서-신뢰성]] — acks 0/1/all 상세, delivery.timeout.ms, enable.idempotence, 에러 유형 및 처리
- [[wiki/kafka/producer-partitioning|프로듀서-파티셔닝]] — 특정 파티션 지정, 키 해시, 커스텀 파티셔너, Sticky Partitioner
- [[wiki/kafka/idempotent-producer|멱등적-프로듀서]] — 프로듀서ID·시퀀스 넘버로 재시도 중복 방지, 한계와 장애 시나리오
- [[wiki/kafka/kafka-transactions|카프카-트랜잭션]] — 원자적 다수 파티션 쓰기, 좀비 펜싱, EOS, isolation.level

## Network
<!-- 페이지 추가 시 여기에 기록 -->

## OS
<!-- 페이지 추가 시 여기에 기록 -->

## Distributed
- [[wiki/distributed/redis-distributed-lock|Redis-분산-락]] — Redlock 알고리즘: N개 독립 Redis Master 과반수 획득, 단일 인스턴스 한계와 크래시 복구

## Patterns
<!-- 페이지 추가 시 여기에 기록 -->

## Spring
- [[wiki/spring/transactional-event-listener|@TransactionalEventListener]] — 트랜잭션 완료 단계에 바인딩된 이벤트 리스너 (TransactionPhase, fallbackExecution, AFTER_COMMIT 주의사항)

## Sources
- [[wiki/sources/kafka_chapter_4|kafka_chapter_4]] — [Chapter 4] 카프카 컨슈머: 카프카에서 데이터 읽기
- [[wiki/sources/KIP-392-allow-consumers-to-fetch-from-closest-replica|KIP-392]] — 컨슈머가 가장 가까운 레플리카에서 fetch할 수 있도록 허용
- [[wiki/sources/kafka-chapter-6-internals|kafka-chapter-6]] — [Chapter 6] 카프카 내부 메커니즘
- [[wiki/sources/kafka-chapter-7-reliability|kafka-chapter-7]] — [Chapter 7] 신뢰성 있는 데이터 전달
- [[wiki/sources/spring-transactional-event-listener|spring-transactional-event-listener]] — Spring @TransactionalEventListener 공식 문서
- [[wiki/sources/kafka-chapter-7-reliability-2|kafka-chapter-7-2]] — [Chapter 7] 신뢰성 있는 데이터 전달 (2부: 프로듀서/컨슈머 실천 사항, 검증·모니터링)
- [[wiki/sources/kafka-chapter-8-exactly-once|kafka-chapter-8]] — [Chapter 8] 정확히 한 번 의미 구조 (멱등적 프로듀서, 트랜잭션, EOS)
- [[wiki/sources/innodb-locking|innodb-locking]] — MySQL 8.4 InnoDB Locking 공식 레퍼런스
- [[wiki/sources/innodb-transaction-isolation-levels|innodb-isolation-levels]] — MySQL 8.4 InnoDB Transaction Isolation Levels 공식 레퍼런스
- [[wiki/sources/distributed-locks-with-redis|distributed-locks-with-redis]] — Redis 공식 분산 락 패턴 (Redlock 알고리즘)
