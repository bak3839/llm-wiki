---
title: "MySQL 8.4 Reference Manual — 17.7.2.1 Transaction Isolation Levels"
category: source
tags: [mysql, innodb, transaction, isolation-level, mvcc, db]
sources: ["raw/articles/MySQL  MySQL 8.4 Reference Manual  17.7.2.1 Transaction Isolation Levels.md"]
updated: 2026-05-14
---

# MySQL 8.4 Reference Manual — 17.7.2.1 Transaction Isolation Levels

## 출처

`raw/articles/MySQL  MySQL 8.4 Reference Manual  17.7.2.1 Transaction Isolation Levels.md`  
원문: https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html

## 핵심 요약

InnoDB는 SQL:1992 표준의 4가지 격리 수준을 모두 지원하며 기본값은 `REPEATABLE READ`다.

**REPEATABLE READ**: 트랜잭션 내 첫 읽기 시점의 스냅샷을 유지한다(일관된 읽기). 잠금 읽기(`FOR UPDATE`/`FOR SHARE`), `UPDATE`, `DELETE`에는 유니크 인덱스+유니크 조건이면 레코드 락만, 범위 조건이면 갭 락/넥스트키 락을 사용해 팬텀 로우를 방지한다.

**READ COMMITTED**: 각 읽기마다 새 스냅샷을 찍는다. 갭 락이 비활성화되어 팬텀 로우가 발생할 수 있고 동시성은 높아진다. `UPDATE` 시 semi-consistent read로 WHERE 조건에 맞지 않는 행의 락을 즉시 해제한다. 행 기반 바이너리 로그만 지원한다.

**READ UNCOMMITTED**: 비잠금 SELECT로 커밋되지 않은 데이터도 읽는다(더티 리드). 그 외 동작은 READ COMMITTED와 동일하다.

**SERIALIZABLE**: REPEATABLE READ와 동일하나, `autocommit`이 꺼진 경우 모든 일반 SELECT를 `SELECT ... FOR SHARE`로 자동 변환한다. XA 트랜잭션·데드락 디버깅에 주로 사용한다.

## 주요 개념 목록

- 4가지 격리 수준: READ UNCOMMITTED / READ COMMITTED / REPEATABLE READ / SERIALIZABLE
- 일관된 읽기 (Consistent Read) / 스냅샷 읽기 (MVCC)
- 잠금 읽기 (Locking Read): `SELECT ... FOR UPDATE` / `SELECT ... FOR SHARE`
- 더티 리드 (Dirty Read)
- 팬텀 로우 (Phantom Row)
- Semi-consistent Read (READ COMMITTED의 UPDATE 최적화)
- 갭 락 비활성화 (READ COMMITTED)
- `SET TRANSACTION` / `--transaction-isolation`
- `autocommit`과 SERIALIZABLE의 관계
