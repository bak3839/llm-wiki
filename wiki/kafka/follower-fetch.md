---
title: 팔로워 Fetch
aliases: [팔로워-페치]
category: kafka
tags: [kafka, consumer, follower-fetch, replica, rack-awareness, KIP-392, preferred-replica]
sources: ["raw/articles/KIP-392 Allow consumers to fetch from closest replica - 한국어 번역.md"]
updated: 2026-04-24
---

# 팔로워 Fetch

## 개념

KIP-392에서 도입된 기능으로, 컨슈머가 파티션 리더가 아닌 **가장 가까운 팔로워 레플리카**에서 fetch할 수 있도록 한다. 멀티 AZ/데이터센터 환경에서 리더 fetch로 인한 크로스-AZ 트래픽 비용과 지연을 줄이는 것이 목적이다.

## 동작 흐름

컨슈머가 직접 레플리카를 선택하지 않고, **브로커가 선호 레플리카를 결정해 Fetch 응답에 포함**한다.

1. 컨슈머가 `client.rack`을 설정한 상태로 Fetch 요청 전송 (`RackId` 필드, v11+)
2. 리더 브로커가 `ReplicaSelector`로 선호 레플리카를 선택
3. Fetch 응답에 `PreferredReadReplica` 필드로 broker ID 포함 (v11+)
4. 컨슈머는 이후 Fetch 요청을 해당 레플리카로 전환

## 리더 vs 팔로워 읽기 트레이드오프

| 읽기 위치 | 장점 | 단점 |
|-----------|------|------|
| 리더에서 읽기 | HW 전파 지연 없음 — 최신 메시지 즉시 노출 | 리더가 멀리 있으면 크로스-AZ 네트워크 비용·지연 증가 |
| 가까운 팔로워에서 읽기 | 네트워크 locality 개선 — 트래픽 비용·지연 감소 가능 | HW 전파 지연 때문에 최신 데이터 노출이 리더보다 늦을 수 있음 |

## High Watermark 전파 지연

팔로워는 리더보다 HW 업데이트가 늦다. 이로 인해 추가 소비 지연이 발생할 수 있다. 완화 방법: 팔로워의 HW가 오래된 경우 리더는 팔로워의 Fetch 요청에 즉시 응답해 HW를 빠르게 전파한다.

## Out of Range 처리 (4가지 케이스)

팔로워 fetch 시 오프셋 불일치 상황이 발생할 수 있다:

| Case | 상황 | 처리 |
|------|------|------|
| 1 | 오프셋이 팔로워에 있으나 아직 커밋 미확인 (HW < offset ≤ LEO) | `OFFSET_NOT_AVAILABLE` → 컨슈머 retry |
| 2 | 오프셋이 커밋됐으나 out-of-sync 팔로워에 없음 (LEO < HW) | `OFFSET_NOT_AVAILABLE` → 컨슈머 retry |
| 3 | 오프셋이 팔로워 log start offset보다 작음 | `OUT_OF_RANGE` + log start offset 반환 → reset policy 적용 |
| 4 | 오프셋이 팔로워 log end offset보다 큼 | KIP-320 offset reconciliation → 리더와 검증 후 재개 |

> Case 3에서 `earliest` reset 시 리더가 아닌 **현재 fetch 중인 팔로워의** log start offset을 기준으로 한다. 레플리카 간 log start offset의 일관성이 보장되지 않기 때문이다.

## ReplicaSelector

브로커에 플러그인 형태로 제공되는 인터페이스. `replica.selector.class` 설정으로 지정한다.

```java
interface ReplicaSelector extends Configurable, Closeable {
    Optional<ReplicaView> select(
        TopicPartition topicPartition,
        ClientMetadata clientMetadata,   // rackId, clientId, address, principal
        PartitionView partitionView      // replicas, leader
    );
}
```

기본 제공 구현체:

| 구현체 | 동작 |
|--------|------|
| `LeaderSelector` (기본값) | 항상 현재 파티션 리더 반환 — 하위 호환 |
| `RackAwareReplicaSelector` | 컨슈머 `rack.id`와 레플리카 `broker.rack`을 매칭, 가장 최신 상태인 레플리카 선택 |

## 설정

```properties
# 컨슈머 설정
client.rack=us-east-1a

# 브로커 설정
replica.selector.class=org.apache.kafka.common.replica.RackAwareReplicaSelector
```

## 프로토콜 변경 요약

| API | 변경 버전 | 추가 필드 |
|-----|-----------|-----------|
| FetchRequest | v11+ | `RackId` (컨슈머 rack 정보) |
| FetchResponse | v11+ | `PreferredReadReplica` (선호 레플리카 broker ID) |
| OffsetsForLeaderEpoch | 신규 | `replica_id` (-1 = 컨슈머, HW로 응답 오프셋 제한) |

## 호환성

하위 호환 지원. 브로커가 팔로워 fetch를 지원하지 않으면 리더 fetch로 자동 fallback.

## 관련 항목

- [[wiki/kafka/consumer-config|컨슈머-설정]] — client.rack, replica.selector.class 설정
- [[wiki/kafka/poll-loop|폴링-루프]] — fetch 메커니즘 전반
- [[wiki/kafka/consumer-group|컨슈머-그룹]] — rack awareness와 파티션 할당
