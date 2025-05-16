# @Transactional 프록시 기반 동작

### ✅ 1. `@Transactional` 기본 개념

### 🔹 역할

- 선언한 메서드(또는 클래스) 실행 시 **트랜잭션을 시작**하고, 정상 종료 시 **커밋**, 예외 발생 시 **롤백**함.
- DB의 **데이터 정합성 보장**에 핵심적인 기능.

### 🔹 사용 위치

- 클래스: 클래스 내 모든 public 메서드에 적용
- 메서드: 해당 메서드에만 적용

```java
@Service
@Transactional // 클래스 레벨 적용
public class UserService {
    public void createUser(User user) {
        userRepository.save(user);
    }
}
```

```java
@Service
public class UserService {
    @Transactional // 메서드 레벨 적용
    public void createUser(User user) {
        userRepository.save(user);
    }
}
```

### 🔹 트랜잭션 커밋/롤백 규칙

- 기본적으로 **`RuntimeException` 또는 `Error`** 발생 시 롤백.
- `Checked Exception`은 롤백하지 않음.
- 필요 시 `rollbackFor`, `noRollbackFor` 옵션 사용.

```java
@Transactional(rollbackFor = FileNotFoundException.class)
public void doSomething() throws Exception {
    // FileNotFoundException 이 발생할 경우는 rollbak 처리됨.
}
```

---

### ✅ 2. 프록시 기반 동작 원리

### 🔹 Spring AOP 기반

- `@Transactional`은 **AOP(Aspect-Oriented Programming)** 기반으로 작동.
- **프록시 객체**를 생성해 메서드 호출 시 트랜잭션 처리를 끼워넣음.

### 🔹 프록시 방식

- **JDK Dynamic Proxy**: 인터페이스 기반
- **CGLIB Proxy**: 클래스 기반 (인터페이스 없을 경우)

> 👉 프록시가 생성되므로 실제 객체가 아닌, 프록시 객체를 통해서만 트랜잭션이 적용됨.
>

### 🔹 자기 호출(self-invocation) 문제

- 클래스 내부에서 **`this.메서드()` 형태로 호출하면 프록시를 우회**하게 되어 트랜잭션이 적용되지 않음.

```java
@Service
public class OrderService {
    @Transactional
    public void order() {
        // 트랜잭션 시작
        createInvoice(); // ← 내부 호출 → 트랜잭션 적용 안 됨!
    }

    @Transactional
    public void createInvoice() {
        // 여기 트랜잭션 동작 안 함
    }
}
```

### 🔹 해결 방법

1. **메서드를 다른 빈으로 분리**
2. **AopContext 사용** (비추천: 복잡도 증가)

```java
@Service
@EnableAspectJAutoProxy(exposeProxy = true)
public class OrderService {
    @Transactional
    public void order() {
        ((OrderService) AopContext.currentProxy()).createInvoice(); // 트랜잭션 정상 적용
    }

    @Transactional
    public void createInvoice() {
        // 트랜잭션 적용됨
    }
}
```

---

### 🧠 정리

| 항목           | 설명                             |
| -------------- | -------------------------------- |
| 적용 대상      | 클래스, 메서드                   |
| 동작 방식      | Spring AOP 프록시 (JDK or CGLIB) |
| 주의사항       | 내부 호출 시 트랜잭션 미적용     |
| 기본 롤백 조건 | RuntimeException, Error          |
| 해결책         | 분리된 빈 사용 또는 AopContext   |