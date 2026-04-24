---
title: 복제
aliases: [복제]
category: kafka
tags: [kafka, replication, ISR, leader-replica, follower-replica, high-watermark, preferred-leader, replica.lag.time.max.ms]
sources: ["raw/articles/[Chapter 6] 카프카 내부 메커니즘 34a055f5905980f38115c3cb54c9dd73.md"]
updated: 2026-04-24
---

# 복제

## 개념

복제는 카프카 아키텍처의 핵심으로, 개별 노드 장애에서도 신뢰성과 지속성을 보장하는 메커니즘이다. 각 파티션은 다수의 레플리카를 가지며, 각 레플리카는 서로 다른 브로커에 저장된다.

## 레플리카 종류

### 리더 레플리카

- 각 파티션에 하나씩 존재
- 모든 **쓰기 요청**은 리더로만 전달됨
- 클라이언트는 리더 또는 팔로워(KIP-392, `client.rack` 설정 시)에서 읽기 가능

### 팔로워 레플리카

- 리더를 제외한 나머지 레플리카
- 기본적으로 클라이언트 요청 처리 불가
- 리더에 주기적으로 **읽기 요청(Fetch)** 을 보내 최신 메시지를 복제

## ISR (In-Sync Replica)

지속적으로 리더의 최신 메시지를 복제하고 있는 레플리카. 리더 장애 시 **ISR에 속한 레플리카만 새 리더로 선출될 수 있음**.

메시지가 **모든 ISR에 복제된 시점**이 committed 상태 → 리더의 high watermark 전진.

### 아웃-오브-싱크 (Out-of-Sync) 판정

팔로워가 `replica.lag.time.max.ms` 시간 동안 다음 중 하나에 해당하면 ISR에서 제외:
- 읽기 요청을 전혀 보내지 않음
- 요청을 보냈지만 가장 최근 메시지를 따라잡지 못함

아웃-오브-싱크 레플리카는 리더 선출 자격을 잃는다.

## High Watermark

리더가 컨슈머에게 노출하는 오프셋 경계선. **모든 ISR에 복제된 마지막 메시지**까지만 컨슈머에게 반환한다.

- HW 이전 메시지만 노출하는 이유: 리더 장애 시 HW 이후 메시지는 복제되지 않아 사라질 수 있음 → 데이터 일관성 보장
- 리더는 팔로워에 보내는 Fetch 응답에 현재 HW를 포함해 전파
- 팔로워에서 읽기 시 HW 전파 지연으로 최신 메시지 노출이 리더보다 늦을 수 있음 → [[wiki/kafka/follower-fetch|팔로워-페치]] 참고

## 복제 메커니즘

팔로워는 컨슈머의 fetch 요청과 **동일한 프로토콜**로 리더에 읽기 요청을 보내 메시지를 복제한다. 리더는 팔로워가 마지막으로 요청한 오프셋으로 각 팔로워의 복제 진행도를 파악할 수 있다.

## 선호 리더 (Preferred Leader)

토픽이 처음 생성될 때 리더였던 레플리카. 파티션 **레플리카 목록의 첫 번째 레플리카**가 선호 리더다.

- 모든 파티션의 선호 리더가 실제 리더가 되면 부하가 브로커 간에 균등 분배됨
- `auto.leader.rebalance.enable=true` (기본값): 선호 리더가 ISR에 있을 경우 자동으로 리더 복구

```bash
# 레플리카 목록 확인 — 첫 번째가 선호 리더
kafka-topics.sh --describe --topic <토픽명>
```

> 수동으로 레플리카를 재할당할 때는 선호 레플리카를 서로 다른 브로커로 분산해 부하가 한쪽에 몰리지 않도록 한다.

## 주요 설정

| 설정 | 위치 | 설명 |
|------|------|------|
| `replica.lag.time.max.ms` | 브로커 | 팔로워가 out-of-sync로 판정되는 최대 지연 시간 |
| `auto.leader.rebalance.enable` | 브로커 | 선호 리더 자동 복구 여부 (기본값: `true`) |

## 관련 항목

- [[wiki/kafka/controller|컨트롤러]] — 파티션 리더 선출 수행
- [[wiki/kafka/follower-fetch|팔로워-페치]] — 팔로워 레플리카에서 컨슈머가 직접 읽기 (KIP-392)
- [[wiki/kafka/request-handling|요청-처리]] — high watermark가 읽기 요청 처리에 적용되는 방식
- [[wiki/kafka/cluster-membership|클러스터-멤버십]] — 브로커 이탈 감지와 레플리카 재할당
