---
title: 폴링 루프
category: kafka
tags: [kafka, consumer, poll, polling-loop, thread-safety, seek, wakeup]
sources: [raw/articles/kafka_chapter_4.md]
updated: 2026-04-21
---

# 폴링 루프

## 개념

카프카 컨슈머 API의 핵심은 브로커에서 추가 데이터가 들어왔는지 **폴링하는 루프**다.

```java
Duration timeout = Duration.ofMillis(100);

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(timeout);

    for (ConsumerRecord<String, String> record : records) {
        // record.topic(), partition(), offset(), key(), value() 사용 가능
    }
}
```

## poll()의 역할

- 첫 번째 `poll()` 호출 시 그룹 코디네이터를 찾아 컨슈머 그룹에 참가하고 파티션을 할당받음
- 리밸런스와 관련 콜백 처리도 `poll()` 내부에서 실행됨
- `poll()`을 호출하지 않으면 컨슈머가 죽은 것으로 간주됨 (`max.poll.interval.ms` 참고)

## poll() 내부 동작

```
poll() 호출
 └─ pollForFetches()
    ├─ collectFetch() → 내부 버퍼 확인
    │  ├─ 있으면 즉시 반환
    │  └─ 없으면 → sendFetches() → client.poll() 블로킹 대기 → collectFetch()
    └─ 레코드 수신 후 반환 직전에 → sendFetches() ← prefetch
```

### Prefetch 최적화

- 레코드를 반환하기 직전, 다음 `FetchRequest`를 브로커에 **미리 전송**
- 애플리케이션이 현재 배치를 처리하는 동안 브로커가 다음 배치를 준비
- 다음 `poll()` 호출 시 블로킹 대기 시간 단축 가능
- 단, prefetch 응답이 아직 오지 않았다면 여전히 블로킹

### 내부 버퍼와 네트워크 요청

`poll()` 한 번이 항상 네트워크 왕복을 의미하지 않는다.

- `FetchResponse`로 100개 레코드 수신 → 내부 버퍼에 저장
- `max.poll.records=20` 설정 시 → 한 번에 20개만 반환, 80개는 버퍼에 유지
- 다음 `poll()` 호출 시 버퍼에서 꺼내어 반환 (네트워크 요청 없음)

### Wakeup 안전 구간

`pollForFetches()` 후 offset(consumed position)이 이미 전진한다. 이 구간에서 wakeup이나 예외가 발생하면 레코드 유실이 발생할 수 있으므로, 반환 직전 구간에서는 `client.transmitSends()`만 호출하고 wakeup 체크를 하는 `client.poll()`은 호출하지 않는다.

## 스레드 안정성

- 하나의 스레드당 하나의 컨슈머 원칙
- 같은 컨슈머를 다수의 스레드가 공유하는 것은 불안전
- 동일 그룹에 여러 컨슈머를 운용하려면 스레드를 여러 개 띄워야 함

### Spring Kafka @KafkaListener

```java
@KafkaListener(topics = "orders")
public void listen(String message) {
    // poll()로 가져온 레코드를 처리하는 콜백
}
```

- Spring Kafka의 `KafkaMessageListenerContainer`가 폴링 루프를 대신 실행
- 설정된 ack 모드에 따라 오프셋 커밋 처리

## 특정 오프셋 탐색

```java
// 파티션 처음부터
consumer.seekToBeginning(partitions);

// 파티션 끝(최신)부터
consumer.seekToEnd(partitions);

// 1시간 전 오프셋으로 탐색
Map<TopicPartition, Long> timestampMap = consumer.assignment()
    .stream()
    .collect(Collectors.toMap(tp -> tp, tp -> oneHourEarlier));
Map<TopicPartition, OffsetAndTimestamp> offsetMap = consumer.offsetsForTimes(timestampMap);
offsetMap.forEach((tp, ot) -> consumer.seek(tp, ot.offset()));
```

## 폴링 루프 종료

```java
// 다른 스레드에서 호출 (유일하게 스레드 안전한 컨슈머 메서드)
consumer.wakeup();
```

- `wakeup()`은 대기 중인 `poll()`에서 `WakeupException`을 발생시킴
- 반드시 종료 전 `consumer.close()` 호출 필요
  - 오프셋 커밋 수행
  - 그룹 코디네이터에 leave group 전송 → 즉시 리밸런스 (세션 타임아웃 대기 불필요)

## 독립 실행 컨슈머 (컨슈머 그룹 없이)

하나의 컨슈머가 특정 파티션만 읽어야 할 때 `subscribe()` 대신 `assign()` 사용

```java
List<PartitionInfo> partitionInfos = consumer.partitionsFor("topic");
List<TopicPartition> partitions = partitionInfos.stream()
    .map(p -> new TopicPartition(p.topic(), p.partition()))
    .collect(Collectors.toList());

consumer.assign(partitions);
```

- 리밸런스 없음, 자동 파티션 추가 감지 없음
- 새 파티션 추가 시 직접 확인(`consumer.partitionsFor()` 주기적 호출) 또는 재시작 필요
- 오프셋 커밋을 위해 `group.id`는 여전히 설정 필요

## 관련 항목

- [[컨슈머-그룹]] — 첫 번째 poll() 시 그룹 참가 및 파티션 할당
- [[오프셋-커밋]] — poll() 내에서의 오프셋 처리
- [[컨슈머-설정]] — max.poll.interval.ms, max.poll.records, fetch 관련 설정
