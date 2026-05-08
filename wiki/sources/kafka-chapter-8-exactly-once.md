---
title: "[Chapter 8] 정확히 한 번 의미 구조"
category: source
tags: [kafka, idempotent-producer, transactions, exactly-once, eos]
sources: ["raw/articles/Chapter 8 정확히 한 번 의미 구조.md"]
updated: 2026-05-08
---

# [Chapter 8] 정확히 한 번 의미 구조

## 출처

`raw/articles/Chapter 8 정확히 한 번 의미 구조.md`

## 핵심 요약

카프카에서 메시지 중복·유실 없이 정확히 한 번 처리를 보장하는 두 가지 메커니즘을 다룬다.

**멱등적 프로듀서(8.1)**는 프로듀서 내부 재시도로 발생하는 중복을 자동 방지한다. `enable.idempotence=true`를 설정하면 각 메시지 배치에 프로듀서ID·시퀀스 넘버가 추가되고, 브로커가 중복을 감지해 거부한다. 단, 애플리케이션이 직접 `send()`를 두 번 호출하거나 여러 프로듀서 인스턴스가 동일 메시지를 전송하는 경우는 방지하지 못한다.

**트랜잭션(8.2)**은 카프카 스트림즈의 '읽기-처리-쓰기' 패턴에서 정확히 한 번을 보장한다. `transactional.id`를 설정한 트랜잭션 프로듀서는 원자적 다수 파티션 쓰기를 수행하며, 에포크 기반 좀비 펜싱으로 크래시 후 재기동된 스테일 프로듀서를 차단한다. 컨슈머는 `isolation.level=read_committed`로 설정해야 커밋되지 않은 트랜잭션 메시지를 건너뛴다.

트랜잭션은 카프카 내부 쓰기에만 적용되며, 외부 DB 쓰기·이메일 발송 등 부수 효과나 클러스터 간 복제 시에는 보장이 유지되지 않는다.

**트랜잭션 성능(8.3)**: 트랜잭션 오버헤드는 포함된 메시지 수와 무관한 고정 비용이므로, 배치 크기를 크게 할수록 처리량 효율이 높아진다.

## 주요 개념 목록

- 멱등적 프로듀서 (`enable.idempotence`)
- 프로듀서ID / 시퀀스 넘버
- `out of order sequence number` 에러
- 트랜잭션 프로듀서 (`transactional.id`)
- `initTransactions()` / `beginTransaction()` / `commitTransaction()` / `abortTransaction()`
- 좀비 펜싱 (Zombie Fencing) / 에포크
- `isolation.level` (`read_committed` / `read_uncommitted`)
- Last Stable Offset (LSO)
- `sendOffsetsToTransaction()`
- 원자적 다수 파티션 쓰기
- KIP-447: 컨슈머 그룹 메타데이터 기반 펜싱
- `__transaction_state` 내부 토픽
- 트랜잭션 코디네이터
- 2PC (two-phase commit)
- `transaction.timeout.ms`
