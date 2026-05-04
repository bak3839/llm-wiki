---
title: "[Chapter 7] 신뢰성 있는 데이터 전달 (2부)"
category: source
tags: [kafka, producer, consumer, reliability, acks, idempotence, retry, consumer-lag, monitoring, DLQ]
sources: ["raw/articles/Chapter 7 신뢰성 있는 데이터 전달2.md"]
updated: 2026-05-05
---

# [Chapter 7] 신뢰성 있는 데이터 전달 (2부)

## 출처

`raw/articles/Chapter 7 신뢰성 있는 데이터 전달2.md`

## 핵심 요약

프로듀서 측 신뢰성은 acks 설정과 에러 처리 두 축으로 관리한다. acks=1은 리더 크래시 시 유실 가능, acks=all은 모든 ISR 복제 보장. 재시도는 at-least-once를 보장하며 enable.idempotence=true로 중복을 방지한다. 컨슈머 측에서는 처리 완료 후 오프셋 커밋, 정확한 오프셋 커밋, 리밸런스 처리가 핵심이다. 재시도 필요 레코드는 pause()+버퍼 또는 DLQ 토픽 패턴으로 처리한다. 상태 유지 컨슈머는 상태를 별도 토픽에 기록해 재시작 시 복구한다. 시스템 검증은 VerifiableProducer/Consumer로 설정 검증, Trogdor로 장애 주입 테스트를 수행한다. 프로덕션 모니터링의 핵심 지표는 consumer lag이며 Burrow 활용을 권장한다.

## 주요 개념 목록

- [[wiki/kafka/producer-reliability|프로듀서-신뢰성]] — acks 0/1/all 상세, delivery.timeout.ms, retries, enable.idempotence, 에러 유형
- [[wiki/kafka/offset-commit|오프셋-커밋]] — 컨슈머 재시도 패턴(pause+버퍼, DLQ), 상태 유지 컨슈머, 커밋 빈도 트레이드오프
- [[wiki/kafka/reliability|신뢰성]] — consumer lag 모니터링, JMX 지표, VerifiableProducer/Consumer
