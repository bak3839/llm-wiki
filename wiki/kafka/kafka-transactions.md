---
title: 카프카 트랜잭션
aliases: [카프카-트랜잭션, kafka-transactions]
category: kafka
tags: [kafka, transactions, exactly-once, eos, zombie-fencing, transactional-id, isolation-level, atomic-write]
sources: ["raw/articles/Chapter 8 정확히 한 번 의미 구조.md"]
updated: 2026-05-08
---

# 카프카 트랜잭션

## 개요

카프카 트랜잭션은 **스트림 처리 애플리케이션의 '정확히 한 번(Exactly-Once Semantics, EOS)'** 을 보장하기 위해 설계된 기능이다. 카프카 스트림즈가 내부적으로 이 기능을 사용하며, 원하면 트랜잭션 API를 직접 사용할 수도 있다.

> 트랜잭션은 **카프카 내부 쓰기**에 한해 원자성을 보장한다. 외부 DB 쓰기, 이메일 발송 등 부수 효과에는 적용되지 않는다.

## 트랜잭션이 해결하는 문제

스트림 처리 애플리케이션은 **읽기-처리-쓰기** 패턴을 따른다. 이 과정에서 두 가지 중복 시나리오가 발생한다.

### 1. 애플리케이션 크래시로 인한 재처리

```
1. 입력 레코드 처리 후 결과를 출력 토픽에 쓰기 ✅
2. 입력 오프셋 커밋 전 크래시 💥
3. 재시작 후 리밸런스 → 마지막 커밋 오프셋부터 재처리
4. 결과가 출력 토픽에 다시 쓰여짐 → 중복
```

### 2. 좀비 애플리케이션으로 인한 재처리

```
1. 컨슈머가 레코드 배치를 읽어온 후 잠시 멈춤 (GC, 네트워크 지연 등)
2. 하트비트 끊김 → 파티션 다른 컨슈머에 재할당
3. 새 컨슈머가 같은 레코드 처리하고 출력 토픽에 쓰기
4. 이때 멈췄던 기존 컨슈머(좀비)가 깨어나 동일 레코드를 다시 처리 → 중복
```

## 해결 방법: 원자적 다수 파티션 쓰기

오프셋 커밋과 결과 쓰기는 **둘 다 파티션에 메시지를 쓰는 작업**이다. 트랜잭션을 시작해서 양쪽을 묶어 원자적으로 커밋하면, 부분적 성공이 발생하지 않는다.

```
[트랜잭션]
  ├─ 출력 토픽 파티션에 결과 레코드 쓰기
  └─ _consumer_offsets에 입력 오프셋 커밋
  → 커밋 or 중단 (전부 or 없음)
```

## 트랜잭션 프로듀서

### transactional.id

```properties
transactional.id=my-app-instance-0
```

- 재시작해도 값이 유지되어야 함 (애플리케이션 인스턴스를 논리적으로 식별)
- 브로커가 자동 생성하는 `producer.id`(PID)와 달리, 설정값으로 고정
- `initTransactions()` 호출 시 동일 `transactional.id`에 기존 PID를 재할당

### 좀비 펜싱 (Zombie Fencing)

`initTransactions()` 호출 시 브로커가 해당 `transactional.id`의 **에포크 값을 증가**시킨다. 에포크가 낮은 스테일 프로듀서(좀비)가 메시지 전송·커밋·중단을 시도하면 `FencedProducerException`이 발생해 거부된다.

```
에포크 증가: transactional.id "app-0" → epoch 1 → epoch 2 → epoch 3
                                          ↑구버전 좀비는 epoch 1로 접근 → 차단
```

**KIP-447 (Kafka 2.5+)**: 컨슈머 그룹 메타데이터(`generation.id`, `member.id`)를 트랜잭션 오프셋 커밋에 포함시켜 리밸런스 상황에서도 좀비 펜싱이 가능해졌다. 서로 다른 `transactional.id`를 가진 프로듀서들도 같은 파티션에 쓸 수 있게 됨.

## 컨슈머 격리 수준

```properties
isolation.level=read_committed
```

| 값 | 동작 |
|----|------|
| `read_committed` | 커밋된 트랜잭션의 레코드 + 비트랜잭션 레코드만 반환. 진행 중·중단된 트랜잭션 레코드는 숨김 |
| `read_uncommitted` (기본값) | 모든 레코드 반환 |

**Last Stable Offset (LSO)**: `read_committed` 모드에서 진행 중인 트랜잭션이 처음 시작된 시점 이후의 메시지는 트랜잭션이 완료될 때까지 리턴되지 않는다. 트랜잭션이 오래 열려 있으면 컨슈머 지연이 길어진다.

## 트랜잭션 API 사용법

