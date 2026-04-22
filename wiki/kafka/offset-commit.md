---
title: 오프셋 커밋
aliases: [오프셋-커밋]
category: kafka
tags: [kafka, offset, commit, consumer, auto-commit, commitSync, commitAsync]
sources: [raw/articles/kafka_chapter_4.md]
updated: 2026-04-21
---

# 오프셋 커밋

## 개념

카프카에서 파티션의 현재 위치를 업데이트하는 작업이다. 컨슈머는 **파티션에서 성공적으로 처리한 마지막 메시지의 오프셋+1**을 커밋한다.

> **오프셋의 의미**: "여기까지 읽었다"가 아니라 **"여기서부터 읽어라"**

오프셋은 내부 토픽 `__consumer_offsets`에 기록된다. 리밸런스나 재시작 시 이 값을 기준으로 읽기를 재개한다.

## position vs committed offset

| 개념 | 의미 |
|------|------|
| position | 현재 컨슈머 인스턴스가 다음에 읽을 위치 |
| committed offset | 리밸런스/재시작 후 복구 기준 위치 |

## 오프셋 불일치 문제

- **커밋 < 처리 완료**: 마지막 처리 오프셋과 커밋 오프셋 사이 메시지가 **중복 처리**됨
- **커밋 > 처리 완료**: 처리되지 않은 메시지가 **누락**됨

## 자동 커밋

`enable.auto.commit=true` (기본값)

- `auto.commit.interval.ms`마다 (기본 5초) 마지막 `poll()` 오프셋을 커밋
- `poll()` 호출 시 커밋 여부를 확인 후 이전 poll의 마지막 오프셋 커밋
- **중복 처리 가능**: 커밋 주기 내에 크래시 발생 시 마지막 커밋 이후 메시지 재처리
- 중복을 줄이려면 `auto.commit.interval.ms`를 줄이되, 완전히 없애는 것은 불가능

## 수동 커밋 — commitSync()

`enable.auto.commit=false` 설정 후 명시적 커밋

```java
consumer.commitSync();
```

- `poll()`이 리턴한 마지막 오프셋을 커밋
- 성공하거나 재시도 불가능한 실패가 발생할 때까지 **재시도**
- 브로커 응답까지 **블로킹** → 처리량 저하

## 수동 커밋 — commitAsync()

```java
consumer.commitAsync();
// 또는 콜백 포함
consumer.commitAsync((offsets, e) -> {
    if (e != null) log.error("Commit failed for offsets {}", offsets, e);
});
```

- 요청만 보내고 처리를 계속함 → **비블로킹**
- **재시도하지 않음**: 응답 수신 시점에 더 큰 오프셋이 이미 커밋되었을 수 있기 때문
  - 오프셋 2000 커밋 실패 → 3000 커밋 성공 → 2000 재시도 성공 시 3000이 2000으로 덮어씌워져 중복 소비 발생

## 동기/비동기 조합 패턴

```java
try {
    while (!closing) {
        var records = consumer.poll(timeout);
        // 처리...
        consumer.commitAsync(); // 일반 상황: 비동기 커밋
    }
    consumer.commitSync(); // 종료 직전: 동기 커밋으로 마지막 오프셋 보장
} catch (Exception e) {
    log.error("Unexpected error", e);
} finally {
    consumer.close();
}
```

## 특정 오프셋 커밋

배치 처리 중간에 오프셋을 커밋하거나, 파티션별 오프셋을 직접 지정할 때 사용

```java
Map<TopicPartition, OffsetAndMetadata> currentOffsets = new HashMap<>();

for (ConsumerRecord<String, String> record : records) {
    // 처리...
    currentOffsets.put(
        new TopicPartition(record.topic(), record.partition()),
        new OffsetAndMetadata(record.offset() + 1, "no metadata")
    );
    if (count % 1000 == 0)
        consumer.commitAsync(currentOffsets, null);
    count++;
}
```

## 커밋과 처리 순서에 따른 위험

| 순서 | 위험 |
|------|------|
| 처리 → 커밋 | 처리 성공 후 커밋 예외 발생 시 **중복 가능** |
| 커밋 → 처리 | 커밋 성공 후 처리 예외 발생 시 **누락 가능** |

## 관련 항목

- [[리밸런스]] — 리밸런스 전 오프셋 커밋 필요
- [[컨슈머-설정]] — enable.auto.commit, auto.offset.reset, offsets.retention.minutes
- [[폴링-루프]] — poll() 내부에서의 오프셋 처리
