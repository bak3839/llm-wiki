---
title: 컨슈머 사망 → 리밸런스 → 재기동 흐름
aliases: [컨슈머-장애-복구]
category: kafka
tags: [kafka, consumer, rebalance, recovery, session-timeout, static-membership, offset]
sources: []
updated: 2026-05-02
---

# 컨슈머 사망 → 리밸런스 → 재기동 흐름

## 1단계: 사망 감지

컨슈머가 크래시하면 하트비트 전송이 중단된다.

```
컨슈머 C3 크래시 → 하트비트 전송 중단
→ session.timeout.ms 경과 (기본 45초)
→ 그룹 코디네이터가 C3 사망으로 판정
→ 리밸런스 트리거
```

## 2단계: 리밸런스 (협력적, Kafka 3.1+ 기본)

```
[Round 1]
코디네이터 → 전체 컨슈머에 재조인 통보
각 컨슈머 → onPartitionRevoked() 호출
  → 재할당 대상 파티션 소유권 포기
  → 오프셋 커밋 (여기서 커밋하지 않으면 중복 위험)

[Round 2]
그룹 리더 → C3 파티션을 나머지 컨슈머에 재배분
코디네이터 → SyncGroup으로 새 할당 전파
각 컨슈머 → onPartitionAssigned() 호출 → 처리 재개
```

협력적 리밸런스이므로 재할당 대상이 아닌 파티션의 컨슈머는 처리를 이어간다.

## 3단계: C3 재기동

```
C3 시작 → 그룹 코디네이터로 JoinGroup 요청
→ 리밸런스 재트리거 (그룹 구성 변경)
→ C3에 파티션 재할당
→ __consumer_offsets에서 마지막 커밋 오프셋 조회
→ 해당 오프셋 이후부터 읽기 재개
```

C3가 합류하면 **두 번째 리밸런스**가 발생한다. 전체 그룹의 파티션이 다시 배분된다.

## 오프셋 처리에 따른 재기동 동작 차이

| 커밋 방식 | 결과 |
|----------|------|
| 자동 커밋 | 마지막 커밋 주기 이후 메시지는 중복 처리 가능 |
| `onPartitionRevoked()`에서 수동 커밋 | 정확한 위치부터 재개 — 중복 최소화 |
| 커밋 없이 사망 | 다른 컨슈머가 처리·커밋했으면 그 이후부터, 아니면 이전 커밋 오프셋부터 |

## 정적 멤버십 사용 시 (`group.instance.id`)

```
C3 사망 → session.timeout.ms 이내 재기동
→ 리밸런스 없이 이전 파티션 그대로 재할당
```

코디네이터가 파티션 할당을 캐시하기 때문에, 타임아웃 이내 재기동 시 리밸런스 없이 이전 파티션을 그대로 돌려받는다. 단, 다운 기간 동안 해당 파티션은 처리되지 않으므로 재기동 후 밀린 메시지를 따라잡아야 한다.

## 관련 항목

- [[wiki/kafka/rebalance|리밸런스]] — 리밸런스 유형(조급한/협력적), JoinGroup 절차, ConsumerRebalanceListener
- [[wiki/kafka/consumer-group|컨슈머-그룹]] — 정적 그룹 멤버십, 파티션-컨슈머 할당 규칙
- [[wiki/kafka/offset-commit|오프셋-커밋]] — onPartitionRevoked()에서의 커밋 전략
- [[wiki/kafka/consumer-config|컨슈머-설정]] — session.timeout.ms, heartbeat.interval.ms
