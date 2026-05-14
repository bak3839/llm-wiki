---
title: "MySQL 8.4 Reference Manual — 17.7.1 InnoDB Locking"
category: source
tags: [mysql, innodb, locking, db, transaction]
sources: ["raw/articles/MySQL  MySQL 8.4 Reference Manual  17.7.1 InnoDB Locking.md"]
updated: 2026-05-14
---

# MySQL 8.4 Reference Manual — 17.7.1 InnoDB Locking

## 출처

`raw/articles/MySQL  MySQL 8.4 Reference Manual  17.7.1 InnoDB Locking.md`  
원문: https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html

## 핵심 요약

InnoDB가 사용하는 8가지 락 유형을 정의한다.

**행 레벨 락(S/X)**은 읽기(Shared)와 쓰기(Exclusive) 접근을 제어한다. S락은 여럿이 동시에 획득 가능하지만, X락은 단독 점유여야 한다.

**인텐션 락(IS/IX)**은 행 락을 걸기 전에 테이블 수준에서 미리 등록하는 신호다. 다중 세분성 락킹(Multi-Granularity Locking)을 가능하게 하며, 전체 테이블 락(`LOCK TABLES ... WRITE`)과의 호환성 검사에 사용된다.

**레코드 락**은 인덱스 레코드를 잠근다. 인덱스가 없는 테이블도 InnoDB가 내부 클러스터드 인덱스를 생성해 레코드 락을 적용한다.

**갭 락**은 인덱스 레코드 사이의 공간을 잠가 팬텀 로우 삽입을 방지한다. `READ COMMITTED` 격리 수준에서는 외래 키·유니크 키 검사를 제외하고 비활성화된다. 갭 락끼리는 충돌하지 않는다(순수 억제 목적).

**넥스트키 락** = 레코드 락 + 해당 레코드 앞 갭 락. `REPEATABLE READ`의 기본 락 방식이며 팬텀 로우를 방지한다.

**인서트 인텐션 락**은 INSERT 직전에 갭에 설정하는 특수 갭 락으로, 같은 갭에 삽입하더라도 위치가 다르면 서로 블록하지 않는다.

**AUTO-INC 락**은 AUTO_INCREMENT 열에 삽입 시 취하는 테이블 레벨 락이다. `innodb_autoinc_lock_mode`로 동작 방식을 조정할 수 있다.

**프레디케이트 락**은 공간 인덱스(SPATIAL)에 사용되며, MBR(최소 경계 직사각형) 값에 락을 설정한다.

## 주요 개념 목록

- Shared Lock (S) / Exclusive Lock (X)
- Intention Shared Lock (IS) / Intention Exclusive Lock (IX)
- Multi-Granularity Locking
- Record Lock
- Gap Lock
- Next-Key Lock
- Insert Intention Lock
- AUTO-INC Lock
- Predicate Lock
- `READ COMMITTED` vs `REPEATABLE READ` 락 동작 차이
- 팬텀 로우(Phantom Row)
- `innodb_autoinc_lock_mode`
