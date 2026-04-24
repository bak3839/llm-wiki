---
title: "[Chapter 6] 카프카 내부 메커니즘"
category: source
tags: [kafka, internals, controller, replication, ISR, KRaft, request-handling, ZooKeeper, zero-copy]
sources: ["raw/articles/[Chapter 6] 카프카 내부 메커니즘 34a055f5905980f38115c3cb54c9dd73.md"]
updated: 2026-04-24
---

# [Chapter 6] 카프카 내부 메커니즘

## 출처

`raw/articles/[Chapter 6] 카프카 내부 메커니즘 34a055f5905980f38115c3cb54c9dd73.md`

## 핵심 요약

카프카 내부 작동 방식을 4개 영역으로 설명한다. 클러스터 멤버십은 ZooKeeper ephemeral 노드로 관리된다. 컨트롤러는 가장 먼저 ZooKeeper에 노드를 등록한 브로커가 담당하며 epoch 번호로 좀비 컨트롤러를 방지한다. KRaft는 ZooKeeper를 래프트 기반 메타데이터 로그로 대체하여 운영 복잡성을 해소한다. 복제는 ISR 메커니즘으로 신뢰성을 보장하며, 요청 처리는 acceptor/processor/IO 스레드 구조와 zero-copy 최적화로 성능을 극대화한다.

## 주요 개념 목록

- [[wiki/kafka/cluster-membership|클러스터-멤버십]] — ZooKeeper ephemeral 노드를 통한 브로커 등록·이탈 감지, broker ID 재사용
- [[wiki/kafka/controller|컨트롤러]] — 파티션 리더 선출, epoch, 좀비 컨트롤러, KRaft (Kafka 3.3+)
- [[wiki/kafka/replication|복제]] — 리더/팔로워 레플리카, ISR, high watermark, 선호 리더, replica.lag.time.max.ms
- [[wiki/kafka/request-handling|요청-처리]] — acceptor/processor/IO 스레드, acks, Purgatory, zero-copy, Fetch Session Cache
