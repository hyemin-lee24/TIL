# 트랜잭션 전파 (Propagation)

### 1. 트랜잭션 전파란?

- 하나의 트랜잭션 안에서 **다른 트랜잭션을 호출할 때, 어떻게 처리할 것인지 결정하는 정책**이다.
- `@Transactional(propagation = Propagation.XXX)` 형식으로 지정.

### + 트랜잭션

- 물리 트랜잭션: 실제 데이터베이스에 적용되는 트랜잭션으로, 커넥션을 통해 커밋/롤백하는 단위
- 논리 트랜잭션: 스프링이 트랜잭션 매니저를 통해 트랜잭션을 처리하는 단위

---

### 2. 주요 전파 속성 요약

| Propagation           | 설명                                                         |
| --------------------- | ------------------------------------------------------------ |
| **REQUIRED** (기본값) | 기존 트랜잭션이 있으면 참여, 없으면 새로 생성                |
| **REQUIRES_NEW**      | 기존 트랜잭션 **중단(Pause)**, 새 트랜잭션 시작. 기존 트랜잭션과 **독립적** |
| **NESTED**            | 기존 트랜잭션 안에서 **Savepoint**를 만들어 부분 롤백 가능. JDBC 지원 필요 |

---

### 3. 전파 속성 별 예시

### ✅ REQUIRED: 부모 트랜잭션에 참여

```java
@Service
public class OrderService {
    @Transactional
    public void order() {
        paymentService.pay();   // 같은 트랜잭션 참여
        deliveryService.ship(); // 같은 트랜잭션 참여
    }
}

@Service
public class PaymentService {
    @Transactional(propagation = Propagation.REQUIRED)
    public void pay() {
        // 기존 트랜잭션이 존재하므로 여기에 참여
    }
}
```

> 💡 하나라도 실패하면 전체 트랜잭션 롤백됨.
>

---

### ✅ REQUIRES_NEW: 별도 트랜잭션 시작

```java
@Service
public class OrderService {
    @Transactional
    public void order() {
        paymentService.pay(); // 별도 트랜잭션
        throw new RuntimeException("Fail"); // 외부 트랜잭션 롤백
    }
}

@Service
public class PaymentService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void pay() {
        // 별도 트랜잭션으로 커밋됨
    }
}
```

> 💡 pay()는 커밋되고, order()만 롤백됨 → 부분 성공 가능.
>

---

### ✅ NESTED: Savepoint로 부분 롤백

```java
@Service
public class OrderService {
    @Transactional
    public void placeOrder() {
        try {
            paymentService.pay(); // 실패해도 order 전체는 롤백 안 됨
        } catch (Exception e) {
            // swallow
        }
    }
}

@Service
public class PaymentService {
    @Transactional(propagation = Propagation.NESTED)
    public void pay() {
        throw new RuntimeException("결제 실패");
    }
}

```

> 💡 pay()는 내부적으로 savepoint → rollback,
>
>
> 외부 트랜잭션은 그대로 진행됨.
>
- 단, **JDBC 드라이버가 savepoint를 지원**해야 함.

---

### ✅ 4. 실전 예제: 서비스 계층 트랜잭션 전파 실험

### 시나리오

- 하나의 주문 처리에서 **결제**와 **포인트 적립**을 함께 처리.
- 포인트 적립이 실패했을 때, 전체 롤백 여부 실험.

---

### 예제 1. 모두 REQUIRED (기본)

```java
@Service
public class OrderService {
    @Transactional
    public void processOrder() {
        paymentService.pay();         // 성공
        pointService.addPoints();     // 실패 (예외 발생)
    }
}

@Service
public class PointService {
    @Transactional(propagation = Propagation.REQUIRED)
    public void addPoints() {
        throw new RuntimeException("포인트 적립 실패");
    }
}
```

> 💥 addPoints() 실패 → 전체 롤백
>

---

### 예제 2. REQUIRES_NEW 사용

```java
@Service
public class OrderService {
    @Transactional
    public void processOrder() {
        paymentService.pay();         // 성공
        try {
            pointService.addPoints(); // 별도 트랜잭션
        } catch (Exception e) {
            // swallow
        }
    }
}

@Service
public class PointService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void addPoints() {
        throw new RuntimeException("포인트 적립 실패");
    }
}
```

> ✅ pay()는 커밋됨,
>
>
> ✅ `addPoints()`만 롤백됨 → **부분 성공 가능**
>

---

### 예제 3. NESTED 사용

```java
@Service
public class OrderService {
    @Transactional
    public void processOrder() {
        try {
            pointService.addPoints(); // NESTED 트랜잭션
        } catch (Exception e) {
            // swallow
        }

        orderRepository.save(...); // 정상 저장됨
    }
}

@Service
public class PointService {
    @Transactional(propagation = Propagation.NESTED)
    public void addPoints() {
        throw new RuntimeException("포인트 적립 실패");
    }
}
```

> ✅ addPoints()는 Savepoint 롤백,
>
>
> ✅ `orderRepository.save()`는 그대로 커밋됨 → **내부 분리된 롤백**
>

---

### ✅ 정리

| 전파 속성    | 트랜잭션 관계       | 실패 시 영향   | 특징           |
| ------------ | ------------------- | -------------- | -------------- |
| REQUIRED     | 부모 참여           | 전체 롤백      | 기본값         |
| REQUIRES_NEW | 독립 트랜잭션       | 별도 커밋/롤백 | 외부 영향 없음 |
| NESTED       | 부모 내부 savepoint | 내부 롤백 가능 | 드라이버 의존  |