### GC 수행

```java
System.gc();
// System.gc()를 호출해도 GC가 바로 동작한다고 보장할 수 없음
// GC가 바로 동작하는 경우 모든 스레드가 중단되기 때문에 사용에 주의가 필요함
```

### GC 대상이 되는 객체

1. 모든 객체의 참조가 모두 null인 경우
2. 객체 A가 B를 참조하고, B가 A를 참조하는 형태
3. 객체 A, B를 다른 살아있는 객체가 참조하고 있지 않는 경우
4. 객체가 블록 안에서 생성되고 블록이 종료된 경우
5. 부모 객체가 null이 된 경우, 자식 객체는 자동적으로 GC 대상이 된다.
6. 객체가 Weak 참조만 가지고 있을 경우
7. 객체가 Soft 참조이지만 메모리 부족이 발생한 경우

### JAVA 객체 참조 유형 4가지

1. 강한 참조 [ Strong Reference ]

   - 객체가 Strong Reference로 참조되고 있다면 해당 객체는 GC 대상이 되지 않음
   - 객체를 참조하는 모든 Strong Reference가 해제될 때까지 객체는 메모리에 유지됨

   ```java
   Object obj = new Object();
   // 만약 GC 를 원한다면 명시적으로 null 표시를 해줘야 한다.
   obj = null;
   ```

2. 약한 참조 [ Soft Reference ]

   - 해당 객체를 참조하는 Strong Reference가 없고, 메모리가 부족한 경우 GC의 대상이 됨

   ```java
   Object obj = new Object();
   SoftReference<Object> softRef = new SoftReference<>(obj);
   obj = null;
   
   System.gc();
   
   // GC 가 여유롭다면 해시코드를 확인할 수 있다.
   System.out.println(softRef.get());
   ```

3. 취약 참조 [ Weak Reference ]

   - 해당 객체를 참조하는 Strong Reference가 없는 경우 바로 GC 대상이 됨
   - Weak Ref는 GC에 여유가 있어도 수거 대상이 될 가능성이 높음

   ```java
   Object obj = new Object();
   WeakReference<Object> weakRef = new WeakReference<>(obj);
   obj = null;
   
   System.gc();
   
   // 무조건 null 을 확인하게 된다.
   System.out.println(weakRef.get());
   ```

4. 유령 참조 [ Phantom Reference ]

   - 가장 약한 참조 유형, 객체의 Finalize() 메서드가 호출된 직후 GC에 의해 수거됨
   - 객체의 사용을 위한 참조보다 객체를 올바르게 삭제하고, 삭제 이후 작업을 조작하기 위해 사용됨

   ```java
   <https://luckydavekim.github.io/development/back-end/java/phantom-reference-in-java/>
   ```

### GC 종류

1. Serial GC
   - 일반적인 GC, GC가 싱글 스레드로 동작함 다른 GC에 비해 STW 시간이 길다.
2. Parallel GC
   - Serial GC와 동일하지만, 멀티 스레드로 동작 다른 GC에 비해 STW 시간이 줄어듬
   - ‘처리량'이 중요한 시스템에서 주로 사용
   - Full GC 수행 시 compaction 작업이 수행되기 때문에 GC 시간 자체는 많이 소요되나 일정한 멈춤 시간을 제공함
3. Parallel old GC
   - Parallel GC에서는 Young 영역에 대해서만 멀티 스레드로 동작하나, 이는 old 영역도 멀티 스레드로 수행
4. CMS GC (Concurrent Mark Sweep)
   - OLD 영역에 대한 GC / Young 영역에 대해서는 parallel gc와 같은 방식으로 수행 // STW 시간을 최소화 하기 위해 설계됨
   - 응답 시간이 중요한 시스템에서 주로 사용
5. G1 GC (Garbage First)
   - 메모리의 Young 영역과, Old 영역을 물리적으로 나누지 않고 Heap을 일정한 크기의 Region이라는 논리적 단위로 나눠 관리
   - G1 GC는 Garbage만 있는 Region을 수거하기 때문에 Garbage First라는 이름이 붙음
   - 성능적으로 우수하지만, JDK 7 버전부터 정식 지원하고 JAVA 9에서 기본 GC 방식으로 채택

### GC 모니터링

> 현재 실행중인 8884번 프로세스에 대해 1초에 한번 씩 총 10번 GC와 관련된 정보를 출력
>
> ```
> jstat -gcutil -t 8844 1000 0
> ```

- 명령어 출력 Column

  | S0   | Survivor 영역 0의 사용률(현재 용량에 대한 비율) |
  | ---- | ----------------------------------------------- |
  | S1   | Survivor 영역 1의 사용율(현재 용량에 대한 비율) |
  | E    | Eden 영역의 사용율 (현재 용량에 대한 비율)      |
  | O    | Old 영역의 사용율 (현재 용량에 대한 비율)       |
  | P    | Permanent 영역의 사용율 (현재 용량에 대한 비율) |
  | YGC  | Young 세대의 GC 이벤트 수                       |
  | YGCT | Young 세대의 GC 시간                            |
  | FGC  | Full GC 이벤트 수                               |
  | FGCT | Full GC 시간                                    |
  | GCT  | GC 총 시간                                      |

### Heap 메모리 크기 지정

- JVM의 시작 크기(-Xms)와 최대 크기(-Xmx)
  - 메모리 크기가 큰 경우
  - GC 발생 횟수 감소
  - GC 수행 시간 증가
  - 메모리 크기가 작은 경우
  - GC 발생 횟수 증가
  - GC 수행 시간 감소

| 구분               | 옵션              | 설명                             |
| ------------------ | ----------------- | -------------------------------- |
| 힙(heap) 영역 크기 | -Xms              | JVM 시작 시 힙 영역 크기         |
| 힙(heap) 영역 크기 | -Xmx              | 최대 힙 영역 크기                |
| New 영역의 크기    | -XX:NewRatio      | New 영역과 Old 영역의 비율       |
| New 영역의 크기    | -XX:NewSize       | New 영역의 크기                  |
| New 영역의 크기    | -XX:SurvivorRatio | Eden 영역과 Survivor 영역의 비율 |

### GC 모드 지정

| **GC MODE**                   | **옵션**                    | **TestModule :: -Xms64m -Xmx256m [ Interval 0 ]** | **TestModule :: -Xms64m -Xmx256m [ Interval 1m ]** |
| ----------------------------- | --------------------------- | ------------------------------------------------ | ------------------------------------------------- |
| Serial GC                     | **-XX:+UseSerialGC**        | Primary : 300mb \| Secondary : 300mb             | Primary : 87mb \| Secondary : 87mb                |
| Parallel GC                   | **-XX:+UseParallelGC**      | Primary : 300mb \| Secondary : 300mb             | Primary : 152mb \| Secondary : 153mb              |
| Parallel Old GC               | **-XX:-UseParallelOldGC**   | Primary : 286mb \| Secondary : 285mb             | Primary : 150mb \| Secondary : 150mb              |
| CMS(Concurrent Mark-Sweep) GC | **-XX:+UseConcMarkSweepGC** | Primary : 122mb \| Secondary : 131mb             | Primary : 85mb \| Secondary : 86mb                |
| G1(Garbage-First) GC          | **-XX:+UseG1GC**            | Primary : 280mb \| Secondary : 300mb             | Primary : 220mb \| Secondary : 220mb              |

[특이사항]

- CMS GC : -Xms -Xmx 없이도 준수한 메모리 사용량 보임 [ 155mb ] 다른 GC의 경우 500~700mb