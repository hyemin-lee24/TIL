# Apache Ignite 트랜잭션 처리

# 명시적 트랜잭션 처리

- Ignite에서의 트랜잭션은 `IgniteTransactions.txStart()` 토크를 통해 명시적으로 시작 / 종료 필요
- Ignite의 SQL은 트랜잭션이 적용되지 않으므로, Java API 방식의 사용이 필요

### MyBatis 가능성 및 해당 해제

- MyBatis는 Ignite와 연동시 모든 직감적 조정은 트랜잭션이 보장되지 않음
- SELECT와 일반적인 조회에서는 MyBatis를 계속 활용 가능

---

## Ignite SQL 구조 바로 변환 하기 힘들 때 대안

### ✔️ JOIN 구문 대안 방식

```
SELECT M.*, subDomain, subType
FROM target_table M
LEFT OUTER JOIN (
  SELECT id, domain AS subDomain, type AS subTyoe
  FROM target_table
  WHERE sub_seq = #{subSeq}
) S ON M.id = S.id
WHERE M.id = #{id} AND M.sub_seq = #{mainSeq}
```

- Ignite SQL에서의 JOIN은 해당 크로스에 대한 크롬프 수행이 빠른 것은 아님
- 대안 방식: 2 번의 `ScanQuery` + Java로 조인 치료 + Model 매핑

### ScanQuery 방식

```
ScanQuery<BinaryObject, BinaryObject> query = new ScanQuery<>((k, v) ->
    id.equals(k.field("id")) && Objects.equals(v.field("sub_seq"), subSeq)
);
QueryCursor<Entry<BinaryObject, BinaryObject>> cursor = cache.query(query);
```

- `withKeepBinary()`를 통해 `BinaryObject`로 객체 조회
- getKey()/getValue()로 검색 결과 찾아 ㅡModel 매핑 진행

---

## Model 매핑 개발 방식

### ⚠️ QueryCursor.getAll() vs iterator()

- Ignite QueryCursor는 stream 구조이며, `getAll()` 및 `iterator()` 가 둘 다 사용이 가능지 않음
- 하나만 사용 해야 예외를 발생시키지 않음

### 매핑 자동화 모습

- Reflection 가능 (ex: `Model.class.getDeclaredFields()` 바탕)
- Ignite가 BinaryObject을 사용하기 때 다양한 형식 (int, String, enum)에 대해 처리 필요
- 배열 순서가 getAll()로 드린 값의 배열 순서와 호출적으로 매칭이 맞아야 함