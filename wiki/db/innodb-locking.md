---
title: InnoDB 락 유형
aliases: [innodb-locking, InnoDB-락]
category: db
tags: [mysql, innodb, locking, gap-lock, next-key-lock, record-lock, intention-lock, transaction]
sources: ["raw/articles/MySQL  MySQL 8.4 Reference Manual  17.7.1 InnoDB Locking.md"]
updated: 2026-05-14
---

# InnoDB 락 유형

InnoDB는 동시성 제어를 위해 여러 수준의 락을 조합하여 사용한다. 기본 격리 수준은 `REPEATABLE READ`이며, 넥스트키 락이 기본으로 적용된다.

---

## 1. Shared Lock (S) / Exclusive Lock (X)

행 레벨 락의 기본 단위.

| 락 | 허용 작업 | 별칭 |
|----|----------|------|
| S (Shared) | 읽기 | 공유 락 |
| X (Exclusive) | 수정·삭제 | 배타 락 |

**호환성**

| 요청 \ 보유 | S | X |
|------------|---|---|
| S | ✅ 허용 | ❌ 대기 |
| X | ❌ 대기 | ❌ 대기 |

- `SELECT ... FOR SHARE` → S락 획득
- `SELECT ... FOR UPDATE`, `UPDATE`, `DELETE` → X락 획득

---

## 2. 인텐션 락 (Intention Locks)

**다중 세분성 락킹(Multi-Granularity Locking)** 을 지원하기 위한 테이블 레벨 신호 락이다. 행 락을 걸기 전에 먼저 테이블에 IS/IX를 등록함으로써, 전체 테이블 락 요청이 진행 중인 행 락과 충돌하는지 빠르게 확인할 수 있다.

| 락 | 의미 | 설정 시점 |
|----|------|----------|
| IS (Intention Shared) | 특정 행에 S락을 걸 예정 | `SELECT ... FOR SHARE` |
| IX (Intention Exclusive) | 특정 행에 X락을 걸 예정 | `SELECT ... FOR UPDATE` |

**테이블 레벨 호환성 매트릭스**

|  | X | IX | S | IS |
|--|---|----|---|----|
| **X** | ❌ | ❌ | ❌ | ❌ |
| **IX** | ❌ | ✅ | ❌ | ✅ |
| **S** | ❌ | ❌ | ✅ | ✅ |
| **IS** | ❌ | ✅ | ✅ | ✅ |

IS/IX끼리는 항상 호환 → 여러 트랜잭션이 동시에 행 락을 준비할 수 있다. 전체 테이블 X락(`LOCK TABLES ... WRITE`)만 IS/IX와 충돌한다.

---

## 3. 레코드 락 (Record Lock)

**인덱스 레코드**에 거는 락. 항상 인덱스를 기준으로 작동한다.

```sql
SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE;
-- t.c1 = 10인 인덱스 레코드를 X락으로 잠금
-- 다른 트랜잭션의 INSERT/UPDATE/DELETE(c1=10) 차단
```

인덱스가 없는 테이블에도 InnoDB가 **내부 숨겨진 클러스터드 인덱스**를 생성해 레코드 락을 적용한다.

`SHOW ENGINE INNODB STATUS` 출력 예:
```
lock_mode X locks rec but not gap
```

---

## 4. 갭 락 (Gap Lock)

인덱스 레코드 **사이의 간격**, 또는 첫 레코드 앞/마지막 레코드 뒤의 공간에 거는 락. 해당 범위에 새 레코드가 **삽입되는 것을 방지**한다.

```sql
SELECT c1 FROM t WHERE c1 BETWEEN 10 AND 20 FOR UPDATE;
-- c1 값 10~20 범위 전체의 갭을 잠금
-- 현재 15가 없어도 다른 트랜잭션의 INSERT INTO t (c1) VALUES (15) 차단
```

**특징**

