# Apache Ignite 클러스터 구성 및 XML 설정 제어

### 🔧 1. Ignite 설정을 코드에서 XML로 분리하는 이유

- **벤더 락인 회피**와 **유연한 운영 관리**를 위해 Java 코드에 하드코딩된 Ignite 설정을 XML 외부 파일로 분리하는 것이 좋다.
- 특히 IP/Port와 같은 변경 가능성이 있는 정보는 외부에서 주입할 수 있도록 구성하는 것이 이상적.

---

### 🗂 2. XML 기반 Ignite 설정 (ignite-config.xml)

- Ignite 설정은 `Ignition.loadSpringBean(xml 파일 경로, "ignite.cfg")` 방식으로 불러올 수 있다.
- 이때 XML은 Spring Bean 형식으로 작성해야 하며, `ignite-spring` 모듈이 classpath에 있어야 한다.
- 잘못된 XML 구조나 네임스페이스 오류, 또는 schema 불일치 시 `SAXParseException` 발생.

---

### 📍 3. IP 주소 목록 동적 수정

- Java 코드에서 `DocumentBuilder`를 사용해 `ignite-config.xml`의 `<property name="addresses">` 하위 `<list>` 값을 동적으로 수정 가능.
- 수정 후 XML을 다시 저장하여 Ignite 시작 전에 반영할 수 있다.
- 예시 코드는 기존 `<list>` 제거 후 새 `<list>` 엘리먼트를 생성해 IP들을 갱신.

---

### 🔄 4. 클러스터 IP 구성에 따른 동작

- 노드 A가 IP 2개(`ip1`,`ip2`)를 등록하고, 노드 B가 자신의 IP 하나만 등록한 경우:
    - 노드 A는 연결을 시도할 수 있지만, 노드 B는 클러스터를 완전히 인식하지 못할 수 있다.
    - **모든 노드가 서로를 인식할 수 있도록 IP 목록은 동일하게 구성하는 것이 안정적**이다.
- IP를 나중에 추가해도 클러스터 합류는 가능하지만, 클러스터 안정성과 발견 속도에 영향이 있음.

---

### ⚠️ 5. BaselineTopology 충돌 문제

- Ignite는 영속성 모드일 때 BaselineTopology(BLT)를 유지한다.
- 클러스터 구성 변경 후 기존 데이터가 있는 상태에서 Ignite를 재기동하면 아래 오류 발생 가능:
    
    ```
    Failed to start SPI: TcpDiscoverySpi
    BaselineTopology of joining node ... is not compatible ...
    ```
    
- 🔧 **해결 방법**:
    - `ignite-data/store`, `wal`, `archive` 등의 디렉터리를 삭제하여 노드를 클린 상태로 초기화
    - 이후 클러스터에 재참여

---

### 🧠 요약

| 항목 | 요약 |
| --- | --- |
| 설정 분리 | Java → XML 분리로 유연한 운영 가능 |
| IP 설정 | `ipFinder`의 `addresses`는 모든 노드에서 일치해야 클러스터가 안정됨 |
| XML 수정 | Java에서 DOM 방식으로 XML 직접 수정 가능 |
| BLT 충돌 | 기존 데이터 삭제 후 재참여 필요 |
| 외부 파일 | `conf/` 같은 폴더에 XML 두고 경로 직접 지정 가능 |

### 🔧 1. Ignite 설정을 코드에서 XML로 분리하는 이유

- **벤더 락인 회피**와 **유연한 운영 관리**를 위해 Java 코드에 하드코딩된 Ignite 설정을 XML 외부 파일로 분리하는 것이 좋다.
- 특히 IP/Port와 같은 변경 가능성이 있는 정보는 외부에서 주입할 수 있도록 구성하는 것이 이상적.

---

### 🗂 2. XML 기반 Ignite 설정 (ignite-config.xml)

- Ignite 설정은 `Ignition.loadSpringBean(xml 파일 경로, "ignite.cfg")` 방식으로 불러올 수 있다.
- 이때 XML은 Spring Bean 형식으로 작성해야 하며, `ignite-spring` 모듈이 classpath에 있어야 한다.
- 잘못된 XML 구조나 네임스페이스 오류, 또는 schema 불일치 시 `SAXParseException` 발생.

---

### 📍 3. IP 주소 목록 동적 수정

- Java 코드에서 `DocumentBuilder`를 사용해 `ignite-config.xml`의 `<property name="addresses">` 하위 `<list>` 값을 동적으로 수정 가능.
- 수정 후 XML을 다시 저장하여 Ignite 시작 전에 반영할 수 있다.
- 예시 코드는 기존 `<list>` 제거 후 새 `<list>` 엘리먼트를 생성해 IP들을 갱신.

---

### 🔄 4. 클러스터 IP 구성에 따른 동작

- 노드 A가 IP 2개(`ip1`,`ip2`)를 등록하고, 노드 B가 자신의 IP 하나만 등록한 경우:
    - 노드 A는 연결을 시도할 수 있지만, 노드 B는 클러스터를 완전히 인식하지 못할 수 있다.
    - **모든 노드가 서로를 인식할 수 있도록 IP 목록은 동일하게 구성하는 것이 안정적**이다.
- IP를 나중에 추가해도 클러스터 합류는 가능하지만, 클러스터 안정성과 발견 속도에 영향이 있음.

---

### ⚠️ 5. BaselineTopology 충돌 문제

- Ignite는 영속성 모드일 때 BaselineTopology(BLT)를 유지한다.
- 클러스터 구성 변경 후 기존 데이터가 있는 상태에서 Ignite를 재기동하면 아래 오류 발생 가능:
  
    ```
    Failed to start SPI: TcpDiscoverySpi
    BaselineTopology of joining node ... is not compatible ...
    ```
    
- 🔧 **해결 방법**:
    - `ignite-data/store`, `wal`, `archive` 등의 디렉터리를 삭제하여 노드를 클린 상태로 초기화
    - 이후 클러스터에 재참여

---

### 🧠 요약

| 항목      | 요약                                                         |
| --------- | ------------------------------------------------------------ |
| 설정 분리 | Java → XML 분리로 유연한 운영 가능                           |
| IP 설정   | `ipFinder`의 `addresses`는 모든 노드에서 일치해야 클러스터가 안정됨 |
| XML 수정  | Java에서 DOM 방식으로 XML 직접 수정 가능                     |
| BLT 충돌  | 기존 데이터 삭제 후 재참여 필요                              |
| 외부 파일 | `conf/` 같은 폴더에 XML 두고 경로 직접 지정 가능             |