# JSON 통신 프로토콜의 상위/하위 호환성 설계 with Jackson

## 📌 주제

Java 프로세스 간 JSON 기반 통신에서 Jackson 직렬화/역직렬화를 사용할 때

프로토콜 스펙 변경(필드 추가, enum 확장 등)에 따른 **상위/하위 호환성**을 유지하는 설계 방법 정리.

---

## 배경

두 Java 서비스 간 통신을 JSON + Jackson으로 처리 중,

한쪽에서 프로토콜 필드(`type`)를 새로 추가했을 때 다음 문제가 발생:

- 구버전은 알 수 없는 필드를 포함한 JSON을 받을 경우 예외 발생 가능
- 신버전은 누락된 필드를 받을 경우 `null` 처리로 인한 NPE 위험 존재

---

## ✅ 해결 방법 요약

### 🔽 하위 호환성 (Backward Compatibility)

*신버전 → 구버전 JSON 메시지 수신 시 안정적으로 처리*

- 새로 추가되는 필드에 **기본값 설정**
  
    ```java
    private String type = "DEFAULT";
    ```
    
- `getter`에서 null-safe 처리
  
    ```java
    public String getType() {
        return type != null ? type : "DEFAULT";
    }
    ```
    
- **역직렬화 후 객체 상태 단위 테스트 작성**
  
    → 필수 필드 누락 시 서비스 예외 방지
    

---

### 🔼 상위 호환성 (Forward Compatibility)

*구버전 → 신버전 JSON 메시지 수신 시 무시 처리*

- Jackson에 다음 어노테이션 적용
  
    ```java
    @JsonIgnoreProperties(ignoreUnknown = true)
    public class CertificateIssueResponse { ... }
    ```
    
- 알 수 없는 필드 무시하도록 설정
  
    → 프로토콜 확장 시에도 구버전과 충돌 없음
    

---

## 🧪 테스트 전략

- JUnit 기반 단위 테스트로 JSON 누락 필드/추가 필드 케이스 커버
- `ObjectMapper.readValue()` 수행 후 필드 상태 검증

---

## ✍️

- Jackson 기반 시스템도 설계만 잘 하면 **충분히 안정적인 프로토콜 호환성 유지가 가능**
- 프로토콜 스펙은 실제 통신 대상 버전도 고려해 **테스트/검증 주기를 반드시 포함**할 것
- 추후에는 **Protobuf 또는 Avro 같은 스키마 기반 직렬화 포맷**도 도입 검토 가치 있음

---

## 🏷 키워드

`Jackson` `JSON` `Protocol Design` `Backward Compatibility` `Forward Compatibility`

`@JsonIgnoreProperties` `null-safe getter` `직렬화 호환성 설계`