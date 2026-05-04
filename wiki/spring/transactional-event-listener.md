---
title: "@TransactionalEventListener"
aliases: [TransactionalEventListener]
category: spring
tags: [spring, event, transaction, TransactionPhase, ApplicationEvent, reactive]
sources: ["raw/articles/Transaction-bound Events  Spring Framework.md", "raw/articles/TransactionalEventListener (Spring Framework 7.0.7 API).md"]
updated: 2026-04-30
---

# @TransactionalEventListener

## 개념

트랜잭션 **완료 단계**에 맞춰 이벤트 리스너를 실행하는 애노테이션 (Spring 4.2+). `@EventListener`의 트랜잭션 인식 버전이며, `TransactionalApplicationListener`와 동등한 애노테이션 기반 대안이다.

트랜잭션 밖에서 이벤트가 발행되면 기본적으로 **무시된다** (`fallbackExecution=false`).

## TransactionPhase

`phase` 속성으로 실행 시점을 지정한다. 기본값: `AFTER_COMMIT`.

| Phase | 실행 시점 |
|-------|----------|
| `BEFORE_COMMIT` | 커밋 직전 |
| `AFTER_COMMIT` (기본) | 커밋 완료 후 |
| `AFTER_ROLLBACK` | 롤백 완료 후 |
| `AFTER_COMPLETION` | 커밋 또는 롤백 완료 후 (둘 다) |

## 기본 사용법

```java
@Component
public class MyComponent {

    @TransactionalEventListener
    public void handleOrderCreatedEvent(CreationEvent<Order> creationEvent) {
        // 트랜잭션 커밋 후 실행
    }
}
```

## 주요 속성

| 속성 | 기본값 | 설명 |
|------|--------|------|
| `phase` | `AFTER_COMMIT` | 실행 트랜잭션 단계 |
| `fallbackExecution` | `false` | 트랜잭션이 없을 때도 실행할지 여부 |
| `classes` / `value` | `{}` | 처리할 이벤트 클래스 목록 |
| `condition` | `""` | SpEL 조건식 (빈 문자열 = 항상 실행) |
| `id` | 메서드 시그니처 | 리스너 식별자 |

## AFTER_COMMIT 단계의 데이터 접근 주의

> **WARNING**: `AFTER_COMMIT`, `AFTER_ROLLBACK`, `AFTER_COMPLETION` 단계에서는 트랜잭션이 이미 커밋/롤백된 상태지만 **트랜잭션 리소스(Connection 등)는 아직 활성 상태**일 수 있다. 이 시점에 데이터 접근 코드를 실행하면 원래 트랜잭션에 "참여"하는 것처럼 동작하지만, **변경 사항은 커밋되지 않는다**.

실제로 AFTER_COMMIT 리스너 내부에서 DB 쓰기를 수행하면 새 트랜잭션(`@Transactional(propagation = REQUIRES_NEW)`)을 명시적으로 열어야 변경이 커밋된다.

## @Order로 리스너 순서 지정

```java
@TransactionalEventListener
@Order(1)
public void handleFirst(MyEvent event) { ... }

@TransactionalEventListener
@Order(2)
public void handleSecond(MyEvent event) { ... }
```

같은 트랜잭션 단계에 등록된 여러 리스너의 실행 순서를 제어할 때 사용한다.

## Reactive 트랜잭션 지원 (Spring 6.1+)

| 트랜잭션 관리자 | 동작 방식 |
|---------------|----------|
| `PlatformTransactionManager` (thread-bound) | 리스너가 현재 스레드 바운드 트랜잭션을 바라봄 |
| `ReactiveTransactionManager` | 스레드 로컬 대신 Reactor Context 사용 → 이벤트 발행 시 트랜잭션 컨텍스트를 이벤트 소스에 포함해야 함 (`TransactionalEventPublisher` 사용) |

## 관련 항목

- (Spring 이벤트 관련 페이지 추가 예정)
