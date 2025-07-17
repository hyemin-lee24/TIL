# Spring Boot 앱을 ALB 뒤에 둘 때 TLS 구조 고민 정리

온프레미스에서 운영 중이던 Spring Boot + Thymeleaf 웹앱을 AWS 환경으로 옮기면서,

Application Load Balancer(ALB)를 앞단에 둘 경우의 HTTPS 처리 방식에 대해 정리해봤다.

------

## 현재 상황

- 기존 앱은 온프레미스에서 Spring Boot 내장 HTTPS 기능을 사용하고 있음 (self-signed 인증서)
- 인증서는 내부 CA 또는 수동 발급된 사설 인증서를 사용 중
- AWS로 이전하면서 Private Subnet에 EC2를 두고, 외부 접근은 ALB를 통해 처리하려고 계획 중

------

## 고민한 내용

### 1. ALB 뒤에 있는 Spring Boot는 HTTPS를 유지해야 할까?

- ALB가 TLS 종료를 맡고, 내부는 HTTP로 통신하면 구성은 단순
- 하지만 사내 정책 또는 보안 요구사항상 End-to-End TLS가 필요할 수도 있음

### 2. ALB는 백엔드(Spring Boot)의 인증서를 검증할까?

- AWS 공식 문서를 통해 확인 결과:

  > ALB는 Target Group(EC2 등)의 인증서를 검증하지 않는다

- 즉, self-signed 인증서든 만료된 인증서든 관계 없이 ALB는 백엔드와 TLS 연결만 수립한다

### 3. Spring Boot 앱에서 TLS를 유지하면서도 ALB 뒤에 두려면?

- `server.ssl.*` 설정은 그대로 두되, ALB의 Target Group을 **HTTPS**로 설정해야 함
- 클라이언트 인증서(mTLS)가 필요한 경우, ALB는 이를 백엔드로 전달할 수 없기 때문에 별도 프록시(Nginx 등)가 필요함

------

## 정리

- ALB에 인증서를 등록하는 이유는 “백엔드 인증서 검증용”이 아니라 **외부 클라이언트를 위한 TLS 종단용**
- ALB는 Spring Boot 서버의 인증서를 **검증하지 않음**
- ForwardedHeaderFilter를 추가해야 Spring Boot 앱이 ALB 뒤에서도 HTTPS 정보를 제대로 인식함
- End-to-End TLS를 구성할 경우에도 Spring Boot에 별도의 인증서 설정이 필요함

------

## 다음 단계

- 실제 구성은 아직 결정하지 않았지만, 구성 방식(SSL 종료 vs 종단간 TLS)을 사내 보안 정책과 비교해서 결정할 예정
- 필요하다면 ALB 뒤에 인증서 검증 가능한 프록시(Nginx, Envoy) 등을 둘지 검토할 것

------