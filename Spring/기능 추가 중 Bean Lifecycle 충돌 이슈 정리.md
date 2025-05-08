# 기능 추가 중 Bean Lifecycle 충돌 이슈 정리

### 🏷️ 태그

```
Spring`, `Bean`, `Lifecycle`, `의존성 주입`, `초기화 순서
```

------

### 🎯 상황 요약

신규 모듈 기능 추가 도중, 새로 등록한 `@Component` Bean 내부에서 기존 Bean을 의존하게 되었는데,

초기화 순서가 꼬여서 **NPE 발생** → 정확히는 의존성 주입은 되었지만 초기화 메서드가 먼저 실행됨.

------

### 🔍 문제 분석

### 현상

- `BeanA` → `BeanB`를 주입받음

- `BeanB`는 내부에서 초기화 시 외부 설정 파일을 읽고 캐싱

- 그런데 `BeanA`의 `@PostConstruct`가 `BeanB` 초기화보다 먼저 실행됨

  → `BeanB.getConfig()`에서 `null` 반환 → NPE

### 원인

- `@PostConstruct`는 **의존성 주입 직후에 실행**,

  하지만 의존 대상의 내부 상태가 아직 준비되지 않았을 수 있음

- 의존 대상이 `@PostConstruct` 에 복잡한 로직을 넣고 있었음

------

### ⚠️ 실수 포인트

- Bean 간 의존 관계를 구성할 때, **단순 DI는 성공해도 초기화 시점은 따로 관리해줘야 함**
- 여러 Bean이 서로 초기화 순서를 암묵적으로 기대하고 있었음 (🙅‍♂️ 안 좋은 설계)

------

### ✅ 해결 방법

1. 의존 Bean의 초기화 시점을 명시적으로 보장하도록 수정
   - `SmartInitializingSingleton` 또는 `ApplicationListener<ContextRefreshedEvent>` 사용
   - `@DependsOn("beanB")`로 순서 지정 (비추지만 급한 대응으로 가능)
2. 초기화 로직 분리
   - `init()` 메서드를 별도로 분리해 필요 시점에 명시적으로 호출
   - 혹은 **Lazy Initialization** 적용
3. 설계 리팩토링
   - Bean 간 순환 초기화 의존을 최소화
   - 캐싱/초기화 역할은 `@Configuration` + `@Bean(initMethod = "init")` 구조로 분리

------

### 💡 정리하며

Bean의 생성-주입-초기화는 **한 줄처럼 보이지만 순서에 따라 심각한 이슈 발생 가능**.

특히 **의존 대상 Bean이 무거운 초기화 로직을 가진 경우**, `@PostConstruct` 신뢰하지 말고 명시적인 흐름 관리가 필요함.