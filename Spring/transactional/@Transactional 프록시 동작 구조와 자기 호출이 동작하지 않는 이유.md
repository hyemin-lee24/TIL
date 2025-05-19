# @Transactional 프록시 동작 구조와 자기 호출이 동작하지 않는 이유

### ✅ 1. `@Transactional`은 어떻게 작동하는가?

### 🔹 선언적 트랜잭션

- `@Transactional`은 **선언적 트랜잭션 관리 방식**이다.
- 스프링 컨테이너가 해당 빈을 감싸는 **프록시 객체(proxy object)** 를 만들어 트랜잭션을 관리한다.

---

### ✅ 2. 프록시 구조를 그림으로 보면

```
(외부 호출)
Client Code
   ↓
[Proxy(OrderService)]  ← 트랜잭션 시작/종료를 처리하는 진짜 주체
   ↓
Real OrderService
```

즉, 호출이 **프록시를 경유**해야 트랜잭션이 동작한다. 프록시는 다음과 같은 책임을 갖는다:

```java
invoke(targetMethod) {
    beginTransaction();
    try {
        Object result = method.invoke(target, args);
        commit();
        return result;
    } catch (Exception e) {
        rollback();
        throw e;
    }
}
```

---

### ✅ 3. 자기 호출(self-invocation)이 동작하지 않는 구조적 이유

### 🔹 내부 호출이란?

```java
public class OrderService {
    @Transactional
    public void order() {
        this.createInvoice(); // 내부 메서드 호출
    }

    @Transactional
    public void createInvoice() {
        // 트랜잭션 기대
    }
}
```

### 🔹 실제로는 어떻게 동작하는가?

> 여기서 중요한 건 this.createInvoice()는 프록시를 거치지 않고, OrderService 클래스의 인스턴스 내부에서 직접 메서드를 호출한다는 점이다.
>

### 호출 경로 비교

| 외부 호출                                      | 내부 호출                                     |
| ---------------------------------------------- | --------------------------------------------- |
| `Proxy.placeOrder()` → `Proxy.createInvoice()` | `Proxy.placeOrder()` → `this.createInvoice()` |
| ✅ 트랜잭션 적용                                | ❌ 트랜잭션 적용 안 됨                         |

### 🔹 왜 안 되는가?

- 트랜잭션은 **프록시가 인터셉트**할 때만 적용됨.
- `this.createInvoice()`는 **프록시를 우회한 직접 호출**이므로 AOP advice (트랜잭션 로직)를 끼워넣을 기회가 없다.

### 🔹 비유

> 프록시는 일종의 “감시자” 같은 존재인데, 호출이 감시자를 지나가지 않으면 어떤 보호 조치도 받을 수 없다.
>

---

### ✅ 4. 어떤 시점에 프록시가 생성되는가?

### 🔹 빈 등록 과정에서 프록시 적용

- `@ComponentScan`이나 `@Bean`으로 등록된 클래스 중,
- `@Transactional`이 붙은 경우 `AutoProxyCreator`가 동작.
- 프록시로 감싼 객체가 빈으로 등록됨.

즉,

```java
OrderService orderService = applicationContext.getBean(OrderService.class);
```

이 `orderService`는 실제로는 `OrderService$$EnhancerBySpringCGLIB` 같은 프록시 클래스의 인스턴스임.

---

### ✅ 5. 정리: 핵심 원인

| 항목               | 설명                                                         |
| ------------------ | ------------------------------------------------------------ |
| 트랜잭션 처리 위치 | AOP 프록시 객체가 트랜잭션 로직 처리                         |
| 자기 호출 시 문제  | 프록시를 거치지 않아 트랜잭션 advice가 적용되지 않음         |
| 해결 방법          | 해당 메서드를 다른 빈으로 분리하거나, 프록시를 직접 참조     |
| 근본 원인          | 프록시 객체는 자신 내부 메서드 호출을 가로채지 못함 (Java 언어의 한계) |

---

### ✅ 실무 팁

- 자기 호출이 필요한 경우:
    1. 트랜잭션 분리 대상 메서드를 **다른 컴포넌트**로 옮긴다.
    2. 혹은 `@TransactionalEventListener`, `ApplicationEventPublisher` 같은 구조로 우회.