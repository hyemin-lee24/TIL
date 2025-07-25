# AWS RDS Custom for Oracle 환경 구축 체크리스트

> RDS Custom for Oracle 인스턴스를 생성하고 운영하기 위해 필요한 사전 준비사항 및 설정 가이드를 정리한 문서입니다.
> 

## 필수 조건

### KMS 설정

- **대칭 암호화 KMS 키 필요**
    - RDS Custom은 **고객이 직접 생성한 KMS 키**만 사용 가능.
    - **Amazon 관리형 KMS 키는 사용 불가**.
    - 인스턴스 생성 시 반드시 KMS 키 식별자 제공 필요.

## 네트워크 설정

### VPC 구성

- RDS Custom 인스턴스를 실행할 VPC가 사전에 구성되어 있어야 함.
- 구성 항목:
    - **서브넷 (Subnet)**
    - **라우팅 테이블 (Routing Table)**
    - **인터넷 게이트웨이 (Internet Gateway)**
    - **보안 그룹 (Security Group)**

> 워크샵 환경이 제공되는 경우, VPC와 관련된 리소스가 사전 생성되어 있을 수 있으므로 확인 필요.
> 

### 예시: 서브넷 구성

- **서브넷은 퍼블릭/프라이빗 구분하여 구성 가능**.
- RDS Custom은 EC2 기반이므로 네트워크 설정이 중요.

### 예시: 라우팅 테이블 구성

- S3, Secrets Manager 등 AWS 서비스와의 통신을 위해 외부 연결이 필요.

### 보안 그룹 설정

- RDS Custom 인스턴스를 보호할 **기본 보안 그룹** 필요.
- 인바운드/아웃바운드 트래픽 규칙 구성.
- 다음 트래픽 허용:
    - EC2 → RDS Custom
    - RDS Custom ↔ AWS 서비스 (S3, Secrets Manager 등)

## IAM 역할 및 인스턴스 프로필

- IAM 리소스 필요 `AWSRDSCustom`
- 일반적으로 RDS Custom 생성 시 자동으로 구성되지만, 확인 필요.

---

## CEV (Custom Engine Version) 생성

RDS Custom 인스턴스를 생성하려면 Oracle 설치 파일을 기반으로 CEV 생성 필요

### 1. 오라클 설치 파일 다운로드

- Oracle 공식 사이트 또는 내부 FTP 등에서 미리 설치 파일 확보

### 2. Amazon S3 버킷 준비

- CEV 생성을 위한 S3 버킷은 **같은 리전**에 있어야 함 (예: `us-east-1`)
- 버킷 구성 예시:
    - 버킷 이름: `s3bucketrdscust2025` (※ 고유 이름 사용)
    - 디렉토리: `/oracle-cev/`
    - **공개 액세스 차단: 활성화**
    - **버전 관리: 비활성화**
    - **서버 측 암호화: 비활성화**
    - **객체 ACL: 비활성화**

### 3. S3 에 설치 파일 업로드

---

## 사용자 엔진 구성 후 인스턴스 생성

1. CEV 등록이 완료되면, RDS Custom 인스턴스를 생성 가능
2. 필수 입력:
    - KMS 키 ID
    - S3 위치 (Oracle 설치 파일 경로)
    - IAM 인스턴스 프로필
    - 네트워크 설정 (서브넷, 보안 그룹 등)
3. 생성 후, EC2 인스턴스처럼 RDS Custom 인스턴스에 접근 및 구성 가능

---

## 참고 사항

- RDS Custom 인스턴스는 내부적으로 EC2에서 실행되며, 운영체제 레벨에 접근 가능
- 자동화된 백업, 모니터링 등 일부 RDS 기능은 사용 불가하거나 제한됨
- 보안 설정 및 네트워크 연결 구성에 주의

---

## 관련 문서

- AWS 공식 문서: RDS Custom for Oracle ([https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-custom.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-custom.html))