# Oracle 접속 방식에 따라 SYS_CONTEXT('USERENV', 'IP_ADDRESS')가 NULL일 수 있다

### 🗓️ 상황

운영 환경에서 사용자 접속 IP를 추적할 일이 생겼다.

Oracle에서 제공하는 `SYS_CONTEXT('USERENV', 'IP_ADDRESS')`로 IP를 가져올 수 있다고 알고 있었는데, 예상과 달리 `NULL`이 반환되는 세션이 있어 원인 분석을 시작했다.

------

### 🔍 문제 현상

```sql
SELECT SYS_CONTEXT('USERENV', 'IP_ADDRESS') FROM dual;
```

- 일부 세션에서는 정상적으로 IP가 나왔고,
- 일부 세션에서는 `NULL`이 반환됨.

직접 접속해서 테스트해본 결과:

| 접속 방식                               | 결과                |
| --------------------------------------- | ------------------- |
| `sqlplus user/pwd` (로컬 서버에서 실행) | ❌ `NULL`            |
| `sqlplus / as sysdba` (OS 인증, 로컬)   | ❌ `NULL`            |
| `sqlplus user/pwd@host:1521/orcl`       | ✅ 실제 IP 주소 반환 |
| SQL Developer, JDBC, 애플리케이션 서버  | ✅ 실제 IP 주소 반환 |

------

### 🎯 원인 분석

- `SYS_CONTEXT('USERENV', 'IP_ADDRESS')`는 **Oracle 세션이 생성될 때 Listener를 통해 전달받은 클라이언트 IP를 저장**한다.
- 로컬에서 `sqlplus`로 접속하는 경우, **Oracle Listener를 거치지 않고 IPC(BEQ)** 방식으로 접속됨.
- 따라서 Oracle 입장에서는 **"외부 클라이언트 IP 정보가 존재하지 않음" → `NULL` 반환**.

Oracle 공식 문서에서는 다음과 같이 설명되어 있음:

> “Returns the IP address of the client machine. If the client and server are on the same machine and the connection uses IPC or BEQ, then the value may be NULL.”
>
> — [Oracle SQL Language Reference (SYS_CONTEXT)](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/SYS_CONTEXT.html)

------

### 💡 실무 적용 포인트

1. **감사 로그, 보안 로깅, 트리거 등에서 IP 추적이 필요할 경우**, 접속 방식에 따라 IP가 NULL일 수 있음을 반드시 고려해야 한다.
2. 모든 클라이언트가 반드시 **TCP 접속(TNS 접속)**을 사용하도록 제한하거나, 로컬 접속을 별도로 추적해야 한다.
3. 필요 시 Logon Trigger 등을 활용하여 IP를 테이블에 저장해 둘 수 있다.

------

### ✅ 접속 방식 강제 예시

**정상적으로 IP가 나오게 하려면:**

```bash
sqlplus user/pwd@ip:1521/orcl
```

이처럼 Listener를 거쳐 TCP 접속하면 `SYS_CONTEXT('USERENV', 'IP_ADDRESS')`에서 실제 IP가 반환된다.

------

### 🧪 유용한 쿼리

현재 세션의 IP 확인:

```sql
SELECT SYS_CONTEXT('USERENV', 'IP_ADDRESS') AS ip FROM dual;
```

전체 세션에서 접속 IP 확인:

```sql
SELECT sid, serial#, username, osuser, machine,
       SYS_CONTEXT('USERENV', 'IP_ADDRESS') AS ip
FROM v$session
WHERE username IS NOT NULL;
```

> 단, SYS_CONTEXT('USERENV', 'IP_ADDRESS')는 현재 세션의 IP만 반환하므로, 다른 세션의 IP는 v$session, v$session_connect_info 등을 참조해야 한다.

------

### ✏️ 느낀 점

- 예전에는 단순히 IP를 찍으면 되는 줄 알았지만, 접속 방식에 따라 다르게 동작할 수 있다는 걸 이번에 명확히 알게 됐다.
- 실무에서 “왜 IP 안 나와요?” 같은 문의가 오면, 접속 방식부터 확인하는 습관이 필요하다.
- Oracle 내부 구조(LIstener → Session 생성 → Context 초기화)를 어느 정도 이해하면 이런 트러블도 빠르게 해결할 수 있다.

------

### 📌 참고 링크

- [Oracle 공식 SYS_CONTEXT 문서](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/SYS_CONTEXT.html)
- [Oracle Forums - SYS_CONTEXT IP_ADDRESS NULL](https://forums.oracle.com/ords/apexds/post/sys-context-userenv-ip-address-and-sys-context-userenv-os-u-0834)