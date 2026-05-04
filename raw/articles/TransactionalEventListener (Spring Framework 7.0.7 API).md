---
title: "TransactionalEventListener (Spring Framework 7.0.7 API)"
source: "https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/event/TransactionalEventListener.html"
author:
published:
created: 2026-04-30
description: "declaration: package: org.springframework.transaction.event, annotation type: TransactionalEventListener"
tags:
  - "clippings"
---
An [`EventListener`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/event/EventListener.html "annotation interface in org.springframework.context.event") that is invoked according to a [`TransactionPhase`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/event/TransactionPhase.html "enum class in org.springframework.transaction.event"). This is an annotation-based equivalent of [`TransactionalApplicationListener`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/event/TransactionalApplicationListener.html "interface in org.springframework.transaction.event").

If the event is not published within an active transaction, the event is discarded unless the [`fallbackExecution()`](#fallbackExecution\(\)) flag is explicitly set. If a transaction is running, the event is handled according to its `TransactionPhase`.

Adding [`@Order`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/annotation/Order.html "annotation interface in org.springframework.core.annotation") to your annotated method allows you to prioritize that listener amongst other listeners running before or after transaction completion.

As of 6.1, transactional event listeners can work with thread-bound transactions managed by a [`PlatformTransactionManager`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/PlatformTransactionManager.html "interface in org.springframework.transaction") as well as reactive transactions managed by a [`ReactiveTransactionManager`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/ReactiveTransactionManager.html "interface in org.springframework.transaction"). For the former, listeners are guaranteed to see the current thread-bound transaction. Since the latter uses the Reactor context instead of thread-local variables, the transaction context needs to be included in the published event instance as the event source: see [`TransactionalEventPublisher`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/reactive/TransactionalEventPublisher.html "class in org.springframework.transaction.reactive").

**WARNING:** if the `TransactionPhase` is set to [`AFTER_COMMIT`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/event/TransactionPhase.html#AFTER_COMMIT) (the default), [`AFTER_ROLLBACK`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/event/TransactionPhase.html#AFTER_ROLLBACK), or [`AFTER_COMPLETION`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/event/TransactionPhase.html#AFTER_COMPLETION), the transaction will have been committed or rolled back already, but the transactional resources might still be active and accessible. As a consequence, any data access code triggered at this point will still "participate" in the original transaction, but changes will not be committed to the transactional resource. See [`TransactionSynchronization.afterCompletion(int)`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/support/TransactionSynchronization.html#afterCompletion\(int\)) for details.

Since:

4.2

Author:

Stephane Nicoll, Sam Brannen, Oliver Drotbohm

See Also:

- ## Optional Element Summary
	Optional Elements
	Modifier and Type
	Optional Element
	Description
	`Class<?>[]`
	`classes`
	The event classes that this listener handles.
	`String`
	`condition`
	Spring Expression Language (SpEL) attribute used for making the event handling conditional.
	`boolean`
	`fallbackExecution`
	Whether the event should be handled if no transaction is running.
	`String`
	`id`
	An optional identifier for the listener, defaulting to the fully-qualified signature of the declaring method (for example, "mypackage.MyClass.myMethod()").
	`TransactionPhase`
	`phase`
	Phase to bind the handling of an event to.
	`Class<?>[]`
	`value`
	Alias for [`classes()`](#classes\(\)).

- ## Element Details
	- ### phase
		[TransactionPhase](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/event/TransactionPhase.html "enum class in org.springframework.transaction.event") phase
		Phase to bind the handling of an event to.
		The default phase is [`TransactionPhase.AFTER_COMMIT`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/event/TransactionPhase.html#AFTER_COMMIT).
		If no transaction is in progress, the event is not processed at all unless [`fallbackExecution()`](#fallbackExecution\(\)) has been enabled explicitly.
		Default:
		`AFTER_COMMIT`
	- ### classes
		The event classes that this listener handles.
		If this attribute is specified with a single value, the annotated method may optionally accept a single parameter. However, if this attribute is specified with multiple values, the annotated method must *not* declare any parameters.
		Default:
		`{}`
	- ### condition
		Spring Expression Language (SpEL) attribute used for making the event handling conditional.
		The default is `""`, meaning the event is always handled.
		See Also:
		Default:
		`""`
	- ### fallbackExecution
		Whether the event should be handled if no transaction is running.
		See Also:
		Default:
		`false`
	- ### id
		An optional identifier for the listener, defaulting to the fully-qualified signature of the declaring method (for example, "mypackage.MyClass.myMethod()").