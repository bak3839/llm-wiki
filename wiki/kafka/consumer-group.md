---
title: 컨슈머 그룹
aliases: [컨슈머-그룹]
category: kafka
tags: [kafka, consumer, consumer-group, partition, scalability]
sources: [raw/articles/kafka_chapter_4.md]
updated: 2026-04-21
---

# 컨슈머 그룹

## 개념

카프카 컨슈머는 보통 **컨슈머 그룹**의 일부로 동작한다. 동일한 컨슈머 그룹에 속한 여러 컨슈머가 같은 토픽을 구독하면, 각 컨슈머는 서로 다른 파티션에서 메시지를 읽는다.

- 컨슈머 그룹에 컨슈머를 추가하는 것이 카프카 읽기 처리량을 확장하는 주된 방법이다.
- 파티션 수를 초과하는 컨슈머를 추가해도 의미가 없다 — 초과분은 유휴(idle) 상태가 된다.
- 여러 애플리케이션이 같은 토픽을 독립적으로 소비하려면 각각 별도의 컨슈머 그룹을 사용한다.

## 파티션-컨슈머 할당 규칙

| 상황 | 결과 |
|------|------|
| 컨슈머 1개, 파티션 4개 | 컨슈머 1이 4개 파티션 모두 소비 |
| 컨슈머 2개, 파티션 4개 | 각 컨슈머가 파티션 2개 소비 |
| 컨슈머 4개, 파티션 4개 | 컨슈머당 파티션 1개 |
| 컨슈머 5개, 파티션 4개 | 컨슈머 1개는 유휴 |

## 컨슈머 그룹 분리

- 서로 다른 컨슈머 그룹은 동일 토픽의 모든 메시지를 각자 독립적으로 수신한다.
- 카프카는 성능 저하 없이 많은 수의 컨슈머와 컨슈머 그룹으로 확장 가능하다.

## 컨슈머 생성 필수 설정

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("group.id", "my-group");
```

## 정적 그룹 멤버십

`group.instance.id`를 설정하면 컨슈머가 **정적 멤버**가 된다.

- 재시작 시 리밸런스 없이 이전에 할당받았던 파티션을 그대로 재할당받는다.
- 그룹 코디네이터가 파티션 할당을 캐시해두기 때문에 가능하다.
- 세션 타임아웃(`session.timeout.ms`) 내에 재시작하면 리밸런스가 발생하지 않는다.
- 로컬 상태나 캐시를 파티션별로 유지하는 애플리케이션에 유리하다.
- **단점**: 컨슈머가 내려간 동안 해당 파티션은 처리되지 않으며, 재시작 후 밀린 메시지를 따라잡아야 한다.

## 관련 항목

- [[리밸런스]] — 컨슈머 그룹 구성 변경 시 파티션 재할당 프로세스
- [[컨슈머-설정]] — group.id, group.instance.id, session.timeout.ms 등
- [[폴링-루프]] — 컨슈머의 메시지 수신 방식
