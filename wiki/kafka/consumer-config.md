---
title: 컨슈머 설정
aliases: [컨슈머-설정]
category: kafka
tags: [kafka, consumer, configuration, fetch, session, heartbeat, offset, partition-assignment]
sources: [raw/articles/kafka_chapter_4.md]
updated: 2026-04-21
---

# 컨슈머 설정

## 설정 우선순위 (중요도 순)

1. 오프셋 처리 의미 (enable.auto.commit, auto.offset.reset)
2. 컨슈머 생존과 리밸런스 (session.timeout.ms, heartbeat.interval.ms, max.poll.interval.ms)
3. 파티션 할당 방식 (partition.assignment.strategy, group.instance.id)
4. 처리량과 메모리 (max.poll.records, max.partition.fetch.bytes, fetch.max.bytes)
5. fetch 지연과 배치 크기 (fetch.min.bytes, fetch.max.wait.ms)
6. 요청 타임아웃 (request.timeout.ms, default.api.timeout.ms)
7. 식별·네트워크 (client.id, client.rack, receive.buffer.bytes, send.buffer.bytes)
8. 오프셋 보존 (offsets.retention.minutes)

---

## 오프셋 처리

### `enable.auto.commit`
- 자동 오프셋 커밋 여부. 기본값: `true`
- `false`로 설정하면 `commitSync()` / `commitAsync()` 로 직접 제어
- 자세한 내용: [[오프셋-커밋]]

### `auto.offset.reset`
- 유효한 커밋 오프셋이 없을 때의 시작 위치
  - `latest` (기본): 컨슈머 시작 이후 새로 쓰여진 레코드부터
  - `earliest`: 파티션의 맨 처음부터
- 자세한 비교: [[auto-offset-reset]]

---

## 컨슈머 생존과 리밸런스

### `session.timeout.ms`
- 하트비트가 없어도 살아있다고 판정하는 최대 시간
- 기본값: `45초` (3.0부터 변경, 이전은 10초)
- 초과 시 그룹 코디네이터가 해당 컨슈머를 사망으로 간주하고 리밸런스 트리거

### `heartbeat.interval.ms`
- 하트비트 전송 주기
- `session.timeout.ms`의 약 1/3 권장
- 주의: `session.timeout.ms=45s`인 경우 1/3 규칙이 기본값에서 유효하지 않을 수 있음

### `max.poll.interval.ms`
- 컨슈머가 `poll()` 없이도 살아있다고 판정하는 최대 시간. 기본값: `5분`
- 하트비트는 백그라운드 스레드가 보내므로, 메인 스레드 데드락 시에도 하트비트는 살아있을 수 있음 → 이 설정으로 커버
- 초과 시 백그라운드 스레드가 `leave group` 요청 전송 후 하트비트 중단
- `max.poll.records`와 함께 설정: 레코드 수 × 처리 시간 < max.poll.interval.ms

---

## 파티션 할당 방식

### `partition.assignment.strategy`
- 컨슈머에 파티션을 할당하는 전략

| 전략 | 특징 |
|------|------|
| Range | 각 토픽의 파티션을 연속된 구간으로 분할. 파티션이 홀수면 첫 컨슈머에 더 많이 배분될 수 있음 |
| RoundRobin | 모든 파티션을 순차적으로 하나씩 할당. 균등 분배 |
| Sticky | RoundRobin과 유사하되, 리밸런스 시 기존 할당을 최대한 유지하여 이동 오버헤드 최소화 |
| CooperativeSticky | Sticky + 협력적 리밸런스 지원 (기본값) |

### `group.instance.id`
- 정적 그룹 멤버십 활성화. 재시작 시 리밸런스 없이 기존 파티션 유지
- 자세한 내용: [[컨슈머-그룹]]

---

## 처리량과 메모리

### `fetch.max.bytes`
- 브로커로부터 받는 데이터 최대 바이트. 기본값: `50MB`
- 첫 레코드 배치가 이 값을 초과해도 해당 배치는 그대로 전송 (읽기 진행 보장)

### `max.partition.fetch.bytes`
- 파티션별 최대 리턴 바이트. 기본값: `1MB`
- 조절이 복잡하므로 특별한 이유가 없으면 `fetch.max.bytes` 사용 권장

### `max.poll.records`
- `poll()` 호출당 리턴되는 최대 레코드 수
- 메모리 버퍼에 레코드가 남아있으면 다음 `poll()`은 네트워크 요청 없이 버퍼에서 반환

---

## Fetch 지연과 배치 크기

### `fetch.min.bytes`
- 브로커가 응답하기 전 모아야 할 최소 데이터 크기. 기본값: `1바이트`
- 값을 올리면 처리량이 적을 때 브로커/컨슈머 부하 감소, 하지만 지연 증가

### `fetch.max.wait.ms`
- `fetch.min.bytes`를 채우지 못한 경우 최대 대기 시간. 기본값: `500ms`

---

## 요청 타임아웃

### `default.api.timeout.ms`
- 대부분의 컨슈머 API 기본 타임아웃. 기본값: `1분`
- `poll()`에는 적용되지 않음 (별도 타임아웃 지정 필요)

### `request.timeout.ms`
- 브로커 응답 대기 최대 시간. 기본값: `30초`
- 초과 시 연결 종료 후 재연결 시도

---

## 식별·네트워크

### `client.id`
- 브로커가 요청 클라이언트를 식별하는 문자열. 로깅/모니터링/메트릭/쿼터에 사용

### `client.rack`
- 컨슈머 위치(AZ 등)를 지정. Kafka 2.4.0+에서 지원
- 같은 `broker.rack`을 가진 레플리카를 우선 읽어 지연/비용 절감
- 브로커의 `replica.selector.class=org.apache.kafka.common.replica.RackAwareReplicaSelector` 필요
- 자세한 내용: [[wiki/kafka/follower-fetch|팔로워-페치]] (KIP-392)

### `receive.buffer.bytes` / `send.buffer.bytes`
- 소켓 TCP 버퍼 크기. `-1` = OS 기본값
- 다른 데이터센터의 브로커와 통신 시 올려잡는 것을 권장

---

## 오프셋 보존

### `offsets.retention.minutes`
- 브로커 설정. 컨슈머 그룹이 비어있을 때 커밋된 오프셋 보관 기간
- 기간 초과 후 오프셋이 삭제된 상태에서 그룹이 재활동하면 새 컨슈머 그룹처럼 동작

---

## 관련 항목

- [[컨슈머-그룹]] — group.id, group.instance.id, 정적 멤버십
- [[리밸런스]] — session.timeout.ms, heartbeat.interval.ms, partition.assignment.strategy
- [[오프셋-커밋]] — enable.auto.commit, auto.offset.reset
- [[폴링-루프]] — max.poll.records, max.poll.interval.ms와 poll() 내부 동작
- [[팔로워-페치]] — client.rack으로 활성화하는 가장 가까운 레플리카 fetch (KIP-392)
