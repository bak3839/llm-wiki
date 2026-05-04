---
title: "Spring @TransactionalEventListener — 공식 문서"
category: source
tags: [spring, event, transaction, TransactionPhase, reactive]
sources: ["raw/articles/Transaction-bound Events  Spring Framework.md", "raw/articles/TransactionalEventListener (Spring Framework 7.0.7 API).md"]
updated: 2026-04-30
---

# Spring @TransactionalEventListener — 공식 문서

## 출처

- `raw/articles/Transaction-bound Events  Spring Framework.md`
- `raw/articles/TransactionalEventListener (Spring Framework 7.0.7 API).md`

## 핵심 요약

Spring 4.2에서 도입된 `@TransactionalEventListener`는 이벤트 리스너를 트랜잭션 완료 단계에 바인딩한다. 기본 단계는 `AFTER_COMMIT`이며 `BEFORE_COMMIT`, `AFTER_ROLLBACK`, `AFTER_COMPLETION`을 선택할 수 있다. 트랜잭션 밖에서 발행된 이벤트는 기본적으로 무시되며 `fallbackExecution=true`로 오버라이드 가능하다. AFTER_COMMIT 이후 단계에서 데이터 접근 시 트랜잭션 리소스가 살아있어 원래 트랜잭션에 참여하지만 변경은 커밋되지 않음을 주의해야 한다. Spring 6.1부터 `PlatformTransactionManager`와 `ReactiveTransactionManager` 모두 지원하며, 리액티브의 경우 `TransactionalEventPublisher`로 컨텍스트를 이벤트에 포함해야 한다.

## 주요 개념 목록

- [[wiki/spring/transactional-event-listener|@TransactionalEventListener]] — TransactionPhase, fallbackExecution, AFTER_COMMIT 데이터 접근 주의, @Order, Reactive 지원
