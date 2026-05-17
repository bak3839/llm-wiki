---
title: MySQL 기본 격리 수준이 REPEATABLE READ인 이유
aliases: [mysql-default-isolation-level, MySQL-기본-격리수준]
category: db
tags: [mysql, innodb, isolation-level, repeatable-read, binlog, replication, statement-based-replication]
sources: []
updated: 2026-05-16
---

# MySQL 기본 격리 수준이 REPEATABLE READ인 이유

MySQL InnoDB의 기본 격리 수준은 `REPEATABLE READ`다. 이는 단순한 기본값 선택이 아니라 **스테이트먼트 기반 바이너리 로그(Statement-Based Replication, SBR)** 와의 호환성을 위한 역사적 설계 결정이다.

---

## 핵심 원인: 바이너리 로그와 복제 일관성

MySQL의 바이너리 로그(binlog)는 원래 **Statement 기반**으로 동작했다. 소스 서버에서 실행된 SQL 문장을 그대로 레플리카에 전송해 재실행하는 방식이다.

`READ COMMITTED`를 기본값으로 사용하면:

```sql
-- Source 서버에서 실행
UPDATE orders SET status = 'processed'
WHERE created_at < '2024-01-01';
-- READ COMMITTED: 각 행을 읽는 시점마다 새 스냅샷
-- → 실행 도중 다른 트랜잭션이 INSERT한 행이 포함될 수 있음

-- Replica 서버에서 동일 SQL 재실행
-- → 다른 시점에 실행되므로 수정 대상 행이 달라질 수 있음
```

소스와 레플리카에서 **같은 SQL이 다른 행에 적용**되어 데이터 불일치가 발생한다.

---

## REPEATABLE READ가 문제를 해결하는 방법

| 메커니즘 | 역할 |
|---------|------|
| **MVCC 스냅샷** | 트랜잭션 내 모든 SELECT가 동일한 시점의 데이터를 읽음 → 실행 결과 결정적 |
| **넥스트키 락** | 범위 조건에 갭 락을 설정해 트랜잭션 도중 다른 트랜잭션의 INSERT 차단 → 대상 행 집합 고정 |

같은 `UPDATE ... WHERE` 문을 실행해도 항상 동일한 행 집합에 적용되므로, 레플리카에서 재실행해도 소스와 동일한 결과가 보장된다.

---

## READ COMMITTED가 SBR에서 안전하지 않은 이유

```
READ COMMITTED + 갭 락 없음
→ 트랜잭션 도중 다른 INSERT 허용
→ WHERE 범위에 속하는 행 집합이 실행 중 변할 수 있음
→ binlog에 기록된 SQL을 레플리카에서 재실행하면 다른 결과
```

이 때문에 MySQL은 `READ COMMITTED` 사용 시 **row-based binlog(`binlog_format=ROW`)를 강제**한다. SQL 문장 대신 변경된 행 데이터 자체를 기록해 복제 불일치를 방지한다.

---

## 왜 SERIALIZABLE이 아닌가

| 격리 수준 | 복제 안전성 | 동시성 |
|----------|------------|-------|
| `SERIALIZABLE` | ✅ 완전 보장 | ❌ 모든 SELECT가 FOR SHARE → 락 경합 증가 |
| **`REPEATABLE READ`** | ✅ SBR 안전 | ✅ 비잠금 읽기는 락 없이 스냅샷 사용 |
| `READ COMMITTED` | ❌ SBR 불안전 | ✅✅ 갭 락 없어 동시성 최고 |

REPEATABLE READ는 복제 안전성과 동시성 사이의 **최적 균형점**이다.

---

## 현재 시점 관점

`binlog_format=ROW`가 기본값이 된 MySQL 8.0 이후로는 복제 관점의 제약이 줄었다. REPEATABLE READ가 기본값을 유지하는 이유는 **하위 호환성**과 기존 애플리케이션의 MVCC 스냅샷 의존 코드를 깨지 않기 위해서다.

---

## 관련 항목

- [[wiki/db/transaction-isolation-levels|트랜잭션-격리-수준]] — 4가지 격리 수준 비교, READ COMMITTED와 binlog_format 제약
- [[wiki/db/innodb-locking|InnoDB-락-유형]] — 넥스트키 락·갭 락 상세
- [[wiki/db/repeatable-read-phantom-locking-reads|REPEATABLE-READ-팬텀-잠금읽기]] — 잠금 읽기 혼용 시 팬텀 발생 예외