- **순수 억제(purely inhibitive)**: 삽입 방지만이 목적
- 갭 락끼리는 충돌하지 않는다 — 여러 트랜잭션이 같은 갭에 갭 락을 동시에 보유 가능
- S 갭락과 X 갭락의 차이가 없음 (기능이 동일)
- 유니크 인덱스로 단일 행을 조회하는 경우 갭 락 불필요 (레코드 락만 사용)
- `READ COMMITTED` 격리 수준에서는 비활성화 (외래 키·중복 키 검사 제외)

---

## 5. 넥스트키 락 (Next-Key Lock)

```
넥스트키 락 = 레코드 락 + 해당 레코드 앞의 갭 락
```

`REPEATABLE READ`에서 InnoDB의 **기본 락 방식**. 팬텀 로우(Phantom Row) 삽입을 방지한다.

인덱스에 값 10, 11, 13, 20이 있다면 가능한 넥스트키 락 구간:

```
(-∞, 10]
(10, 11]
(11, 13]
(13, 20]
(20, +∞)   ← supremum pseudo-record (가장 큰 값 이후의 갭)
```

`SHOW ENGINE INNODB STATUS` 출력 예:
```
lock_mode X
```
(갭 락과 레코드 락이 결합된 형태)

---

## 6. 인서트 인텐션 락 (Insert Intention Lock)

`INSERT` 직전에 대상 갭에 설정하는 **특수 갭 락**. 같은 갭에 삽입하더라도 삽입 위치가 다르면 서로를 블록하지 않는다.

**예시**: 인덱스에 4와 7이 존재할 때
- 트랜잭션 A: 5 삽입 시도 → 갭(4,7)에 인서트 인텐션 락
- 트랜잭션 B: 6 삽입 시도 → 갭(4,7)에 인서트 인텐션 락
- A와 B는 **서로를 기다리지 않음** (삽입 위치가 다름)

단, 다른 트랜잭션이 갭(4,7)에 X락(갭 락)을 보유하고 있으면 인서트 인텐션 락은 대기해야 한다.

---

## 7. AUTO-INC 락

`AUTO_INCREMENT` 열이 있는 테이블에 INSERT 시 획득하는 **테이블 레벨 락**. 연속적인 PK 값 생성을 보장한다.

`innodb_autoinc_lock_mode` 변수로 동작 방식을 조정할 수 있다:

| 값 | 모드 | 특징 |
|----|------|------|
| 0 | traditional | 항상 테이블 락 |
| 1 | consecutive | 단순 INSERT는 경량 락 사용 (기본값 변경됨) |
| 2 | interleaved | 락 없이 최대 동시성, 연속성 미보장 |

---

## 8. 프레디케이트 락 (Predicate Lock)

공간 인덱스(`SPATIAL`)에 사용되는 특수 락. 다차원 데이터에는 "다음 키" 개념이 없어 넥스트키 락을 적용할 수 없기 때문에, **MBR(최소 경계 직사각형)** 값에 락을 설정한다.

---

## 격리 수준별 락 동작 요약

| 격리 수준 | 갭 락 | 넥스트키 락 | 팬텀 로우 방지 |
|----------|-------|------------|--------------|
| `READ COMMITTED` | ❌ (FK·중복 키 검사 제외) | ❌ | ❌ |
| `REPEATABLE READ` | ✅ | ✅ (기본) | ✅ |
| `SERIALIZABLE` | ✅ | ✅ | ✅ |

`READ COMMITTED`에서는 WHERE 조건에 맞지 않는 행의 레코드 락이 즉시 해제되고, UPDATE 시 semi-consistent read를 수행한다.

---

## 관련 항목

- [[wiki/db/transaction-isolation-levels|트랜잭션-격리-수준]] — 격리 수준별 갭 락·넥스트키 락 동작 차이
- [[wiki/kafka/kafka-transactions|카프카-트랜잭션]] — 카프카에서의 원자성 보장 (비교 참고)
