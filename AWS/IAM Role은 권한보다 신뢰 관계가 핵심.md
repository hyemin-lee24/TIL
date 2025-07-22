# IAM Role은 권한보다 신뢰 관계가 핵심

오늘 AWS RDS Custom 실습 중 IAM Role을 다시 보게 됐다.

기존에는 Role에 부여된 권한(Permissions)만 신경 썼는데, 이번에 진짜 중요한 건 신뢰 관계(Trust Policy) 라는 걸 체감했다.

------

## IAM Role 구성 요소 요약

1. **Permissions (권한)**

   → 이 Role이 **무슨 작업을 할 수 있는지**

2. **Trust Relationship (신뢰 관계)**

   → 이 Role을 **누가 사용할 수 있는지**

------

## 예시: EC2에서 사용할 수 있는 IAM Role의 신뢰 정책

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

- 위 설정이 없으면 EC2는 Role을 사용할 수 없다.
- 즉, 아무리 S3나 RDS에 접근 가능한 권한이 있어도 **신뢰 정책에 EC2가 명시되지 않으면 무용지물**이다.

------

## 깨달은 점

- **Permissions**만 봐선 안 된다.
- **Trust Relationship이 있어야 서비스가 Role을 "Assume"할 수 있다.**
- IAM Role은 결국 이렇게 작동한다:

```
[EC2] --(AssumeRole)--> [IAM Role] --(Permissions)--> [AWS 리소스 접근]
```