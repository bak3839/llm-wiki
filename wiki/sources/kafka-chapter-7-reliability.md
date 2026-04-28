---
title: "[Chapter 7] 신뢰성 있는 데이터 전달"
category: source
tags: [kafka, reliability, replication-factor, unclean-leader-election, min-insync-replicas, ISR, acks, page-cache]
sources: ["raw/articles/[Chapter 7] 신뢰성 있는 데이터 전달.md"]
updated: 2026-04-25
---

# [Chapter 7] 신뢰성 있는 데이터 전달

## 출처

`raw/articles/[Chapter 7] 신뢰성 있는 데이터 전달.md`

## 핵심 요약

카프카의 신뢰성 보장과 트레이드오프를 3개 영역으로 설명한다. 기본 보장은 파티션 내 순서, ISR 복제 완료 시 커밋, 컨슈머의 커밋 메시지만 읽기다. acks 설정으로 프로듀서 내구성 수준을 조절한다. 복제 팩터는 가용성·지속성과 처리량·비용·지연 사이 트레이드오프이며 broker.rack으로 AZ 분산을 보장한다. 언클린 리더 선출은 가용성 vs 데이터 유실·일관성 트레이드오프다. min.insync.replicas는 커밋 허용 최소 ISR 수를 강제해 acks=all과 함께 강한 내구성을 보장한다. 카프카는 디스크 fsync 대신 복제와 OS 페이지 캐시로 내구성을 확보한다.

## 주요 개념 목록

- [[wiki/kafka/reliability|신뢰성]] — 신뢰성 보장, acks, 복제 팩터 트레이드오프, 언클린 리더 선출, min.insync.replicas, 페이지 캐시
- [[wiki/kafka/replication|복제]] — ISR 조건 상세 (ZooKeeper 하트비트, 랙 조건), 느린 ISR의 영향
