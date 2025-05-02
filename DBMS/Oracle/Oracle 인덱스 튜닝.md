# 📌Oracle 인덱스 튜닝

## 🔎 개요

인덱스(Index)는 데이터 접근을 빠르게 하기 위한 핵심 수단이다.
하지만 잘못 설계되거나 관리되지 않은 인덱스는 오히려 성능을 저하시킬 수 있다.
Oracle 기반 시스템에서의 인덱스 설계와 튜닝 전략을 실무 관점에서 정리한다.

---

## ✅ 인덱스의 종류 및 특징

| 종류             | 설명                         | 특징                                       | 활용 예시                |
| ---------------- | ---------------------------- | ------------------------------------------ | ------------------------ |
| B-Tree 인덱스    | 기본 인덱스 구조             | 값의 **선택도**가 높을수록 효율적          | 고객ID, 주문번호         |
| Bitmap 인덱스    | 비트맵 기반 인덱스           | **선택도가 낮고**, 읽기 위주 테이블에 적합 | 성별, 회원등급           |
| 복합 인덱스      | 여러 컬럼 조합               | **WHERE절 조건에 포함된 순서** 고려 필요   | (user_id, status)        |
| 함수 기반 인덱스 | 함수 연산 결과에 대한 인덱스 | 조회 시 변형 조건이 있는 경우 사용         | UPPER(name), TRUNC(date) |

---

## ⚙️ 튜닝 핵심 전략

### 1. 복합 인덱스 설계

- WHERE 조건에 자주 쓰이는 컬럼 조합으로 구성
- **선택도 높은 컬럼을 앞에 배치**
- 예: `WHERE region = ? AND user_id = ?` → 복합 인덱스 (user_id, region)보다 (region, user_id)가 효율적
- ❗ **WHERE 절 순서**와 **인덱스 컬럼 순서**는 밀접한 관계가 있으며, **선택도가 높은 컬럼을 인덱스 앞에 배치**하는 것이 좋다.

### 2. 옵티마이저 힌트 활용

- 옵티마이저가 비효율적인 실행 계획을 선택할 경우 직접 힌트 제공

```sql
SELECT /*+ INDEX(emp emp_name_idx) */ * FROM emp WHERE name = 'John';
```

### 3. Index Skip Scan 활용

- 선행 컬럼이 없어도 후행 컬럼으로 인덱스 검색 가능 (효율은 낮음)
- 옵티마이저가 자동 판단

### 4. 통계 정보 최신화

```sql
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(ownname => 'SCOTT', tabname => 'EMP', cascade => TRUE);
END;
```

- 통계 정보가 오래되면 인덱스가 무시될 수 있음
- 옵티마이저는 통계 정보에 따라 실행 계획 수립

### 5. 불필요한 인덱스 제거

- INSERT/UPDATE/DELETE 성능에 악영향
- `USER_INDEXES`, `V$OBJECT_USAGE` 통해 사용 빈도 확인

### 6. Covering Index (커버링 인덱스)

- 인덱스에 포함된 컬럼만으로 쿼리를 처리하면 **Table Access를 생략**할 수 있음

```sql
-- Index에 (customer_id, order_status, created_at) 포함되어 있으면
SELECT order_status, created_at 
FROM orders 
WHERE customer_id = '12345';
```

- 👉 실행 계획: **Index Only Scan**

### 7. Invisible Index

- **인덱스를 비활성화해도 삭제하지 않고 성능 테스트 가능**

```sql
ALTER INDEX idx_orders_cust_status INVISIBLE;
```

- 👉 옵티마이저는 해당 인덱스를 무시함.  
- 실무에서 **안전한 인덱스 제거 전 테스트**에 유용

### 8. 인덱스 모니터링 (사용 여부 분석)

- 인덱스 사용 여부 확인: `ALTER INDEX ... MONITORING USAGE`

```sql
ALTER INDEX idx_orders_cust_status MONITORING USAGE;

-- 며칠 후
SELECT * FROM V$OBJECT_USAGE WHERE INDEX_NAME = 'IDX_ORDERS_CUST_STATUS';
```

- ✅ 실무에서 불필요한 인덱스 식별에 활용

---

## 💡 실무 사례 (예시)

### 문제:

- 주문 테이블에서 고객과 주문 상태 기준 조회 느림 (응답 1.2초)

```sql
SELECT * FROM orders 
WHERE customer_id = '12345' AND order_status = 'SHIPPED';
```

- customer_id 단일 인덱스 존재 (order_status는 없음)
- 실행 계획: Index Range Scan + Filter

### 해결:

- 복합 인덱스 추가: `CREATE INDEX idx_orders_cust_status ON orders(customer_id, order_status);`
- 실행 계획: Index Range Scan using idx_orders_cust_status
- 응답 시간: 1.2초 → 110ms (예시)