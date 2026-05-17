---
title: Distributed Locks with Redis
aliases: [distributed-locks-with-redis]
category: source
tags: [redis, distributed-lock, redlock, mutual-exclusion]
sources: ["raw/articles/Distributed Locks with Redis.md"]
updated: 2026-05-17
---

# Distributed Locks with Redis

**출처**: https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/
**원본 파일**: `raw/articles/Distributed Locks with Redis.md`

## 핵심 요약

Redis를 이용한 분산 락 구현의 공식 레퍼런스. 단순 단일 인스턴스 방식의 비동기 복제 취약점을 설명하고, 이를 극복하는 **Redlock 알고리즘**을 제안한다. 분산 락의 세 가지 보장 속성(상호 배제·교착 없음·내결함성)을 정의하고, N개의 독립 Redis Master에 과반수 락을 획득하는 방식으로 단일 장애점을 제거한다. 마지막으로 클럭 드리프트 문제와 펜싱 토큰의 필요성 등 일관성 한계를 솔직하게 언급한다.

## 주요 개념

- **단일 인스턴스 락**: `SET NX PX` + `DELEX`(또는 Lua 스크립트)로 원자적 획득·해제
- **Redlock 알고리즘**: N=5 독립 Master, 과반수 획득, 경과 시간 검증
- **지연 재시작**: 영속성 없이도 안전성 확보 — 크래시 후 TTL 이상 대기
- **펜싱 토큰**: 만료된 락 보유자의 "늦은 쓰기" 차단을 위한 단조 증가 토큰
- **클럭 드리프트 한계**: Redis TTL은 단조시계 미사용 → 시각 변동 시 안전성 위협

## 생성된 위키 페이지

- [[wiki/distributed/redis-distributed-lock|Redis-분산-락]] — Redlock 알고리즘 전체 설명
