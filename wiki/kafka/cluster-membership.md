---
title: 클러스터 멤버십
aliases: [클러스터-멤버십]
category: kafka
tags: [kafka, cluster, broker, zookeeper, ephemeral-node, broker-id]
sources: ["raw/articles/[Chapter 6] 카프카 내부 메커니즘 34a055f5905980f38115c3cb54c9dd73.md"]
updated: 2026-04-24
---

# 클러스터 멤버십

## 개념

카프카는 현재 클러스터 멤버인 브로커 목록을 관리하기 위해 **아파치 주키퍼**를 사용한다 (KRaft 이전). 각 브로커는 고유한 브로커 ID를 가지며, 시작 시 ZooKeeper에 Ephemeral 노드로 등록된다.

## 브로커 등록

- 시작 시 `/brokers/ids/<broker-id>` 경로에 **Ephemeral 노드** 생성
- Ephemeral 노드: 생성한 세션이 살아있는 동안만 존재하는 ZNode. 세션 종료 시 ZooKeeper가 자동 삭제
- 동일 ID를 가진 다른 브로커가 이미 등록되어 있으면 에러 발생 (충돌 방지)

## 브로커 이탈 감지

브로커와 ZooKeeper 연결이 끊기면 (정지, 네트워크 단절, 긴 GC 등):

1. Ephemeral 노드 자동 삭제
2. `/brokers/ids` 경로에 **watch**를 설정한 컴포넌트(컨트롤러 포함)에 알림 전파
3. 컨트롤러가 해당 브로커의 이탈을 인식하고 파티션 리더 재선출 수행

## 브로커 ID 재사용

브로커가 종료되어도 브로커 ID는 **토픽 레플리카 목록 등 다른 자료구조에 남아있다**. 동일 ID로 새 브로커를 투입하면:
- 클러스터에서 자동으로 기존 브로커의 역할 이어받음
- 이전 브로커가 담당하던 토픽과 파티션들을 재할당받음

## 관련 항목

- [[wiki/kafka/controller|컨트롤러]] — 클러스터 멤버십 변화를 감지해 파티션 리더 선출 수행
- [[wiki/kafka/replication|복제]] — 레플리카 목록에서 broker ID가 사용되는 방식
