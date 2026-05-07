---
title: 프로듀서 파티셔닝
aliases: [프로듀서-파티셔닝]
category: kafka
tags: [kafka, producer, partitioner, partition, key, sticky-partitioner, murmur2]
sources: []
updated: 2026-05-05
---

# 프로듀서 파티셔닝

## 파티션 결정 방법 (우선순위 순)

| 우선순위 | 방법 | 설명 |
|---------|------|------|
| 1 | 파티션 직접 지정 | `ProducerRecord`에 파티션 번호 명시 |
| 2 | 커스텀 파티셔너 | `Partitioner` 인터페이스 구현 |
| 3 | 키 해시 | `murmur2(key) % 파티션 수` |
| 4 | 키 없음 | Sticky Partitioner (Kafka 2.4+) / 라운드 로빈 (이전) |

## 방법 1: 파티션 직접 지정

```java
ProducerRecord<String, String> record = new ProducerRecord<>(
    "my-topic",
    2,           // 파티션 번호
    "my-key",
    "my-value"
);
producer.send(record);
```

키 해시 로직을 무시하고 지정한 파티션으로 전송한다.

## 방법 2: 키 기반 파티셔닝 (기본)

```java
ProducerRecord<String, String> record = new ProducerRecord<>(
    "my-topic",
    "order-123",  // 키
    "event-data"
);
```

- `murmur2(key) % 파티션 수`로 파티션 결정
- **같은 키 → 항상 같은 파티션** (파티션 수가 변하지 않는 한)
- 파티션 내 순서 보장 활용: 동일 주문 ID, 동일 사용자 ID 등

## 방법 3: 커스텀 파티셔너

```java
public class CustomPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        int numPartitions = cluster.partitionCountForTopic(topic);
        if (keyBytes != null && new String(keyBytes).startsWith("vip-")) {
            return 0;  // VIP 이벤트는 파티션 0으로
        }
        return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
    }
    @Override public void close() {}
    @Override public void configure(Map<String, ?> configs) {}
}
```

```properties
partitioner.class=com.example.CustomPartitioner
```

## 키 없을 때의 기본 동작

| Kafka 버전 | 동작 |
|-----------|------|
| 2.4 이전 | 라운드 로빈 |
| 2.4+ (기본) | **Sticky Partitioner** — 배치가 채워질 때까지 동일 파티션에 누적 후 이동 → 배치 효율 향상 |

## 주의사항

- 파티션 수를 늘리면 키 해시 결과가 달라져 **같은 키가 다른 파티션으로 라우팅**될 수 있음 → 순서 보장 전략 재검토 필요
- 파티션 직접 지정 시 해당 파티션의 리더 장애 → 에러 발생 (재시도 가능 에러 `LEADER_NOT_AVAILABLE`)

## 관련 항목

- [[wiki/kafka/producer-reliability|프로듀서-신뢰성]] — acks 설정, 재시도, 에러 처리
- [[wiki/kafka/replication|복제]] — 파티션 리더, ISR
- [[wiki/kafka/consumer-group|컨슈머-그룹]] — 파티션-컨슈머 할당 관계
