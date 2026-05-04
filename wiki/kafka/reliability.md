---
title: 신뢰성 있는 데이터 전달
aliases: [신뢰성]
category: kafka
tags: [kafka, reliability, replication-factor, unclean-leader-election, min-insync-replicas, acks, page-cache, flush, monitoring, consumer-lag]
sources: ["raw/articles/[Chapter 7] 신뢰성 있는 데이터 전달.md", "raw/articles/Chapter 7 신뢰성 있는 데이터 전달2.md"]
updated: 2026-05-05
---

# 신뢰성 있는 데이터 전달

## 카프카의 신뢰성 보장

카프카가 항상 지키는 기본 보장:

1. **파티션 내 메시지 순서 보장** — 나중에 쓰여진 메시지가 먼저 쓰여진 메시지보다 오프셋이 큼
2. **커밋 = 모든 ISR 복제 완료** — 모든 인-싱크 레플리카에 쓰여진 시점에 커밋. 디스크 flush까지는 불필요
3. **커밋된 메시지는 유실되지 않음** — 최소 1개의 작동 가능한 레플리카가 남아있는 한
4. **컨슈머는 커밋된 메시지만 읽음** — high watermark 이후는 노출 안 됨

## acks 설정과 트레이드오프

프로듀서의 `acks` 설정으로 신뢰성 수준을 선택한다:

| acks | 응답 시점 | 의미 |
|------|----------|------|
| `0` | 네트워크 전송 직후 | 유실 가능성 높음, 최고 처리량 |
| `1` | 리더 쓰기 완료 후 | 리더 장애 시 유실 가능 |
| `all` (-1) | 모든 ISR 쓰기 완료 후 | 가장 강한 내구성 보장 |

## 복제 팩터 트레이드오프

`replication.factor` (토픽 단위) / `default.replication.factor` (브로커 단위)

복제 팩터 N → N-1개 브로커 중단에도 읽기·쓰기 가능

| 관점 | 설명 |
|------|------|
| **가용성** | 레플리카가 많을수록 더 많은 브로커 장애를 견딤 |
| **지속성** | 복사본이 많을수록 전체 데이터 유실 가능성 감소 |
| **처리량** | 레플리카 N개 → 복제 트래픽 (N-1)배 증가 |
| **종단 지연** | ISR 중 하나가 느려지면 컨슈머 전체가 함께 느려짐 |
| **비용** | 레플리카 수에 비례하여 저장소·네트워크 비용 증가 |

**랙/AZ 분산**: `broker.rack` 설정으로 레플리카를 서로 다른 랙(AZ)에 분산 → 랙 단위 장애에서 가용성 보장.

## 언클린 리더 선출

`unclean.leader.election.enable` (기본값: `false`)

리더가 사라졌을 때 ISR이 하나도 없는 경우 아웃-오브-싱크 레플리카를 리더로 허용할지 결정한다.

| 설정 | 효과 | 리스크 |
|------|------|--------|
| `false` (기본) | 원래 리더 복구 전까지 파티션 오프라인 | 가용성 저하 |
| `true` | 아웃-오브-싱크 레플리카가 즉시 리더 취임 | 데이터 유실 + 오프셋 불일치로 일관성 깨짐 |

**일관성 깨짐 예시**: 레플리카 2가 오프셋 100~200을 가진 채 리더였다가 죽고, 오프셋 0~100만 가진 레플리카 0이 새 리더가 되면 — 기존 컨슈머가 읽은 100~200은 완전히 다른 데이터로 교체됨.

## 최소 인-싱크 레플리카

`min.insync.replicas` (브로커·토픽 단위)

커밋을 허용하기 위한 **최소 ISR 수**. ISR이 이 값 미만이면 프로듀서 쓰기를 거부한다.

**예시**: 레플리카 3개 + `min.insync.replicas=2`
- ISR 3개: 정상
- ISR 2개: 정상
- ISR 1개: 쓰기 거부 (`NotEnoughReplicasException`), 읽기만 가능 → **사실상 읽기 전용**
- ISR 0개: 리더 없음

> `acks=all` + `min.insync.replicas=2`로 설정하면 언클린 리더 선출 없이도 데이터 유실을 방지하면서 언클린 리더 선출 위험을 줄일 수 있다.

## 디스크 저장과 페이지 캐시

카프카는 **매 메시지마다 fsync를 하지 않는다**. 대신 OS 페이지 캐시에 의존하고 복제로 내구성을 보장한다.

- 세그먼트 교체 시 (기본 1GB) 및 재시작 직전에만 flush
- 서로 다른 랙/AZ에 있는 3개 레플리카가 단일 디스크 fsync보다 더 안전하다는 판단

**fsync 강제 설정** (권장하지 않음):
- `flush.messages`: 미flush 최대 메시지 수
- `flush.ms`: flush 주기

> 소규모 batch에서 매 메시지 fsync는 기본 설정 대비 처리량을 약 3~5배 낮춘다.

## 시스템 신뢰성 검증

### 설정 검증

`org.apache.kafka.tools` 패키지의 두 CLI 도구를 활용한다:

| 도구 | 역할 |
|------|------|
| `VerifiableProducer` | 순번 메시지를 전송하며 성공/에러 출력 |
| `VerifiableConsumer` | 수신 순서, 커밋, 리밸런스 정보 출력 |

권장 테스트 시나리오: 리더 선출, 컨트롤러 선출, 롤링 재시작, 언클린 리더 선출.

### 프로덕션 모니터링

**프로듀서 핵심 지표 (JMX)**
- 레코드별 에러율 (`error-rate`)
- 재시도율 (`record-retry-rate`)
- 재시도 횟수 소진 이벤트 → `delivery.timeout.ms`, `retries` 설정 확인 필요

**컨슈머 핵심 지표: consumer lag**
- 컨슈머가 파티션의 최신 커밋 메시지에서 얼마나 뒤떨어져 있는지
- 이상적으로는 0, 현실에서는 poll 주기로 인해 오르락내리락
- 중요한 것은 컨슈머가 계속 따라붙는지 여부
- **Burrow** (LinkedIn 개발 오픈소스): consumer lag 추세 분석에 유용

**브로커 에러 응답 JMX 지표**
- `kafka.server:type=BrokerTopicMetrics,name=FailedProduceRequestsPerSec`
- `kafka.server:type=BrokerTopicMetrics,name=FailedFetchRequestsPerSec`

## 관련 항목

- [[wiki/kafka/producer-reliability|프로듀서-신뢰성]] — acks 상세, 재시도, 멱등성, 에러 핸들링
- [[wiki/kafka/replication|복제]] — ISR, high watermark, 선호 리더, out-of-sync 판정 상세
- [[wiki/kafka/request-handling|요청-처리]] — acks 처리, Purgatory, 쓰기 요청 흐름
- [[wiki/kafka/controller|컨트롤러]] — 파티션 리더 선출 메커니즘
- [[wiki/kafka/offset-commit|오프셋-커밋]] — 컨슈머 측 신뢰성 실천 사항
