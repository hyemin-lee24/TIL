# Apache Ignite 도입 및 운영 노트 (TIL)

> 기존 RDB 기반 시스템에 Apache Ignite를 도입하면서 학습한 내용을 정리하였습니다.  
> 주요 내용은 설정, 영속성, 클러스터 구성, 보안, 운영 팁 등입니다.

---

## 🔌 DBeaver 연결 주의 사항

- Ignite 버전에 맞는 JDBC Driver 필요
- 기본 SQL 접근 가능 (단, 관계형 무결성 제약은 미지원)

---

## 🧠 메모리 기반 아키텍처

- 기본은 In-Memory (RAM 상주)
- 별도 설정 없으면 **재기동 시 데이터 소실**
- Disk 기반 저장 활성화 가능 → `Persistence` 설정 필요

### 🔧 Persistence 설정 예시

```xml
<property name="dataRegionConfigurations">
  <list>
    <bean class="org.apache.ignite.configuration.DataRegionConfiguration">
      <property name="name" value="persistence_region" />
      <property name="initialSize" value="#{100L * 1024 * 1024}" />
      <property name="maxSize" value="#{100L * 1024 * 1024}" />
      <property name="persistenceEnabled" value="true" />
    </bean>
  </list>
</property>
```

- 위 설정 후, `CacheConfiguration`에서 해당 Region을 사용하도록 지정 필요

  

------

## 🔑 무결성 제약

- Ignite는 FK 등의 **관계형 무결성 제약 미지원**

- 무결성은 **애플리케이션 로직 또는 별도 체크**로 보장

  ```java
  ignite.cache("table").containsKey(seq);
  ```

- `IgniteAtomicSequence`로 유사한 auto-increment 구현 가능

------

## ⚙️ 운영 환경 권장 설정

### 1. 클러스터 노드 구성

| 노드 수    | 설명                        |
| ---------- | --------------------------- |
| 1개        | 개발/테스트 용도            |
| 2개        | 고가용성 불가               |
| ✅ 3개 이상 | 장애 복구, 파티션 복제 가능 |

- **홀수 개수** 추천 (Quorum 안정성 확보)

### 2. 파티션 & 백업

```java
new RendezvousAffinityFunction(false, 1024); // 기본 파티션 수
```

- 백업 수 최소 `1` 이상 설정 권장 → 노드 장애 시 데이터 보존

### 3. Persistence 옵션

- 운영 환경에서는 영속성 활성화 권장

- 설정:

  ```java
  DataStorageConfiguration.setPersistenceEnabled(true);
  ```

------

## 🏗 클러스터 고려 사항

| 항목                 | 권장 설정                                            |
| -------------------- | ---------------------------------------------------- |
| Discovery SPI        | `TcpDiscoverySpi` + 정적 IP 목록                     |
| 클라이언트/서버 모드 | 서버 = 데이터 저장 / 클라이언트 = 경량 접근          |
| Baseline Topology    | Persistence 사용 시 필수 구성                        |
| Node Memory          | 최소 4~8GB 이상                                      |
| GC 설정              | G1GC 또는 ZGC (JDK 11 이상) 사용                     |
| Metrics & Logging    | JMX / `ignite-log4j2.xml` 통한 로그 및 모니터링 설정 |

------

## 🔐 보안 설정 (운영환경 필수)

- 기본은 인증/암호화 OFF
- SSL/TLS, 인증(Authentication), 권한 설정 필수
- 서버-클라이언트 간 **보안 채널 구성 필요**

------