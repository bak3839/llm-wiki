---
title: 요청 처리
aliases: [요청-처리]
category: kafka
tags: [kafka, broker, request-handling, acceptor, processor, io-thread, purgatory, zero-copy, fetch-session, acks, ApiVersions]
sources: ["raw/articles/[Chapter 6] 카프카 내부 메커니즘 34a055f5905980f38115c3cb54c9dd73.md"]
updated: 2026-04-24
---

# 요청 처리

## 개념

카프카 브로커는 클라이언트, 팔로워 레플리카, 컨트롤러가 전송하는 요청을 처리한다. TCP 위에서 동작하는 **이진 프로토콜**을 사용하며, 특정 클라이언트의 요청은 **받은 순서대로** 처리된다 → 메시지 순서 보장.

## 스레드 구조

```
클라이언트
  └─ Acceptor 스레드 (포트별 1개) — 연결 수락 및 Processor에 위임
        └─ Processor(Network) 스레드
              ├─ 클라이언트에서 요청 읽기 → 요청 큐에 적재
              └─ 응답 큐에서 꺼내 클라이언트로 전송
                    └─ 요청 큐
                          └─ I/O 스레드 — 실제 처리
                                └─ 응답 큐 or Purgatory
```

## Purgatory

즉시 응답할 수 없는 요청을 일시 보관하는 메모리 구조:
- `acks=all` 쓰기: 모든 ISR이 복제를 완료할 때까지 대기
- 읽기의 `fetch.min.bytes`: 최소 데이터가 쌓일 때까지 대기
- 어드민: 토픽 삭제 등 작업 완료까지 대기

## 요청 공통 헤더

| 필드 | 설명 |
|------|------|
| API key | 요청 유형 식별자 |
| 요청 버전 | 클라이언트/브로커 버전 호환 지원 |
| Correlation ID | 요청-응답 매핑용 고유 식별자 |
| Client ID | 요청 애플리케이션 식별 (로깅/메트릭 등) |

---

## 쓰기 요청 (Produce Request)

해당 파티션의 **리더 레플리카를 가진 브로커**로만 전달. 클라이언트는 메타데이터 요청으로 리더 위치를 파악.

### 처리 흐름

1. 유효성 검사: 쓰기 권한, acks 값, ISR 수 충분 여부 (acks=all 시)
2. 로컬 파일 시스템 캐시에 쓰기 (디스크 반영 시점은 미보장)
3. acks 설정에 따라 응답

### acks 설정

| 값 | 동작 | 특징 |
|----|------|------|
| `0` | 응답 없이 즉시 성공 처리 | 가장 빠름, 유실 가능 |
| `1` | 리더 수신 확인 후 응답 | 기본 절충 |
| `all` (`-1`) | 모든 ISR 복제 확인 후 응답 | 가장 안전, 지연 증가 |

`acks=all`이면 리더는 요청을 **Purgatory**에 저장하고 팔로워 복제 완료 확인 후 응답한다.

> **카프카는 메시지 지속성을 디스크 fsync가 아닌 레플리카 복제에 의존한다.** `acks=all`이면 리더 장애 발생 시에도 ISR에 복제된 메시지는 유실되지 않는다.

---

## 읽기 요청 (Fetch Request)

해당 파티션의 **리더 브로커**로 전달 (단, KIP-392 팔로워 fetch 설정 시 팔로워로도 가능).

### 처리 흐름

1. 요청 오프셋 유효성 확인
2. **High Watermark** 이하 메시지만 반환 (모든 ISR에 복제된 메시지만 노출)
3. **Zero-copy**로 파일 시스템 캐시 → 네트워크 채널 직접 전송

### 최소/최대 데이터량 제어

- `fetch.min.bytes`: 최소 데이터가 모일 때까지 응답 지연 → 빈 응답 빈도 감소, CPU/네트워크 절약
- `fetch.max.wait.ms`: 최소 데이터 미충족 시 응답하는 최대 대기 시간
- 파티션별 최대 반환량 지정 → 클라이언트 메모리 과부하 방지

### Zero-copy

파일 시스템 캐시의 데이터를 **중간 버퍼 없이** 네트워크 채널로 직접 전송. 복사·버퍼 관리 오버헤드 제거로 처리량 향상.

### Fetch Session Cache

파티션 수가 많을 때 매 요청마다 전체 파티션 목록을 전송하는 오버헤드를 줄이는 최적화:
- 첫 요청에서 **세션 생성**, 이후 요청은 **변경분(incremental)만** 전송
- 브로커도 변경 사항이 있을 때만 메타데이터를 응답에 포함
- 세션 한도 초과 또는 내부 결정으로 제거될 수 있음 → 클라이언트는 전체 파티션 목록으로 재요청

---

## 기타 요청 유형

| 요청 | 발신 | 설명 |
|------|------|------|
| `Metadata` | 클라이언트 | 토픽/파티션/리더 정보 조회. 모든 브로커에 전송 가능 |
| `LeaderAndISR` | 컨트롤러 → 브로커 | 파티션 새 리더 선출 알림 |
| `UpdateMetadata` | 컨트롤러 → 브로커 | MetadataCache 갱신 |
| `OffsetCommit` / `OffsetFetch` | 컨슈머 | 오프셋 기록 (구 ZooKeeper 방식 대체) |
| `CreateTopic` | AdminClient | 토픽 생성 |
| `ApiVersions` | 클라이언트 | 브로커가 지원하는 API 버전 조회 |

### 버전 호환성

- 클라이언트 업그레이드 전 **브로커를 먼저 업그레이드** 권장
  - 새 브로커는 구버전 요청 처리 가능
  - 구 브로커는 새버전 요청 처리 불가
- `ApiVersions` 요청으로 브로커 지원 API 버전을 확인 후 적절한 버전 선택

## 관련 항목

- [[wiki/kafka/replication|복제]] — high watermark, ISR, 팔로워 복제
- [[wiki/kafka/controller|컨트롤러]] — LeaderAndISR, UpdateMetadata 요청 발생 원인
- [[wiki/kafka/follower-fetch|팔로워-페치]] — 읽기 요청이 팔로워로 전달되는 경우 (KIP-392)
- [[wiki/kafka/offset-commit|오프셋-커밋]] — OffsetCommit 요청 상세
