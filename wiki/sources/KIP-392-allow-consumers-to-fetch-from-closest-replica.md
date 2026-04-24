---
title: "[KIP-392] 컨슈머가 가장 가까운 레플리카에서 fetch할 수 있도록 허용"
category: source
tags: [kafka, consumer, follower-fetch, replica, rack-awareness, KIP-392]
sources: ["raw/articles/KIP-392 Allow consumers to fetch from closest replica - 한국어 번역.md"]
updated: 2026-04-24
---

# [KIP-392] 컨슈머가 가장 가까운 레플리카에서 fetch할 수 있도록 허용

## 출처

`raw/articles/KIP-392 Allow consumers to fetch from closest replica - 한국어 번역.md`

원문: https://cwiki.apache.org/confluence/display/KAFKA/KIP-392%3A+Allow+consumers+to+fetch+from+closest+replica

**상태**: Accepted (KAFKA-8443)

## 핵심 요약

멀티 AZ/데이터센터 환경에서 컨슈머가 항상 파티션 리더에서만 fetch해야 했던 제약을 해소하는 제안이다. 브로커가 `ReplicaSelector` 플러그인으로 컨슈머에게 선호 레플리카를 안내하고, 컨슈머는 이후 Fetch 요청을 해당 레플리카로 전환한다. 팔로워 fetch 시 HW 전파 지연과 out-of-range 오류(4가지 케이스)를 다루며, `OFFSET_NOT_AVAILABLE` 오류 코드로 일관된 처리를 제공한다.

## 주요 개념 목록

- [[wiki/kafka/follower-fetch|팔로워-페치]] — KIP-392 핵심: 가장 가까운 레플리카에서 fetch, out-of-range 4가지 케이스, ReplicaSelector
- [[wiki/kafka/consumer-config|컨슈머-설정]] — client.rack, replica.selector.class
