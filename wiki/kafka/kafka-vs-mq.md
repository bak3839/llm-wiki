---
title: Kafka vs 전통적 메시지 큐
aliases: [kafka-vs-mq]
category: kafka
tags: [kafka, message-queue, RabbitMQ, comparison, pull-model, log, replication]
sources: []
updated: 2026-04-30
---

# Kafka vs 전통적 메시지 큐

## 핵심 설계 철학 차이

전통적 MQ(RabbitMQ, ActiveMQ 등)는 **메시지 브로커** 모델 — 메시지를 전달하고 소비되면 삭제한다. Kafka는 **분산 로그(Distributed Log)** 모델 — 메시지를 파티션에 순서대로 저장하고 보존 기간 동안 유지한다.

## 비교표

| 항목 | 전통적 MQ (RabbitMQ 등) | Kafka |
|------|------------------------|-------|
| **메시지 소비 방식** | 브로커가 컨슈머에게 Push | 컨슈머가 브로커에서 Pull (`poll()`) |
| **소비 후 메시지** | 즉시 삭제 | 보존 기간 동안 유지 (replay 가능) |
| **소비 위치 관리** | 브로커가 관리 | 컨슈머가 오프셋으로 직접 관리 |
| **멀티 컨슈머** | 경쟁 소비 (하나만 받음) | 컨슈머 그룹별로 독립적으로 전체 수신 |
| **메시지 순서** | 큐 단위 순서 보장 | 파티션 단위 순서 보장 |
| **확장성** | 수직 확장 중심 | 파티션 추가로 수평 확장 |
| **처리량** | 낮은 지연 최적화 | 높은 처리량 최적화 (zero-copy, batch) |
| **신뢰성** | 주로 단일 큐 지속성 | ISR 복제 + min.insync.replicas |
| **메시지 재처리** | 어려움 (DLQ 등 필요) | 오프셋 리셋으로 재처리 가능 |

## Kafka 설계 특성 상세

### Pull 기반 소비

컨슈머가 자신의 속도에 맞게 `poll()`을 호출한다. 브로커가 컨슈머 상태를 직접 관리하지 않아 backpressure가 자연스럽게 발생한다. 단, 메시지 도착 즉시 처리되지 않고 poll 주기에 의존한다. → [[wiki/kafka/poll-loop|폴링-루프]]

### 독립적 멀티 컨슈머

같은 토픽을 여러 컨슈머 그룹이 각자 독립적으로 처음부터 읽을 수 있다. 전통적 MQ의 경쟁 소비(한 컨슈머만 특정 메시지 수신)와 근본적으로 다르다. → [[wiki/kafka/consumer-group|컨슈머-그룹]]

### 오프셋 기반 재처리

컨슈머가 오프셋을 되돌려 과거 메시지를 다시 처리할 수 있다. 버그 수정 후 데이터 재처리, 새 컨슈머 그룹의 전체 히스토리 소비 등이 가능하다. → [[wiki/kafka/offset-commit|오프셋-커밋]], [[wiki/kafka/auto-offset-reset|auto-offset-reset]]

### 높은 처리량

zero-copy, OS 페이지 캐시, 배치 처리로 전통적 MQ 대비 훨씬 높은 처리량을 제공한다. 낮은 지연이 요구되는 작업엔 오버스펙일 수 있다. → [[wiki/kafka/request-handling|요청-처리]]

### 복제 기반 내구성

ISR + `acks=all` + `min.insync.replicas`로 강한 내구성을 보장한다. 디스크 fsync 대신 다중 복제본으로 내구성을 확보한다. → [[wiki/kafka/reliability|신뢰성]], [[wiki/kafka/replication|복제]]

## 선택 기준

| 상황 | 선택 |
|------|------|
| 대용량 이벤트 스트리밍, 로그 수집 | **Kafka** |
| 메시지 재처리·replay 필요 | **Kafka** |
| 여러 시스템이 동일 이벤트 독립 소비 | **Kafka** |
| 단순 작업 큐, 낮은 지연 응답 필요 | **전통적 MQ** |
| 복잡한 라우팅·필터링 필요 | **전통적 MQ** |

## 관련 항목

- [[wiki/kafka/consumer-group|컨슈머-그룹]] — 컨슈머 그룹의 독립 소비 모델
- [[wiki/kafka/poll-loop|폴링-루프]] — Kafka Pull 모델 상세
- [[wiki/kafka/replication|복제]] — ISR, high watermark
- [[wiki/kafka/reliability|신뢰성]] — acks, min.insync.replicas, 복제 팩터
