---
title: "KIP-392: 컨슈머가 가장 가까운 레플리카에서 fetch할 수 있도록 허용"
source: "https://cwiki.apache.org/confluence/display/KAFKA/KIP-392%3A+Allow+consumers+to+fetch+from+closest+replica"
translation_of: "raw/articles/KIP-392 Allow consumers to fetch from closest replica - Apache Kafka - Apache Software Foundation.md"
author:
  - "[[Jason Gustafson]]"
published:
created: 2026-04-24
description: "KIP-392 한국어 번역"
tags:
  - "clippings"
  - "translation"
---

## 상태

**현재 상태**: *Accepted*

**논의 스레드**:

**JIRA**: [KAFKA-8443](https://issues.apache.org/jira/browse/KAFKA-8443) - 컨슈머를 위해 선호 읽기 레플리카를 브로커가 선택할 수 있도록 허용 Resolved

위키에 댓글을 남기기보다는 메일링 리스트에서 논의를 이어가 주세요. 위키 논의는 금방 다루기 어려워집니다.

## 동기

Kafka 클러스터가 여러 데이터센터에 걸쳐 있는 경우는 흔하다. 예를 들어 일반적인 배포 형태 중 하나는 AWS 리전 안에서 각 가용 영역을 데이터센터처럼 취급하는 것이다. 현재 Kafka는 이 시나리오에서 레플리카 배치를 제어하는 데 사용할 수 있는 기본적인 rack awareness 지원을 제공한다. 이때 가용 영역을 rack으로 취급한다. 하지만 현재 컨슈머는 리더에서만 fetch할 수 있도록 제한되어 있으므로, 비용이 큰 데이터센터 간 트래픽을 줄이기 위해 지역성을 활용할 쉬운 방법이 없다. 이 공백을 해결하기 위해, 여기서는 컨슈머가 가장 가까운 레플리카에서 fetch할 수 있도록 허용하는 방안을 제안한다.

## 제안 변경 사항

가장 가까운 레플리카에서 fetch하는 기능을 지원하려면 1) Fetch 프로토콜에 팔로워 fetch 지원을 추가하고, 2) 주어진 컨슈머에 대해 "가장 가까운" 레플리카를 찾는 메커니즘이 필요하다.

## 팔로워 Fetch

컨슈머가 임의의 레플리카에서 fetch할 수 있도록 fetch 프로토콜을 확장하는 것을 제안한다. 리더에서 fetch할 때와 마찬가지로, replication 프로토콜은 커밋된 데이터만 컨슈머에게 반환되도록 보장한다. 이는 리더의 fetch 응답을 통해 high watermark가 전파되는 것에 의존한다. 이 전파 지연 때문에 팔로워 fetch는 더 높은 지연 시간을 가질 수 있다. 또한 팔로워 fetch는 out-of-sync 레플리카가 fetch를 수신할 경우 가짜 out of range 오류가 발생할 가능성을 도입한다. 아래 다이어그램은 가능한 여러 경우를 보여준다.

