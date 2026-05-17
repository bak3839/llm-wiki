---
title: Redis 분산 락 (Redlock)
aliases: [redis-distributed-lock, Redlock, 분산-락]
category: distributed
tags: [redis, distributed-lock, redlock, mutual-exclusion, fault-tolerance, ttl, lua-script]
sources: ["raw/articles/Distributed Locks with Redis.md"]
updated: 2026-05-17
---

# Redis 분산 락 (Redlock)

Redis를 이용한 분산 락(Distributed Lock Manager) 구현 패턴. Redis 공식 알고리즘인 **Redlock**은 단순한 단일 인스턴스 방식보다 더 강한 안전성을 제공한다.

---

## 분산 락의 3가지 보장 속성

| 속성 | 설명 |
|------|------|
| **Safety (안전성)** | 상호 배제 — 어느 순간에도 락을 보유한 클라이언트는 하나뿐 |
| **Liveness A (교착 없음)** | 락을 보유한 클라이언트가 크래시하거나 파티션이 발생해도 결국 락 획득 가능 |
| **Liveness B (내결함성)** | Redis 노드 과반수가 살아있는 한 클라이언트는 락을 획득·해제 가능 |

---

## 단일 인스턴스 방식의 한계

Master-Replica 구조에서 단순히 페일오버(Failover)를 추가하는 방식은 안전하지 않다.

```
1. Client A가 Master에서 락 획득
2. Master가 쓰기를 Replica에 전달하기 전에 크래시
3. Replica가 Master로 승격
4. Client B가 같은 리소스에 락 획득 → 상호 배제 위반!
```

Redis 복제는 **비동기(asynchronous)** 이므로, 페일오버 시 아직 복제되지 않은 락 정보가 소실될 수 있다.

---

## 단일 인스턴스 올바른 구현

단일 인스턴스에서 안전한 락 구현의 기초.

### 락 획득

```
SET resource_name my_random_value NX PX 30000
```

- `NX`: 키가 없을 때만 설정 (원자적 조건부 쓰기)
- `PX 30000`: 30초 TTL — 클라이언트 크래시 시 자동 해제
- `my_random_value`: 클라이언트별 고유 값 (요청마다 다른 값)

### 락 해제

```
-- Redis 8.4+
DELEX resource_name IFEQ my_random_value

-- 이전 버전: Lua 스크립트로 원자적 처리
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

**단순 `DEL`을 쓰면 안 되는 이유**: TTL 만료 후 다른 클라이언트가 같은 키로 락을 획득했을 때, 이전 클라이언트의 `DEL`이 그 락을 삭제해버린다. 고유 값으로 "내가 설정한 락인지" 확인 후 삭제해야 한다.

---

## Redlock 알고리즘 (N=5)

단일 인스턴스의 단점을 극복하는 분산 알고리즘. **독립적인 N개의 Redis Master 노드**(복제 없음)를 사용한다.

### 락 획득 절차

1. **현재 시각** T1 기록 (밀리초)
2. N개 인스턴스에 **병렬로** 락 획득 시도
   - 각 인스턴스에 동일한 키·랜덤값 사용
   - 인스턴스당 타임아웃: 전체 TTL 대비 매우 작게 (예: TTL=10s → 타임아웃 5~50ms)
   - 타임아웃 내 응답 없으면 다음 인스턴스로 진행
3. **과반수 획득 판정**: 락 획득 인스턴스 수 ≥ N/2+1 **AND** 경과 시간 < TTL이면 성공
4. 실제 유효 시간 = `TTL - (현재 시각 - T1) - 클럭 드리프트 보정값`
5. **실패 시**: 락을 걸었던 모든 인스턴스에 즉시 언락 요청 (부분 획득 상태 해제)

```
N=5 예시:

인스턴스1 ✅ 획득 (5ms)
인스턴스2 ✅ 획득 (3ms)
인스턴스3 ✅ 획득 (7ms)  ← 과반수(3/5) 달성
인스턴스4 ❌ 타임아웃
인스턴스5 ✅ 획득 (4ms)

총 경과 시간: 19ms < TTL → 락 획득 성공
실제 유효 시간: TTL - 19ms
```

### 안전성 논증

과반수 인스턴스에 같은 키가 설정된 동안, 다른 클라이언트는 N/2+1개의 `SET NX`를 성공시킬 수 없다. 따라서 동시에 두 클라이언트가 락을 보유하는 것은 불가능하다.

### 재시도 전략

- 실패 시 **랜덤 딜레이** 후 재시도 → 동시 경쟁으로 인한 스플릿 브레인 방지
- 부분 획득한 락은 **즉시 해제** → TTL 만료를 기다리지 않아 가용성 향상

---

## 크래시 복구와 영속성

| 설정 | 안전성 | 비고 |
|------|-------|------|
| 영속성 없음 | ❌ | 재시작 시 락 정보 소실 → 이중 락 가능 |
| AOF (기본 fsync: 1초) | 부분적 | 파워 아웃 시 최대 1초 데이터 소실 |
| `fsync=always` | ✅ | 성능 저하 |
| **지연 재시작** | ✅ | 크래시 후 TTL 이상 대기 후 복귀 — 영속성 없이도 안전성 확보 가능 |

---

## 일관성 주의사항

Redlock은 강한 일관성을 보장하지 않는다. 높은 정확성이 필요한 경우:

1. **펜싱 토큰(Fencing Token)** 구현 필요
   - 락 획득 시마다 단조 증가하는 토큰 발급
   - 리소스 접근 시 토큰 버전 검증 → 만료된 클라이언트의 "늦은 쓰기" 차단

2. **클럭 드리프트 위험**
   - Redis는 TTL에 단조시계(monotonic clock)를 사용하지 않음
   - 서버 시각이 급격히 바뀌면 다수 프로세스가 동시에 락을 보유할 수 있음
   - NTP 설정 + 관리자 수동 시각 변경 금지로 완화 가능

> ⚠️ 참고: Martin Kleppmann은 Redlock이 진정한 강일관성(strong consistency)을 제공하지 못한다고 분석했다. 반박은 antirez의 블로그에 있다. 분산 트랜잭션 수준의 정확성이 필요하면 Zookeeper 기반 락이나 RDBMS 기반 락을 고려할 것.

---

## 관련 항목

- [[wiki/distributed/distributed-lock-patterns|분산-락-패턴]] — 분산 락의 일반적 사용 사례 및 패턴 비교 (미작성)
