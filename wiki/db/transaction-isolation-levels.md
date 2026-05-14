---
title: 트랜잭션 격리 수준 (InnoDB)
aliases: [transaction-isolation-levels, 격리-수준]
category: db
tags: [mysql, innodb, transaction, isolation-level, mvcc, consistent-read, phantom-row, dirty-read]
sources: ["raw/articles/MySQL  MySQL 8.4 Reference Manual  17.7.2.1 Transaction Isolation Levels.md"]
updated: 2026-05-14
---

# 트랜잭션 격리 수준 (InnoDB)

격리 수준(Isolation Level)은 ACID의 I에 해당하며, 동시 트랜잭션 간 성능과 일관성의 균형을 조정하는 설정이다. InnoDB의 기본값은 **`REPEATABLE READ`**다.

---

## 4가지 격리 수준 비교

| 격리 수준 | 더티 리드 | 반복 불가능 읽기 | 팬텀 로우 | 갭 락 |
|----------|----------|----------------|----------|-------|
| `READ UNCOMMITTED` | 발생 | 발생 | 발생 | ❌ |
| `READ COMMITTED` | 없음 | 발생 | 발생 | ❌ |
| `REPEATABLE READ` (기본) | 없음 | 없음 | 없음¹ | ✅ |
| `SERIALIZABLE` | 없음 | 없음 | 없음 | ✅ |

> ¹ InnoDB의 넥스트키 락으로 REPEATABLE READ에서도 팬텀 로우가 방지된다. (SQL 표준상 REPEATABLE READ는 팬텀 로우를 허용하지만 InnoDB는 더 강하게 보장)

---

## REPEATABLE READ (기본값)

### 일관된 읽기 (Consistent Read)

같은 트랜잭션 내 **첫 번째 SELECT 시점의 스냅샷**을 모든 후속 SELECT에 재사용한다.

```sql
-- T1 시작
SELECT * FROM t WHERE id = 1;   -- 스냅샷 생성: value=100

-- 이 사이 다른 트랜잭션이 id=1을 200으로 UPDATE & COMMIT

SELECT * FROM t WHERE id = 1;   -- 여전히 value=100 반환 (스냅샷 유지)
```

### 잠금 읽기 (Locking Read) 규칙

| 조건 | 사용하는 락 |
|------|------------|
| 유니크 인덱스 + 유니크 조건 (`WHERE id = 5`) | 레코드 락만 |
| 범위 조건 또는 비유니크 인덱스 | 갭 락 / 넥스트키 락 |

### 주의: 잠금 SELECT + 비잠금 SELECT 혼용

같은 REPEATABLE READ 트랜잭션 내에서 `UPDATE`/`SELECT ... FOR UPDATE`와 일반 `SELECT`를 함께 쓰면 두 쿼리가 **서로 다른 데이터베이스 상태**를 볼 수 있다.
- 잠금 읽기: 최신 커밋된 상태 사용
- 비잠금 SELECT: 트랜잭션 시작 시점의 스냅샷 사용

이런 경우 `SERIALIZABLE`로 올리는 것이 권장된다.

---

## READ COMMITTED

### 특징

- 각 SELECT마다 **새 스냅샷**을 생성 → 다른 트랜잭션의 커밋이 즉시 반영됨
- **갭 락 비활성화** → 팬텀 로우 가능, 동시성 향상
- 행 기반 바이너리 로그(`binlog_format=ROW`)만 지원 (`MIXED` 설정 시 자동으로 row 모드)

### Semi-Consistent Read (UPDATE 최적화)

`UPDATE` 실행 시 InnoDB가 각 행의 **최신 커밋 버전**을 MySQL에 먼저 반환해 WHERE 조건을 평가한다. 조건에 맞지 않으면 락을 즉시 해제하여 불필요한 락 대기를 줄인다.

**REPEATABLE READ vs READ COMMITTED UPDATE 동작 비교**

```sql
-- 테이블: a INT, b INT, 인덱스 없음
-- 초기 데이터: (1,2),(2,3),(3,2),(4,3),(5,2)

-- Session A
UPDATE t SET b = 5 WHERE b = 3;   -- (2,3), (4,3) 수정

-- Session B
UPDATE t SET b = 4 WHERE b = 2;
```

| 격리 수준 | Session B 동작 |
|----------|--------------|
| `REPEATABLE READ` | A가 모든 행에 x-lock을 유지 → B는 첫 행부터 대기 |
| `READ COMMITTED` | A는 수정하지 않는 행(b=2)의 락 즉시 해제 → B는 semi-consistent read로 병렬 진행 가능 |

---

## READ UNCOMMITTED

- 비잠금 방식으로 SELECT 수행
- 다른 트랜잭션이 커밋하지 않은 데이터도 읽음 → **더티 리드(Dirty Read)**
- 그 외 락 동작은 READ COMMITTED와 동일

---

## SERIALIZABLE

- REPEATABLE READ와 동일하되, **`autocommit`이 꺼진 경우** 모든 일반 SELECT를 `SELECT ... FOR SHARE`로 자동 변환
- `autocommit`이 켜진 경우 SELECT는 자체 트랜잭션으로 처리 → 논블로킹 일관된 읽기
- 주요 사용처: XA 트랜잭션, 동시성·데드락 문제 디버깅

---

## 격리 수준 설정 방법

```sql
-- 현재 세션
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 전역 (이후 연결에 적용)
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

또는 서버 시작 옵션:
```
--transaction-isolation=READ-COMMITTED
```

---

## 관련 항목

- [[wiki/db/innodb-locking|InnoDB-락-유형]] — S/X·갭·넥스트키 락 상세 (격리 수준별 락 동작 기반)