![](https://cwiki.apache.org/confluence/download/attachments/95653762/Follower%20Fetching%20%281%29.png?version=1&modificationDate=1545431950000&api=v2)

이 다이어그램에서 로그의 초록색 부분은 레플리카에서 로컬로 사용 가능하면서 동시에 커밋된 것으로 알려진 레코드를 나타낸다. 이 레코드들은 언제든 컨슈머에게 안전하게 반환될 수 있다. 노란색 부분은 로컬에서 사용할 수 없거나 커밋된 것으로 알려지지 않은 레코드를 나타낸다. 즉, ISR의 모든 멤버에 성공적으로 복제되지 않은 레코드다. 이 레코드들은 high watermark가 전진한 것으로 알려지기 전까지 반환될 수 없다.

리더는 항상 어떤 레플리카보다 먼저 high watermark 업데이트를 확인한다. 따라서 in-sync 레플리카에는 커밋되었지만 아직 소비 가능하지 않은 데이터가 있을 수 있다. out-of-sync 레플리카의 경우 high watermark는 일반적으로 해당 레플리카의 log end offset보다 위에 있으므로, 커밋되었지만 아직 로컬에서 사용할 수 없는 데이터가 있을 수 있다.

로그 시작 오프셋은 리더와 팔로워 사이에서 일관되게 유지되지 않는다는 점에 유의하라. ISR에 속한 팔로워도 마찬가지다. 이는 retention이 각 레플리카에서 독립적으로 적용되기 때문이다. 이 지점은 아래에서 더 자세히 논의한다.

### High watermark 전파

팔로워에서 fetch할 때 약간의 추가 지연 시간은 예상되지만, high watermark 전파에 불필요한 지연이 없도록 해야 한다. [KIP-227](https://cwiki.apache.org/confluence/display/KAFKA/KIP-227%3A+Introduce+Incremental+FetchRequests+to+Increase+Partition+Scalability)의 개선 이후, 리더는 fetch session을 사용해 팔로워의 high watermark를 추적할 수 있다. 현재 파티션에 새 데이터가 쓰이지 않으면 팔로워의 high watermark 업데이트가 `replica.fetch.max.wait.ms`만큼 지연될 수 있다.

fetch 만족 조건 로직을 변경해 high watermark 업데이트를 고려하도록 제안한다. 팔로워가 오래된 high watermark를 가지고 있다면 리더는 fetch에 즉시 응답한다. 특히 fetch session의 첫 요청은 요청된 파티션에 대해 팔로워가 최신 high watermark를 갖도록 보장하기 위해 리더에서 block되지 않는다. 이 변경의 부수 효과는 리더 선출 직후 high watermark의 단조성을 보존하기 위해 클라이언트에 high watermark를 숨겨야 하는 시간 창을 줄인다는 것이다. 자세한 내용은 [KIP-207](https://cwiki.apache.org/confluence/display/KAFKA/KIP-207%3A+Offsets+returned+by+ListOffsetsResponse+should+be+monotonically+increasing+even+during+a+partition+leader+change)을 참고하라.

### Out of range 처리

팔로워 fetch의 주된 과제는 컨슈머가 리더에서 커밋된 오프셋을 관찰했지만, 그 오프셋이 아직 팔로워에서는 사용 가능하지 않을 수 있다는 점이다. 오래된 로그 메타데이터에 의존하는 팔로워에서 컨슈머가 유효한 오프셋을 fetch하려고 할 때 기대 동작을 정의해야 한다.

고려할 경우는 네 가지다. 각 경우는 위 다이어그램에 표시되어 있다.

**Case 1 (커밋되지 않은 오프셋)**: 오프셋은 fetch를 받은 레플리카에서 로컬로 사용 가능하지만, 커밋된 것으로 알려지지 않았다. 사실 브로커에는 이미 이 경우를 처리하는 로직이 있다. 레플리카가 리더로 선출되면 현재 ISR로부터 fetch를 받기 전까지 high watermark의 실제 값이 무엇이어야 하는지 알 수 없다. 컨슈머가 마지막으로 업데이트된 high watermark 값과 log end offset 사이의 오프셋에서 fetch하면, 현재 브로커는 빈 record set을 반환한다. 이는 레코드가 커밋된 것으로 알려지기 전까지 반환되지 않도록 보장한다.

유사한 문제는 [KIP-207](https://cwiki.apache.org/confluence/display/KAFKA/KIP-207%3A+Offsets+returned+by+ListOffsetsResponse+should+be+monotonically+increasing+even+during+a+partition+leader+change)에서 다뤄졌다. 그 전에는 막 선출된 리더가 오래된 high watermark 값을 반환할 수 있었다. KIP-207이 채택되면서 브로커는 실제 high watermark를 결정할 수 있을 때까지 대신 OFFSET_NOT_AVAILABLE 오류 코드를 반환한다. 이는 fetch 동작과 약간 일관되지 않는다. 빈 record set을 반환하면서 오류를 반환하지 않는 의도치 않은 부작용은, 커밋된 것으로 알려지기 전에 오프셋을 노출할 수 있다는 점이다.

이 KIP에서 만드는 개선 중 하나는 이 두 API의 동작을 일관되게 만드는 것이다. 브로커가 로컬 high watermark와 log end offset 사이의 오프셋에 대한 fetch를 받으면, 리더이든 팔로워이든 OFFSET_NOT_AVAILABLE 오류 코드를 반환한다. 컨슈머는 이를 retry로 처리한다.

**Case 2 (사용 불가능한 오프셋)**: 오프셋은 로컬에서 사용할 수 없지만, 커밋된 것으로 알려져 있다. 이는 out-of-sync 레플리카가 fetch를 수신한 경우 발생할 수 있다. out-of-sync 레플리카도 계속 리더에서 fetch하며, 자신의 log end offset보다 큰 high watermark 값을 수신했을 수 있다. 따라서 log end offset과 리더에게서 받은 high watermark 사이의 오프셋에서 fetch하면, 레플리카가 데이터를 가지고 있지는 않지만 해당 오프셋은 커밋된 것으로 알려져 있다.

이상적으로 컨슈머는 ISR에서만 fetch해야 하지만, ISR 상태가 각 레플리카에 전파되는 데 시간이 걸리기 때문에 ISR 밖에서 fetch하는 것을 막을 수는 없다. 여기서는 이 경우에도 OFFSET_NOT_AVAILABLE을 반환하도록 제안한다. 이렇게 하면 컨슈머가 retry하게 된다. 일반적인 경우 out-of-sync 레플리카는 짧은 시간 후 ISR로 돌아올 것으로 예상한다. 반면 레플리카가 더 오랜 시간 out-of-sync 상태로 남아 있다면, 컨슈머는 일정 시간이 지난 뒤 새 레플리카를 찾을 기회를 갖게 된다. 이 지점은 컨슈머가 선호 레플리카를 찾는 방법을 다루는 아래 섹션에서 더 자세히 논의한다.

**Case 3 (오프셋이 너무 작음)**: 한 레플리카에서 유효한 오프셋이 다른 레플리카의 log start offset보다 작을 수 있다. 이 경우를 감지하기 위해 fetch 응답에 작은 변경을 제안한다. OUT_OF_RANGE 오류를 반환할 때 브로커는 현재 log start offset과 high watermark를 제공한다. 컨슈머는 이 값들을 fetch offset과 비교해 오프셋이 너무 작은지 또는 너무 큰지 판단할 수 있다.

이 경우를 클라이언트에서 추론하는 것은 log start offset의 일관성에 대한 보장이 없기 때문에 어렵다. 그러나 레플리카들의 log start offset은 상대적으로 가까울 것으로 예상하므로, 추가 처리를 통해 얻는 이점은 제한적이라고 본다. 여기서는 이 오류를 처리할 때 KIP-320에 개요가 제시된 offset reconciliation을 건너뛰고 단순히 offset reset policy에 의존하는 것을 제안한다.

offset reset policy가 "earliest"로 설정된 경우에는 현재 fetch 중인 레플리카의 log start offset을 찾아야 한다는 점에 유의하라. 리더의 log start offset이 우리가 fetch 중인 팔로워에 존재한다는 보장은 없으며, 수렴을 기다리는 동안 컨슈머가 reset loop에 빠지기를 원하지 않는다. 따라서 처리를 단순화하기 위해 컨슈머는 자신이 fetch하는 레플리카의 earliest offset을 사용한다.

요약하면, 이 경우를 처리하기 위해 컨슈머는 다음 단계를 수행한다.

1. reset policy가 "earliest"이면, out of range 오류를 발생시킨 현재 레플리카의 log start offset을 fetch한다.
2. reset policy가 "latest"이면, 리더에서 log end offset을 fetch한다.
3. reset policy가 "none"이면, 예외를 발생시킨다.

**Case 4 (오프셋이 너무 큼)**: 오프셋은 레플리카에서 로컬로 사용할 수 없고, 커밋된 것으로 알려져 있지도 않다. Case 2와 마찬가지로 out-of-sync 레플리카가 리더에게서 마지막으로 받은 high watermark 값보다 큰 오프셋에서 fetch를 수신하면 이런 일이 발생할 수 있다. 이 경우 레플리카는 OFFSET_OUT_OF_RANGE를 반환하는 것 외에 선택지가 없다. 일반적으로 팔로워에서 fetch할 때 out of range 오류는 확정적인 것으로 받아들일 수 없다. 오류를 발생시키기 전에 컨슈머는 해당 오프셋이 올바른지 검증해야 한다.

사실 이 경우를 처리하는 로직은 이미 [KIP-320](https://cwiki.apache.org/confluence/display/KAFKA/KIP-320%3A+Allow+fetchers+to+detect+and+handle+log+truncation)에 개요가 제시되어 있다. 컨슈머는 현재 오프셋을 리더와 reconcile한 다음 fetch를 재개한다. 이를 통해 같은 레플리카에서 fetch를 재개하기 전에 ISR 변경을 감지할 기회를 얻는다.

OffsetsForLeaderEpoch를 사용해 오프셋이 여전히 유효한지 판단할 수 있다는 점은 즉시 명확하지 않을 수 있다. 컨슈머는 현재 fetch offset에 해당하는 epoch의 end offset을 리더에게 요청한다. fetch offset이 unclean leader election에서 truncate되었다면, OffsetsForLeaderEpoch 응답은 1) 동일한 epoch와 fetch offset보다 작은 end offset, 또는 2) 더 작은 epoch와 더 작은 fetch offset을 반환한다. 어느 경우든 반환된 end offset은 fetch offset보다 작다. 반대로 오프셋이 존재한다면 응답에는 동일한 epoch와 fetch offset보다 큰 end offset이 포함된다.

따라서 OFFSET_OUT_OF_RANGE 오류 코드를 처리하기 위해 컨슈머는 다음 단계를 수행한다.

1. OffsetForLeaderEpoch API를 사용해 현재 position을 리더와 검증한다.
	1. fetch offset이 여전히 유효하면, 메타데이터를 새로고침하고 fetch를 계속한다.
		2. truncation이 감지되면, KIP-320의 단계에 따라 오프셋을 reset하거나 truncation 오류를 발생시킨다.
		3. 그렇지 않으면, 위 Case 3과 동일한 단계를 따른다.

현재 fetch offset에 해당하는 epoch가 없다면, truncation check를 건너뛰고 FETCH_OFFSET_TOO_SMALL 경우에 사용한 것과 동일한 처리를 사용한다. 기본적으로 reset policy에 의존한다. 실제로 ISR 전파 지연으로 인한 out of range 오류는 매우 드물 것으로 보인다. 이는 컨슈머가, 여전히 ISR에 속한 것으로 간주되는 팔로워보다 먼저 커밋된 오프셋을 보아야만 발생하기 때문이다.

**요약**: 여기서의 변경 사항은 다음뿐이다.

1. 오프셋이 존재하는 것으로 알려져 있지만 사용할 수 없거나 커밋된 것으로 알려지지 않은 경우, 브로커는 OFFSET_NOT_AVAILABLE을 반환한다.
2. out of range 오류에 대해 fetch offset이 log start offset보다 작으면 KIP-320의 offset truncation check를 건너뛴다.
3. earliest offset으로 reset할 때는 우리가 fetch할 레플리카에 질의해야 한다.

## 선호 팔로워 찾기

컨슈머 입장에서의 문제는 어떤 레플리카가 선호되는지를 알아내는 것이다. 선택지는 두 가지다. 하나는 컨슈머가 브로커의 메타데이터, 즉 rackId와 host 정보를 사용해 선호 레플리카를 직접 찾도록 하는 것이고, 다른 하나는 클라이언트의 정보를 기반으로 브로커가 선호 레플리카를 결정하도록 하는 것이다. 여기서는 후자를 제안한다.

클라이언트에 대해 어떤 레플리카가 선호되는지를 브로커가 결정하도록 하는 이점은 부하를 고려할 수 있다는 것이다. 예를 들어 브로커는 부하를 분산하기 위해 가까운 레플리카들 사이에서 round robin을 수행할 수 있다. 이와 같은 고려 사항은 많이 가능하므로, 사용자가 로직을 처리하기 위해 브로커에 ReplicaSelector 플러그인을 제공할 수 있도록 제안한다. 이는 `replica.selector.class` 설정을 통해 브로커에 노출된다. 이 플러그인을 사용하기 위해 클라이언트가 자신의 위치 정보를 제공할 수 있도록 Fetch API를 확장한다. 응답에서 브로커는 fetch할 선호 레플리카를 표시한다.

## 공개 인터페이스

## Consumer API

브로커의 위치를 식별하기 위해 "broker.rack" 설정을 사용하므로, 컨슈머에도 유사한 설정을 추가한다. 이는 아래에 문서화된 대로 Fetch 요청에서 사용된다.

이 제안에서 컨슈머는 `rack.id`가 제공되었는지 여부와 관계없이 Metadata 요청이 반환한 선호 레플리카에서 항상 fetch한다.

<table><colgroup><col> <col> <col></colgroup><thead><tr><th></th><th></th><th colspan="1"></th></tr></thead><tbody><tr><td>client.rack</td><td><nullable string></td><td colspan="1">null</td></tr></tbody></table>

## Broker API

fetch할 선호 레플리카를 결정하는 사용자 정의 로직을 제공할 수 있도록, "replica.selector.class" 설정을 통해 구성되는 새 플러그인 인터페이스를 노출한다.

<table><colgroup><col> <col> <col></colgroup><thead><tr><th></th><th></th><th colspan="1"></th></tr></thead><tbody><tr><td>replica.selector.class</td><td>class name of ReplicaSelector implementation</td><td colspan="1">LeaderSelector</td></tr></tbody></table>

기본적으로 Kafka는 항상 현재 파티션 리더를 반환하는 구현체를 사용한다. 단, 리더가 존재하는 경우에 한한다. 이는 하위 호환성을 위한 것이다.

ReplicaSelector 인터페이스는 아래와 같다. Metadata API를 통해 전달된 rackId와 기타 연결 관련 정보를 포함하는 클라이언트 메타데이터를 전달한다.

```java
interfaceClientMetadata {
  String rackId();
  String clientId();
  InetAddress clientAddress();
  KafkaPrincipal principal();
  String listenerName();
}
 
interfaceReplicaView {
  Node endpoint();
  longlogEndOffset();
  longtimeSinceLastCaughtUpMs();
}
 
interfacePartitionView {
  Set<ReplicaView> replicas();
  Optional<ReplicaView> leader();
}
 
interfaceReplicaSelector extendsConfigurable, Closeable {
    /**
     * Select the preferred replica a client should use for fetching.
     * If no replica is available, this method should return an empty optional.
     */
    Optional<ReplicaView> select(TopicPartition topicPartition, ClientMetadata metadata, PartitionView partitionView);
 
    /**
     * Optional custom configuration
     */
    defaultvoidconfigure(Map<String, ?> configs) {}
 
    /**
     * Optional shutdown hook
     */
    defaultvoidclose() {}
}
```

기본 제공 구현체는 두 가지를 제공한다. 그중 하나는 위에서 언급한 LeaderSelector다.

```java
classLeaderSelector implementsReplicaSelector {
 
  Optional<ReplicaView> select(TopicPartition topicPartition, ClientMetadata metadata, PartitionView partitionView) {
    returnpartitionView.leader();
  }
}
```

두 번째 구현체는 클라이언트가 제공한 rackId를 사용하며, 레플리카의 rackId와 정확히 매칭한다.

```java
classRackAwareReplicaSelector implementsReplicaSelector {
 
  Optional<ReplicaView> select(TopicPartition topicPartition, ClientMetadata metadata, PartitionView partitionView) {
    // if rackId is not null, iterate through the online replicas 
    // if one or more exists with matching rackId, choose the most caught-up replica from among them
    // otherwise return the current leader
  }
}
```

## 프로토콜 변경

선호 레플리카 선택을 가능하게 하기 위해 Fetch API를 확장한다. 위에서 언급한 컨슈머의 "client.rack" 설정을 사용해 브로커의 위치를 식별하므로, 컨슈머에도 유사한 설정을 추가한다.

```java
{
  "validVersions": "0-11",
  "fields": [
    {
      "name": "ReplicaId",
      "type": "int32",
      "versions": "0+",
      "about": "The broker ID of the follower, of -1 if this request is from a consumer."
    },
    {
      "name": "MaxWait",
      "type": "int32",
      "versions": "0+",
      "about": "The maximum time in milliseconds to wait for the response."
    },
    {
      "name": "MinBytes",
      "type": "int32",
      "versions": "0+",
      "about": "The minimum bytes to accumulate in the response."
    },
    {
      "name": "MaxBytes",
      "type": "int32",
      "versions": "3+",
      "default": "0x7fffffff",
      "ignorable": true,
      "about": "The maximum bytes to fetch. See KIP-74 for cases where this limit may not be honored."
    },
    {
      "name": "IsolationLevel",
      "type": "int8",
      "versions": "4+",
      "default": "0",
      "ignorable": false,
      "about": "This setting controls the visibility of transactional records. Using READ_UNCOMMITTED (isolation_level = 0) makes all records visible. With READ_COMMITTED (isolation_level = 1), non-transactional and COMMITTED transactional records are visible. To be more concrete, READ_COMMITTED returns all data from offsets smaller than the current LSO (last stable offset), and enables the inclusion of the list of aborted transactions in the result, which allows consumers to discard ABORTED transactional records"
    },
    {
      "name": "SessionId",
      "type": "int32",
      "versions": "7+",
      "default": "0",
      "ignorable": false,
      "about": "The fetch session ID."
    },
    {
      "name": "Epoch",
      "type": "int32",
      "versions": "7+",
      "default": "-1",
      "ignorable": false,
      "about": "The fetch session ID."
    },
    {
      "name": "Topics",
      "type": "[]FetchableTopic",
      "versions": "0+",
      "about": "The topics to fetch.",
      "fields": [
        {
          "name": "Name",
          "type": "string",
          "versions": "0+",
          "entityType": "topicName",
          "about": "The name of the topic to fetch."
        },
        {
          "name": "FetchPartitions",
          "type": "[]FetchPartition",
          "versions": "0+",
          "about": "The partitions to fetch.",
          "fields": [
            {
              "name": "PartitionIndex",
              "type": "int32",
              "versions": "0+",
              "about": "The partition index."
            },
            {
              "name": "CurrentLeaderEpoch",
              "type": "int32",
              "versions": "9+",
              "default": "-1",
              "ignorable": true,
              "about": "The current leader epoch of the partition."
            },
            {
              "name": "FetchOffset",
              "type": "int64",
              "versions": "0+",
              "about": "The message offset."
            },
            {
              "name": "LogStartOffset",
              "type": "int64",
              "versions": "5+",
              "default": "-1",
              "ignorable": false,
              "about": "The earliest available offset of the follower replica. The field is only used when the request is sent by the follower."
            },
            {
              "name": "MaxBytes",
              "type": "int32",
              "versions": "0+",
              "about": "The maximum bytes to fetch from this partition. See KIP-74 for cases where this limit may not be honored."
            }
          ]
        }
      ]
    },
    {
      "name": "Forgotten",
      "type": "[]ForgottenTopic",
      "versions": "7+",
      "ignorable": false,
      "about": "In an incremental fetch request, the partitions to remove.",
      "fields": [
        {
          "name": "Name",
          "type": "string",
          "versions": "7+",
          "entityType": "topicName",
          "about": "The partition name."
        },
        {
          "name": "ForgottenPartitionIndexes",
          "type": "[]int32",
          "versions": "7+",
          "about": "The partitions indexes to forget."
        }
      ]
    },
    {
      "name": "RackId",
      "type": "string",
      "versions": "11+",
      "default": "",
      "ignorable": true,
      "about": "Rack ID of the consumer making this request"
    }
  ]
}
```

fetch 응답은 선호 레플리카를 표시한다.

```java
{
  "apiKey": 1,
  "type": "response",
  "name": "FetchResponse",
  "validVersions": "0-11",
  "fields": [
    {
      "name": "ThrottleTimeMs",
      "type": "int32",
      "versions": "1+",
      "ignorable": true,
      "about": "The duration in milliseconds for which the request was throttled due to a quota violation, or zero if the request did not violate any quota."
    },
    {
      "name": "ErrorCode",
      "type": "int16",
      "versions": "7+",
      "ignorable": false,
      "about": "The top level response error code."
    },
    {
      "name": "SessionId",
      "type": "int32",
      "versions": "7+",
      "default": "0",
      "ignorable": false,
      "about": "The fetch session ID, or 0 if this is not part of a fetch session."
    },
    {
      "name": "Topics",
      "type": "[]FetchableTopicResponse",
      "versions": "0+",
      "about": "The response topics.",
      "fields": [
        {
          "name": "Name",
          "type": "string",
          "versions": "0+",
          "entityType": "topicName",
          "about": "The topic name."
        },
        {
          "name": "Partitions",
          "type": "[]FetchablePartitionResponse",
          "versions": "0+",
          "about": "The topic partitions.",
          "fields": [
            {
              "name": "PartitionIndex",
              "type": "int32",
              "versions": "0+",
              "about": "The partiiton index."
            },
            {
              "name": "ErrorCode",
              "type": "int16",
              "versions": "0+",
              "about": "The error code, or 0 if there was no fetch error."
            },
            {
              "name": "HighWatermark",
              "type": "int64",
              "versions": "0+",
              "about": "The current high water mark."
            },
            {
              "name": "LastStableOffset",
              "type": "int64",
              "versions": "4+",
              "default": "-1",
              "ignorable": true,
              "about": "The last stable offset (or LSO) of the partition. This is the last offset such that the state of all transactional records prior to this offset have been decided (ABORTED or COMMITTED)"
            },
            {
              "name": "LogStartOffset",
              "type": "int64",
              "versions": "5+",
              "default": "-1",
              "ignorable": true,
              "about": "The current log start offset."
            },
            {
              "name": "Aborted",
              "type": "[]AbortedTransaction",
              "versions": "4+",
              "nullableVersions": "4+",
              "ignorable": false,
              "about": "The aborted transactions.",
              "fields": [
                {
                  "name": "ProducerId",
                  "type": "int64",
                  "versions": "4+",
                  "entityType": "producerId",
                  "about": "The producer id associated with the aborted transaction."
                },
                {
                  "name": "FirstOffset",
                  "type": "int64",
                  "versions": "4+",
                  "about": "The first offset in the aborted transaction."
                }
              ]
            },
            {
              "name": "PreferredReadReplica",
              "type": "int32",
              "versions": "11+",
              "ignorable": true,
              "about": "The preferred read replica for the consumer to use on its next fetch request"
            },
            {
              "name": "Records",
              "type": "bytes",
              "versions": "0+",
              "nullableVersions": "0+",
              "about": "The record data."
            }
          ]
        }
      ]
    }
  ]
}
```

FetchRequest 스키마에는 replica id 필드가 있다. 컨슈머는 일반적으로 sentinel 값 -1을 사용하며, 이는 리더에서만 fetch가 허용된다는 뜻이다. 덜 알려진 sentinel 값으로 -2가 있는데, 이는 원래 디버깅 용도로 사용되도록 의도되었고 팔로워에서 fetch하는 것을 허용한다. 여기서는 컨슈머가 팔로워에서 fetch할 의도를 표시하기 위해 이를 사용할 수 있도록 제안한다. 마찬가지로 log start offset을 찾기 위해 팔로워에 ListOffsets 요청을 보내야 할 때도 replica id 필드에 같은 sentinel 값을 사용한다.

그러나 위에서 언급한 OFFSET_NOT_AVAILABLE 오류 코드 사용 때문에 Fetch 요청의 버전 증가는 여전히 필요하다. 요청 및 응답 스키마는 변경되지 않는다. 또한 모든 fetch 버전의 동작을 수정해 fetch offset이 out of range이면 현재 log start offset과 high watermark를 반환하도록 한다. 이를 통해 브로커는 out of range 오류를 구분할 수 있다.

KIP-320에서 놓친 한 가지 지점은 OffsetsForLeaderEpoch API가 log end offset을 노출한다는 사실이다. 컨슈머에게 log end offset은 high watermark여야 하므로, 이 프로토콜이 레플리카 요청과 컨슈머 요청을 구분할 방법이 필요하다. 여기서는 Fetch와 ListOffsets API에서 사용되는 동일한 `replica_id` 필드를 추가할 것을 제안한다. 새 스키마는 아래와 같다.

```java
OffsetsForLeaderEpochRequest => [Topic]
  ReplicaId => INT32              // New (-1 means consumer)
  Topic => TopicName [Partition]
    TopicName => STRING
    Partition => PartitionId CurrentLeaderEpoch LeaderEpoch
      PartitionId => INT32
      CurrentLeaderEpoch => INT32
      LeaderEpoch => INT32
```

컨슈머로부터 OffsetsForLeaderEpoch 요청이 수신되면, 반환되는 오프셋은 high watermark로 제한된다. Fetch API와 마찬가지로, 리더가 막 선출되었을 때 실제 high watermark는 짧은 시간 동안 알려지지 않는다. 이 시간 창 동안 최신 epoch를 포함한 OffsetsForLeaderEpoch 요청을 받으면, 리더는 OFFSET_NOT_AVAILABLE 오류 코드로 응답한다. 이로 인해 컨슈머는 retry한다. 또한 OffsetsForLeaderEpoch 질의는 리더만 처리할 수 있다는 점도 유의하라.

선호 읽기 레플리카가 있을 때 Java 클라이언트에 새 JMX metric이 추가된다. `"kafka.consumer:type=consumer-fetch-manager-metrics,client-id=*,topic=*,partition=*"` 객체 아래에 새 attribute `"preferred-read-replica"`가 추가된다. 이 metric은 컨슈머가 현재 fetch 중인 브로커 ID와 같은 값을 가진다. attribute가 없거나 -1로 설정되어 있으면, 컨슈머가 리더에서 fetch하고 있다는 뜻이다.

## 호환성, 지원 중단, 마이그레이션 계획

이 변경은 이전 버전과 하위 호환된다. 브로커가 팔로워 fetch를 지원하지 않으면, 컨슈머는 리더에서 fetch하는 기존 동작으로 되돌아간다.

## 거절된 대안

- **컨슈머 레플리카 선택**: 이 KIP의 첫 번째 iteration에서는 위에서 본 것과 유사한 인터페이스를 사용해 어떤 레플리카에서 fetch할지 컨슈머가 결정하도록 제안했다. 최종적으로는 replica selection 로직을 브로커가 제어할 수 있을 때 멀티 테넌트 환경에서 시스템을 추론하기 더 쉽다고 판단했다. 클라이언트 플러그인을 통해 공통 replica selection 로직을 갖는 것은 훨씬 더 어렵다. 많은 애플리케이션에 걸쳐 의존성과 구성을 조율해야 하기 때문이다. 같은 이유로, 선호 레플리카를 클라이언트가 사용할지 여부를 지정하는 옵션도 거절했다.
- **이전 Fetch 버전 처리**: 컨슈머가 이전 버전의 Fetch API를 사용할 수 있도록 하는 방안을 고려했다. 유일한 복잡함은 이전 버전에서는 더 구체적인 out of range 오류 코드를 사용할 수 없다는 점이다. 이로 인해 out of range 처리가 복잡해진다. out of range 오프셋이 리더에 존재할 것으로 기대해야 하는지 아닌지를 알 수 없기 때문이다. 팔로워에서 오프셋이 너무 작더라도 리더에서는 여전히 범위 안에 있을 수 있다. 한 가지 선택지는 컨슈머가 fetch 중인 레플리카에서 항상 offset reconciliation을 수행하는 것이다. 이 방식의 단점은 ISR 상태의 오래된 전파 때문에 컨슈머가 어떤 레플리카보다 더 이후의 오프셋을 본 상황을 감지할 수 없다는 것이다.
- **개선된 Log Start Offset 처리**: retention이 모든 레플리카에서 독립적으로 적용된다는 사실 때문에 log start offset의 일관성을 추론하기 어렵다는 점을 언급했다. 우리가 고려한 한 가지 선택지는 리더만 retention을 적용하도록 허용하고, 팔로워 삭제에는 Fetch 응답에서 전파되는 log start offset에만 의존하는 것이다. 원칙적으로 이는 팔로워의 log start offset이 항상 리더의 log start offset보다 작거나 같아야 한다는 보장을 제공한다. 그러나 log start offset acknowledgment를 ISR membership의 기준으로 삼지 않는 한, 실제로 이를 보장할 수 없다. 짧은 시간 안에 여러 번의 리더 선출이 발생하는 경우 이 보장이 깨질 수 있다. 그럼에도 팔로워 삭제를 위해 log start offset 전파에 의존하는 것은 여전히 좋은 아이디어일 수 있다고 생각하지만, 이는 별도로 검토할 것이다.
