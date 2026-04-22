---
title: 리밸런스
aliases: [리밸런스]
category: kafka
tags: [kafka, rebalance, consumer-group, partition, cooperative, eager]
sources: [raw/articles/kafka_chapter_4.md]
updated: 2026-04-21
---

# 리밸런스

## 개념

컨슈머에 할당된 파티션을 다른 컨슈머에게 재할당하는 작업이다. 컨슈머 그룹에 **높은 가용성**과 **규모 가변성**을 제공한다.

### 리밸런스 트리거 조건

- 새 컨슈머가 그룹에 참여
- 기존 컨슈머가 종료되거나 크래시
- 컨슈머가 하트비트를 `session.timeout.ms` 동안 보내지 않음
- 구독 중인 토픽에 파티션이 추가됨
- 정규식 구독 시 새 토픽이 패턴에 매칭됨

## 그룹 코디네이터

- 각 컨슈머 그룹마다 하나의 브로커가 **그룹 코디네이터** 역할을 담당
- 핵심 기능
  - 컨슈머 그룹 멤버십 관리 (추가/제거/사망 감지)
  - 리밸런스 트리거
  - 파티션 할당 절차 조율 (계산은 그룹 리더가, 전파는 코디네이터가 담당)
  - 오프셋 커밋 관리 (`__consumer_offsets` 토픽에 기록)

## 파티션 할당 절차

1. 컨슈머가 그룹 코디네이터에 `JoinGroup` 요청 전송
2. 가장 먼저 참여한 컨슈머가 **그룹 리더**가 됨
3. 리더가 살아있는 컨슈머 목록을 받아 `PartitionAssignor`로 할당 계산
4. 리더가 할당 내역을 코디네이터에 전달
5. 코디네이터가 모든 컨슈머에 할당 내역 전파

## 리밸런스 유형 비교

### 조급한 리밸런스 (Eager Rebalance)

- 모든 컨슈머가 읽기를 **중단**하고 파티션 소유권을 **전부 포기**
- 그룹 전체가 재조인 후 새 파티션을 할당받음
- **Stop the World** 현상 발생
- Kafka 3.1부터 기본값에서 제외됨 (추후 삭제 예정)

### 협력적 리밸런스 (Cooperative Rebalance / Incremental Rebalance)

- 재할당 대상 파티션만 선별적으로 이동
- 재할당되지 않은 파티션의 컨슈머는 계속 처리를 이어감
- 최소 2단계로 진행:
  1. 리더가 재할당 대상 파티션을 각 컨슈머에 통보 → 해당 파티션 소유권 포기
  2. 리더가 포기된 파티션을 재할당
- Stop the World 없이 점진적으로 안정화
- **Kafka 3.1부터 기본값**

## ConsumerRebalanceListener

`subscribe()` 호출 시 `ConsumerRebalanceListener`를 전달해 리밸런스 이벤트를 처리할 수 있다.

| 메서드 | 호출 시점 |
|--------|-----------|
| `onPartitionAssigned()` | 파티션이 재할당된 후, 읽기 시작 전 |
| `onPartitionRevoked()` | 파티션이 해제될 때 (조급한: 리밸런스 전, 협력적: 리밸런스 완료 후) |
| `onPartitionLost()` | 협력적 리밸런스에서 해제 전에 다른 컨슈머에 먼저 할당된 예외 상황 |

`onPartitionRevoked()`에서는 반드시 오프셋을 커밋해야 다음 컨슈머가 이어서 읽을 수 있다.

## 관련 항목

- [[컨슈머-그룹]] — 컨슈머 그룹 개념 및 파티션 할당 규칙
- [[컨슈머-설정]] — session.timeout.ms, heartbeat.interval.ms, partition.assignment.strategy
- [[오프셋-커밋]] — 리밸런스 전 오프셋 커밋 전략
