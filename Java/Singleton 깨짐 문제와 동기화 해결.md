# 🔐 Singleton 깨짐 문제와 동기화 해결

## 📌 문제 상황

서비스 초기화 시점에 전역 싱글톤 객체(`ConfigManager`)를 lazy하게 생성하고자 

아래 예시처럼 구현:

```java
public class ConfigManager {
    private static ConfigManager instance;

    public static ConfigManager getInstance() {
        if (instance == null) {
            instance = new ConfigManager(); // ❗ 동시성 문제 발생
        }
        return instance;
    }
}
```

### 🧨 증상

- 대량 요청이 몰리면 동시에 여러 스레드가 `getInstance()` 진입
- `instance == null` 조건을 동시에 통과 → 싱글턴이 아닌 여러 객체 생성
- 내부 설정 값 충돌, NullPointerException 등 서비스 이상 발생

------

## ✅ 해결 방법: Double-Checked Locking + volatile

```java
public class ConfigManager {
    private static volatile ConfigManager instance;

    public static ConfigManager getInstance() {
        if (instance == null) {                         // 1차 체크
            synchronized (ConfigManager.class) {
                if (instance == null) {                 // 2차 체크
                    instance = new ConfigManager();
                }
            }
        }
        return instance;
    }
}
```

### 🔍 핵심 포인트

- `volatile`: 초기화 이전 상태가 다른 스레드에 보이지 않도록 보장
- `2단 체크`: 성능 고려. 객체 초기화 이후에는 lock 없이 빠르게 반환