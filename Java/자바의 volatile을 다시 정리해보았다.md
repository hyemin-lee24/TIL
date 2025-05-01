# 자바의 `volatile`을 다시 정리해보았다  



## 🧠 메모리 가시성과 happens-before 관계



### 🔍 왜 다시 보게 되었나?



5년차가 되면서 동시성 문제를 다뤄야 할 일이 늘어났다.
하지만 `volatile`을 정확히 설명하라고 하면 **막연하게 "가시성 보장" 정도로만 기억**하고 있었음. 
이번 기회에 **JMM(Java Memory Model)**과 함께 개념을 다시 정리.

---

### 📌 핵심 정리

| 개념                | 설명                                                         |
| ------------------- | ------------------------------------------------------------ |
| 가시성 (visibility) | 변수의 최신 값이 모든 스레드에서 보이는 것                   |
| `volatile`          | 쓰기 시 즉시 main memory로 flush, <br />읽기 시 main memory에서 가져옴 |
| happens-before      | A 스레드가 `volatile`에 값을 쓴 후 B 스레드가 그 값을 읽으면, A의 이전 작업은 B에게 보임 |
| 원자성 (atomicity)  | `volatile`은 보장하지 않음 (`count++`는 여전히 위험)         |

---

### 🧪 실전 예제: 상태 플래그에 적합

```java
class Worker extends Thread {
    private volatile boolean running = true;

    public void run() {
        while (running) {
            // 반복 작업 수행
        }
    }

    public void shutdown() {
        running = false;
    }
}
```

- `shutdown()`을 호출하면 `running`이 false로 바뀌고,
- `run()` 루프가 종료됨.
- `volatile` 없으면 루프가 끝나지 않을 수 있음 (가시성 문제).

------

### ⚠️ 주의: `volatile`은 복합 연산에 불안전

```java
volatile int count = 0;

public void increment() {
    count++; // 위험! 원자적이지 않음
}
```

- `++`는 read-modify-write.
- `AtomicInteger`나 `synchronized` 필요.

------

### 💡 예제: Double-Checked Locking (DCL)

```java
public class Singleton {
    private static volatile Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton(); // 생성자 + 할당 재정렬 방지
                }
            }
        }
        return instance;
    }
}
```

- `volatile` 없으면 재정렬로 인해 partially constructed 객체가 노출될 수 있음.

------

### ❌ 자주 하는 오해

| 오해                               | 진실                              |
| ---------------------------------- | --------------------------------- |
| `volatile`이면 동기화 안 해도 된다 | 복합 연산엔 불충분                |
| 항상 lock보다 빠르다               | 케이스에 따라 lock이 더 빠를 수도 |
| 모든 멀티스레드 문제를 해결해준다  | 대부분은 불가능                   |



------

### ✅ 정리

- `volatile`은 **동시성 툴박스 중 하나일 뿐**, 만능은 아님.
- 정확한 사용 시점은 다음과 같음:
  - 단순 상태 공유
  - 플래그 전달
  - 초기화된 객체의 안전한 공개