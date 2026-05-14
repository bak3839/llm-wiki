---
title: REPEATABLE READ에서 잠금 읽기의 팬텀 리드
aliases: [repeatable-read-phantom-locking-reads]
category: db
tags: [mysql, innodb, phantom-read, repeatable-read, locking-read, mvcc, gap-lock, for-update, for-share]
sources: []
updated: 2026-05-14
---

# REPEATABLE READ에서 잠금 읽기(FOR UPDATE/FOR SHARE)의 팬텀 리드

InnoDB의 REPEATABLE READ는 갭 락/넥스트키 락으로 팬텀 로우를 방지하지만, 이는 **비잠금 SELECT(MVCC 스냅샷)**에 한정된 이야기다. `FOR UPDATE`/`FOR SHARE` 잠금 읽기는 MVCC 스냅샷을 우회하기 때문에 특정 조건에서 팬텀이 발생한다.

---

## 읽기 원본의 근본 차이

| 읽기 방식 | 읽는 데이터 원본 | 팬텀 로우 |
|----------|----------------|----------|
| 일반 `SELECT` (비잠금) | 트랜잭션 첫 SELECT 시점의 **MVCC 스냅샷** (고정) | 없음 |
| `FOR UPDATE` / `FOR SHARE` | **현재 커밋된 최신 상태** (스냅샷 무시) | 상황에 따라 발생 |

갭 락/넥스트키 락은 **잠금 읽기 실행 이후에 발생하는 INSERT를 차단**한다. 그러나 잠금 읽기가 "현재 상태"를 읽는다는 특성상, 잠금 설정 이전에 이미 커밋된 행은 막을 수 없다.

---

## 팬텀 발생 시나리오

### 시나리오: 비잠금 SELECT + 잠금 읽기 혼용

```sql
-- T1 시작 (REPEATABLE READ)
SELECT * FROM t WHERE age > 20;
-- → 스냅샷 생성: 2행 반환 (age=25, age=30)

-- [T2: INSERT INTO t (age) VALUES (22); COMMIT]
--    T1의 스냅샷 이후, T1의 잠금 읽기 이전에 커밋

SELECT * FROM t WHERE age > 20 FOR UPDATE;
-- → 현재 상태 읽기: 3행 반환 (25, 30, 22) ← 팬텀!
-- 이 시점에 갭 락 획득 → 이후 INSERT는 차단됨

SELECT * FROM t WHERE age > 20;
-- → 여전히 스냅샷: 2행만 반환 (T2 행 보이지 않음)
```

같은 트랜잭션 내 일반 SELECT는 2행, FOR UPDATE는 3행을 반환하는 **불일치**가 팬텀 리드다.

### 팬텀의 실질적 피해

```sql
UPDATE t SET flag = 1 WHERE age > 20;
-- MVCC 스냅샷에서는 보이지 않았던 T2의 행(age=22)까지 수정됨
-- 개발자가 "2행만 수정된다"고 예상했지만 실제로는 3행 수정
```

---

## 갭 락이 막지 못하는 이유

```
타임라인:

T1 시작 ───────────── FOR UPDATE 실행 ─────────── T1 커밋
                             ↑
         [T2: INSERT + COMMIT]
          ← 이미 현재 상태에 존재. 갭 락 획득 전이라 차단 불가

갭 락 효력: ───────────────────────────────────→ (이후 INSERT 차단)
            (FOR UPDATE 실행 시점부터)
```

**핵심**: 갭 락은 "앞으로 올 INSERT"만 차단한다. 잠금 읽기 실행 이전에 이미 커밋된 행은 현재 상태의 일부로서 그대로 보인다.

---

## 왜 잠금 읽기는 스냅샷 대신 현재 상태를 쓰는가

`FOR UPDATE`/`FOR SHARE`의 목적은 "이 행들을 수정하겠다"는 의도 표명이다. 오래된 스냅샷 기반으로 락을 걸면:
- 스냅샷에 없던 신규 행이 실제로는 존재해 수정 충돌 발생 가능
- "없다"고 판단하고 진행한 로직이 실제 데이터와 불일치

따라서 잠금 읽기는 최신 커밋 상태를 기준으로 락을 설정해야 한다. 이것이 MVCC 스냅샷을 우회하는 이유다.

---

## 격리 수준별 비교

| 격리 수준 | 비잠금 SELECT | 잠금 읽기 | 일관성 |
|----------|-------------|----------|-------|
| `REPEATABLE READ` | MVCC 스냅샷 | 현재 상태 | **불일치 가능** → 팬텀 위험 |
| `SERIALIZABLE` | 현재 상태 (FOR SHARE 변환) | 현재 상태 | 일치 → 팬텀 없음 |

SERIALIZABLE에서는 `autocommit=OFF` 시 모든 일반 SELECT가 `SELECT ... FOR SHARE`로 자동 변환되어 읽기 원본이 통일된다.

---

## 해결책

| 상황 | 해결책 |
|------|--------|
| 잠금/비잠금 읽기를 같은 트랜잭션에서 혼용 | `SERIALIZABLE`로 격리 수준 올리기 |
| REPEATABLE READ 유지 필요 | 트랜잭션 내 모든 읽기를 잠금 읽기로 통일, 또는 모든 읽기를 비잠금으로 통일 |
| 두 읽기의 결과 일치가 비즈니스적으로 중요 | `SERIALIZABLE` 또는 애플리케이션 레벨 잠금 |

---

## 관련 항목

- [[wiki/db/transaction-isolation-levels|트랜잭션-격리-수준]] — 격리 수준별 스냅샷/잠금 읽기 동작, SERIALIZABLE 동작
- [[wiki/db/innodb-locking|InnoDB-락-유형]] — 갭 락·넥스트키 락 상세
