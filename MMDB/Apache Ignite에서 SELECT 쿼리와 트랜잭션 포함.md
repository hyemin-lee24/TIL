# Apache Ignite에서 SELECT 쿼리와 트랜잭션: 포함해야 할까?

Apache Ignite에서 `SELECT`(조회)는 트랜잭션에 **반드시 포함될 필요는 없지만**, 

특정 상황에서는 **트랜잭션에 포함시키는 것이 데이터 일관성과 동시성 측면에서 중요**하다.

---

## SELECT가 트랜잭션 대상이 아닌 이유

- Ignite의 트랜잭션은 주로 `put`, `remove`, `invoke` 등 **쓰기 작업**에 대해 ACID를 보장.
- `get` 혹은 `ScanQuery`/`SqlFieldsQuery` 같은 조회는 기본적으로 **트랜잭션 읽기 일관성**을 보장하지 않음.
- 따라서 SELECT 결과가 중요한 **조건 기반 로직에 영향을 준다면**, 트랜잭션 범위 내에서 실행하는 것이 안전함.

---

## 트랜잭션에 SELECT를 포함해야 하는 경우 예시

### 예시 1: 중복 처리 방지 (타겟 상태 변경 예시)

```java
try (Transaction tx = ignite.transactions().txStart()) {
    IgniteCache<AgentKey, Agent> cache = ignite.cache("target");

    Key key = new Key(id, subSeq);
    Target target = cache.get(key);

    // 상태가 이미 삭제된 경우 처리 중단
    if ("DELETED".equals(target.getStatus())) {
        tx.rollback(); // optional
        return;
    }

    agent.setStatus("DELETED");
    cache.put(key, agent);

    tx.commit();
}

```

 트랜잭션 밖에서 조회하고 안에서 수정하면 **중간에 상태가 바뀔 수 있어서** 안전하지 않음.

---

### 예시 2: 조건부 삭제

```java
try (Transaction tx = ignite.transactions().txStart()) {
    IgniteCache<String, MyData> cache = ignite.cache("target_data");

    MyData data = cache.get("data1");

    if (data.getCount() < 5) {
        cache.remove("target1");
    }

    tx.commit();
}
```

 `get()`으로 읽고 바로 조건 판단 후 삭제하는 작업은 **트랜잭션 안에서 일괄 수행되어야 안정성 확보**됨.

---

## SELECT를 트랜잭션에 포함하지 않아도 되는 경우

### 예시 3: 단순 조회 (통계/모니터링)

```java
IgniteCache<String, Agent> cache = ignite.cache("target");

long count = cache.query(new ScanQuery<>(
    (k, v) -> "ACTIVE".equals(v.getStatus()))
).getAll().size();
```

 단순 카운트나 모니터링용 조회는 **트랜잭션과 무관**하게 수행해도 무방.

---

## SELECT 트랜잭션 포함 여부 판단 기준

| 상황 | 트랜잭션 포함 권장 여부 | 이유 |
| --- | --- | --- |
| 조회 → 조건 판단 → 쓰기 | ✅ 포함해야 안전 | 조건이 변경되면 처리 오류 발생 가능 |
| 두 값 동기 조회 및 비교 | ✅ 포함 권장 | 참조 시점 불일치 가능성 있음 |
| 단순 통계/모니터링 조회 | ❌ 포함 불필요 | 일관성이 중요한 영역이 아님 |
| 조회만 수행 (읽기 전용 API) | ❌ 포함 불필요 | 트랜잭션 오버헤드만 발생 |

---

> Ignite에서 SELECT는 기본적으로 트랜잭션 보호 대상이 아니지만, 조회 결과가 후속 로직의 정확성에 영향을 준다면 트랜잭션에 포함하는 것이 필수적이다.
>