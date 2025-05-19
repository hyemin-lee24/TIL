# DB별 Mapper 관리 전략 (프로파일 기반 설계)

### 🧩 주제: MyBatis 기반 RDB 및 Ignite 환경에서 프로파일로 Mapper 충돌 없이 깔끔하게 관리하기

---

### ✅ 배경

기존 시스템은 Oracle, Tibero 등의 RDB와 MyBatis Mapper 인터페이스 기반 자동 구현체를 사용했습니다.

최근 Apache Ignite를 도입하며, Ignite는 MyBatis와 연동 가능하지만, 프로젝트 특성상 **IgniteI에 맞춘 커스텀 구현체를 직접 작성해야 하는 상황**이었습니다.

---

### 🧨 문제점

- 동일한 Mapper 인터페이스를 두고, RDB는 MyBatis가 자동 구현체를 생성하지만 Ignite는 직접 구현체(`UserMapperImpl` 등)를 만들어 Bean으로 등록해야 했습니다.
- 이때 Spring DI 과정에서 **`UserMapper` 타입 빈 충돌 또는 없음 오류**가 발생했습니다.
예시 : UserMapper

---

### 🎯 목표

- 공통 Mapper 인터페이스(`UserMapper`) 유지
- RDB는 MyBatis 자동 구현체 사용
- Ignite는 커스텀 구현체 수동 등록
- 프로파일(`rdb`, `ignite`)에 따라 충돌 없이 Bean 주입
- 중복 Mapper 인터페이스 생성 없이 간결하고 확장성 있는 구조

---

### 🛠️ 해결 전략

### 1. 공통 인터페이스 유지

```java
@Mapper
public interface UserMapper {
    int selectUserCnt();
    ...
}
```

- RDB 환경에서 MyBatis가 자동으로 구현체 생성

---

### 2. Ignite 전용 구현체 수동 등록

```java
@Repository
@Profile("ignite")
public class IgniteUserMapper implements UserMapper {
    @Override
    public int selectUserCnt() {
        // Ignite 기반 코드
    }
}
```

- Ignite 특성에 맞는 직접 구현체 등록 (MyBatis 미사용 가능성 있음)

---

### 3. 프로파일 별 `@MapperScan` 분리 설정

```java
@Configuration
@Profile("rdb")
@MapperScan({"com.example.mapper", "com.example.mappercommon"})
public static class RdbConfig {}

@Configuration
@Profile("ignite")
@MapperScan({"com.example.mapper.ignite", "com.example.mappercommon"})
public static class IgniteConfig {}
```

- RDB는 MyBatis Mapper 인터페이스 자동 스캔
- Ignite는 커스텀 구현체 및 필요한 Mapper 스캔
- 공통 Mapper는 `mappercommon`에 분리해 중복 최소화

---

### 🧠 정리

- MyBatis는 `@MapperScan`으로 인터페이스를 빈으로 등록해야 한다
- `@Profile`로 환경에 따라 Bean 등록 범위를 분리하는 것이 핵심
- 동일 인터페이스를 공유하면서도 프로파일별 구현체 주입이 가능하도록 구조화해야 한다
- 복잡한 상속 구조 대신, Bean 스캔 범위를 명확히 나누는 것이 유지보수에 유리하다
- Ignite는 MyBatis 연동 가능하지만, 프로젝트 상황에 따라 직접 구현체를 작성하는 전략도 필요하다

---

### 🚀 결과 및 기대효과

- 프로파일 기반 자동 Mapper 구현체 주입으로 환경 전환 간편
- 중복 코드 없이 확장 가능하며, DB별 특화 구현체 관리 용이
- 개발 생산성 및 유지보수성 향상
- 다른 DB나 신규 스토리지 도입 시에도 유연하게 대응 가능