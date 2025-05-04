# Java 동시성 - ReentrantLock 깊이 있게 이해하기

## 1. 왜 `ReentrantLock`을 써야 했는가?

실제 서비스 운영 중, 특정 요청 처리에서 간헐적인 latency spike가 발생했다. 

GC 로그를 확인해보니, `synchronized` 블록을 보유한 쓰레드가 GC로 인해 STW(Stop-The-World) 상태에 빠지면서, 다른 쓰레드들이 락을 기다리며 지연이 급격히 증가한 현상이 원인이었다.

### 문제점
- `synchronized`는 대기 중 인터럽트나 타임아웃 처리 불가 → 장애 시 복구 어려움
- 락 점유 시간에 대한 로깅/모니터링이 어려움

이 문제를 해결하기 위해 `ReentrantLock`을 도입했다.

---

## 2. `ReentrantLock`이란?

`java.util.concurrent.locks` 패키지에 포함된 명시적 락 구현체로, `synchronized`보다 유연하게 쓰레드 동기화 제어 가능.

### 주요 특징
- **재진입 가능**: 동일 쓰레드가 중첩 락 획득 가능
- **공정성 설정 가능**: FIFO 순서대로 락 부여 (`new ReentrantLock(true)`)
- **tryLock / lockInterruptibly 지원**: 데드락 방지, 응답성 향상
- **Condition 객체로 조건 동기화 지원**: `wait/notify` 보다 정교

---

## 3. 동작 원리 (AQS 기반)

내부적으로 `ReentrantLock`은 `AbstractQueuedSynchronizer` (AQS)를 사용한다.

- CAS로 락 상태 변경 → 경합 최소화
- 락이 이미 점유되면 FIFO 큐에 쓰레드가 대기
- `fair=true`일 경우, 대기 순서대로 락 부여 (but 성능 손실 있음)

> 기본값은 `fair=false` → 락 획득 순서 보장은 없지만 Throughput은 더 좋다

---

## 4. 적용 코드 예시

```java
private final ReentrantLock lock = new ReentrantLock();

public void process() {
    long start = System.nanoTime();
    if (lock.tryLock(500, TimeUnit.MILLISECONDS)) {  // 타임아웃을 사용한 예시
        try {
            // 공유 자원 접근
        } finally {
            lock.unlock();
            log.info("Lock held for {} ms", (System.nanoTime() - start) / 1_000_000);
        }
    } else {
        log.warn("Lock acquisition timed out, triggering fallback");
        // fallback 처리
    }
}
```

### 적용 결과

- 락 획득 실패 시 fallback 처리 가능 → 요청 유실/지연 감소
- 락 점유 시간 측정 및 로그로 병목 구간 파악 가능
- GC 간섭으로 인한 일시 정지 문제 완화

------

## 5. 주의할 점

- `unlock()` 누락 시, 해당 락은 영원히 해제되지 않음 → 반드시 `try-finally`로 관리
- `fair=true`는 공정성은 보장되지만 Throughput은 감소
- `tryLock()` 반복 시 busy spin 가능 → 적절한 대기 또는 backoff 전략 필요

------

## 6. GC로 인한 STW 지연 확인 방법

GC로 인한 **Stop-The-World(STW)** 상태에 의해 발생한 지연 **GC 로그**에서 확인

**GC 덤프**는 객체 스냅샷만을 제공하고, **STW 시간**을 포함하지 않기 때문에 GC 로그를 확인 필요

### GC 로그 수집 방법

- JVM 옵션에 GC 로그 수집 활성화

  ```
  -Xlog:gc*:file=gc.log:time,level,tags
  ```

  또는 Java 8 이하에서는:

  ```
  -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log
  ```

### GC 로그 내 STW 시간 확인

예시 로그:

```
[GC (Allocation Failure)  512M->256M(1024M), 0.2567890 secs]
```

- 여기서 `0.2567890 secs`는 **STW 시간**입니다. 이 시간 동안 모든 쓰레드가 멈추게 되며, **락을 보유한 쓰레드가 GC로 인해 작업을 계속하지 못하게 됩니다**.

### 스레드 덤프 분석

- latency spike 발생 시간대에 `jstack`으로 스레드 덤프를 떴을 때, 락을 보유한 쓰레드가 **`runnable` 상태인데도 멈춰 있는** 것을 확인할 수 있다면 STW 중일 가능성이 높습니다.
- GC 로그와 타임스탬프를 비교하여 **정황 증명** 가능

------

## 7. 결론: 단순 동기화가 아닌 "제어가 필요한 동기화"를 위해

서비스 운영 중 발생하는 예외적인 락 경합 상황, 장애 복구 가능성, 병목 분석 등의 **운영 관점 요구사항**을 충족시키려면 `ReentrantLock`이 유리하다. 단순한 동기화는 `synchronized`로도 충분하지만, **복잡한 환경을 제어하려면 명시적 락을 쓰는 것이 불가피**하다는 걸 실무에서 체감하게 됐다.