```java
// 프로듀서 설정
Properties producerProps = new Properties();
producerProps.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, transactionalId);

// 컨슈머 설정
Properties consumerProps = new Properties();
consumerProps.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");   // 자동 커밋 비활성화 필수
consumerProps.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");

producer.initTransactions();  // 트랜잭션 ID 등록, 에포크 증가, 진행 중 트랜잭션 중단

while (true) {
    ConsumerRecords<?, ?> records = consumer.poll(Duration.ofMillis(200));
    if (records.count() > 0) {
        producer.beginTransaction();
        try {
            for (ConsumerRecord<?, ?> record : records) {
                producer.send(transform(record));   // 결과를 출력 토픽에 쓰기
            }
            // 오프셋을 트랜잭션의 일부로 커밋 (consumer.commitSync() 사용 금지)
            producer.sendOffsetsToTransaction(consumerOffsets(), consumer.groupMetadata());
            producer.commitTransaction();
        } catch (ProducerFencedException | InvalidProducerEpochException e) {
            // 좀비 판정 → 종료
            throw new KafkaException("Fenced: " + transactionalId);
        } catch (KafkaException e) {
            producer.abortTransaction();
            resetToLastCommittedPositions(consumer);
        }
    }
}
```

### 핵심 주의사항

- `enable.auto.commit=false` 필수
- `consumer.commitSync()` / `consumer.commitAsync()` 직접 호출 금지 — 트랜잭션 보장이 깨짐
- `sendOffsetsToTransaction()`에 `consumer.groupMetadata()` 전달 (KIP-447, Kafka 2.5+)
- `commitTransaction()` 성공 시 결과 레코드 쓰기 + 오프셋 커밋이 함께 확정됨
- `ProducerFencedException` 수신 시 현재 인스턴스 즉시 종료

## 트랜잭션의 한계

| 시나리오 | 보장 여부 |
|---------|----------|
| 카프카 → 카프카 (스트림 처리) | ✅ 정확히 한 번 |
| 카프카 쓰기 중 이메일/REST API 호출 | ❌ 외부 효과는 보장 불가 |
| 카프카 → 외부 DB 쓰기 | ❌ DB와 원자적 커밋 불가 |
| 클러스터 간 복제 (MirrorMaker 2) | ❌ 트랜잭션 경계 유실 |
| 발행/구독 패턴 (단순 컨슈머) | ⚠️ `read_committed`는 보장하나 EOS는 아님 |

> ⚠️ 주의: 메시지를 쓴 뒤 커밋 전에 다른 애플리케이션의 응답을 기다리는 패턴은 데드락을 유발한다. 커밋되지 않은 트랜잭션의 메시지는 `read_committed` 컨슈머에 보이지 않기 때문이다.

## 내부 동작: 트랜잭션 코디네이터

각 브로커가 일부 트랜잭션의 코디네이터 역할을 담당한다 (컨슈머 그룹 코디네이터와 유사한 구조). 알고리즘은 **찬디-램포드 스냅샷 알고리즘**에서 영감을 받았으며, 2PC + `__transaction_state` 내부 토픽으로 구현된다.

```
1. initTransactions() → 코디네이터에 transactional.id 등록, 에포크 증가
2. beginTransaction() → 프로듀서 로컬 상태만 변경 (브로커 통신 없음)
3. send() → 새 파티션에 쓸 때마다 AddPartitionsToTxn 요청 → 트랜잭션 로그 기록
4. sendOffsetsToTransaction() → 코디네이터가 컨슈머 그룹 코디네이터에 오프셋 커밋
5. commitTransaction() → EndTxn 요청 → 로그에 커밋 시도 기록 → 모든 파티션에 커밋 마커 쓰기 → 로그에 완료 기록
```

코디네이터 크래시 시 새 코디네이터가 트랜잭션 로그를 보고 미완료 커밋을 이어서 처리한다. `transaction.timeout.ms` 내에 커밋/중단이 없으면 자동 중단.

## 성능 특성

- 트랜잭션 오버헤드는 **포함된 메시지 수와 무관한 고정 비용** (트랜잭션ID 등록, 커밋 마커 추가 등)
- 따라서 **트랜잭션당 메시지 수를 늘릴수록 처리량 효율이 높아진다**
- `read_committed` 컨슈머는 `read_uncommitted`보다 약간 지연되어 메시지를 읽는다
- 트랜잭션이 오래 열려 있으면 LSO가 이동하지 않아 컨슈머 랙이 증가한다

## 관련 항목

- [[wiki/kafka/idempotent-producer|멱등적-프로듀서]] — 단일 프로듀서 내 재시도 중복 방지
- [[wiki/kafka/producer-reliability|프로듀서-신뢰성]] — acks, 재시도, 에러 처리
- [[wiki/kafka/offset-commit|오프셋-커밋]] — 일반 오프셋 커밋 전략
- [[wiki/kafka/replication|복제]] — ISR, high watermark
- [[wiki/spring/transactional-event-listener|@TransactionalEventListener]] — Spring 트랜잭션 이벤트 처리
