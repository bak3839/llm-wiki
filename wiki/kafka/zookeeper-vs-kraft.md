---
title: ZooKeeper vs KRaft 비교
aliases: [주키퍼-KRaft-비교]
category: kafka
tags: [kafka, zookeeper, KRaft, raft, controller, metadata, comparison]
sources: ["raw/articles/[Chapter 6] 카프카 내부 메커니즘 34a055f5905980f38115c3cb54c9dd73.md"]
updated: 2026-04-25
---

# ZooKeeper vs KRaft 비교

## 핵심 차이 요약

| 항목 | ZooKeeper 방식 | KRaft 방식 |
|------|---------------|------------|
| **메타데이터 저장소** | 외부 ZooKeeper 클러스터 | 내부 메타데이터 이벤트 로그 (컨트롤러 쿼럼) |
| **컨트롤러 선출** | ZooKeeper ephemeral 노드 경쟁 | 래프트(Raft) 알고리즘으로 자체 선출 |
| **메타데이터 변경 전파** | 컨트롤러 → 브로커 (push) | 브로커 → 액티브 컨트롤러 (pull, MetadataFetch API) |
| **브로커 등록** | 시작/종료 시 ephemeral 노드 자동 생성·삭제 | 컨트롤러 쿼럼에 등록, 운영자가 명시적으로 해제 전까지 유지 |
| **컨트롤러 장애 복구** | 전체 메타데이터 ZooKeeper에서 리로드 필요 (파티션 수 증가 시 병목) | 팔로워 컨트롤러가 최신 상태 유지 → 즉시 투입 가능 |
| **운영 복잡성** | 카프카 + ZooKeeper 두 분산 시스템 관리 필요 | 단일 시스템 (ZooKeeper 불필요) |
| **좀비 방지 메커니즘** | 에포크(epoch) — 낮은 에포크 메시지 무시 | Fenced state — 메타데이터 비최신 브로커의 요청 처리 차단 |

## ZooKeeper 방식의 문제점

1. **메타데이터 불일치**: ZooKeeper 쓰기(동기) ↔ 브로커 메시지 전송(비동기) 불일치
2. **확장성 병목**: 컨트롤러 재시작 시 ZooKeeper 전체 메타데이터 로드 후 전 브로커에 전송 — 파티션/브로커 수에 비례하는 지연
3. **소유권 분산**: 메타데이터가 컨트롤러 · 브로커 · ZooKeeper에 분산되어 일관성 보장 어려움
4. **운영 오버헤드**: 두 분산 시스템 별도 운영·모니터링 필요

## KRaft 핵심 아이디어

컨트롤러 노드들이 **메타데이터 이벤트 로그를 관리하는 래프트 쿼럼**이 됨:

- 토픽, 파티션, ISR, 설정 등 모든 메타데이터를 이벤트 로그로 저장
- 래프트 알고리즘으로 **액티브 컨트롤러** 자체 선출 (외부 시스템 의존 없음)
- 팔로워 컨트롤러들이 데이터 복제 → 장애 시 리로드 없이 즉시 투입

## Fenced State

브로커가 온라인이지만 메타데이터가 최신이 아닌 경우 **fenced 상태**로 전환된다. 클라이언트 요청 처리 불가. 자신이 리더가 아님에도 리더라 착각하는 브로커의 잘못된 쓰기를 방지한다. ZooKeeper 방식의 에포크 개념과 같은 역할이지만, 더 넓은 범위(브로커 전체)에 적용된다.

> KRaft는 Kafka 2.8 프리뷰, **3.3부터 프로덕션 사용 가능**

## 관련 항목

- [[wiki/kafka/controller|컨트롤러]] — 컨트롤러 선출, 에포크, 좀비 컨트롤러, KRaft 상세
- [[wiki/kafka/cluster-membership|클러스터-멤버십]] — ZooKeeper ephemeral 노드를 통한 브로커 등록·이탈 감지
- [[wiki/kafka/replication|복제]] — ISR, 리더/팔로워 레플리카
