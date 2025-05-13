# Gradle에서 CreateStartScripts 템플릿 커스터마이징 및 윈도우/리눅스 스크립트 처리

### 📌 내용 요약

1. **Gradle의 `CreateStartScripts`** 작업은 `windowsStartScriptGenerator`와 `unixStartScriptGenerator`를 통해 OS별 실행 스크립트를 생성할 수 있다.

2. `.template` 파일을 지정하면 배치/셸 스크립트 내용을 직접 정의할 수 있다.

3. 템플릿 내부에서 Windows 배치 변수 

   ```
   %VAR%
   ```

    사용 시:

   - 템플릿 파서나 실행 시점에서 오류가 발생할 수 있으므로, 경로나 변수는 반드시 `"..."`로 감싸야 한다.
   - 문법에 문제가 있을 경우 스크립트가 **빌드되지 않음**.

4. 리눅스용 스크립트(

   ```
   start
   ```

   , 

   ```
   stop
   ```

    등)는 Windows 환경에서 생성되면 줄바꿈이 

   ```
   CRLF
   ```

   로 되어 있어 

   `dos2unix` 후처리

   가 필요하다.

   - Gradle 6.8은 `lineSeparator` 설정이 없어, 템플릿을 직접 **LF로 저장**하거나 `dos2unix` 명령어를 통해 처리해야 한다.

5. Windows에서 PID를 조회하고 종료하는 배치 스크립트는 `ENABLEDELAYEDEXPANSION`과 `FOR` 문 내 변수 처리에 주의해야 한다.

------

### 🛠 적용한 코드 리팩토링

### 🔁 기존 방식

```groovy
task stop(type: CreateStartScripts) {
    def stopFile = file("script/stop.sh")
    def stopWinFile = file("script/stop.bat")

    def unix = resources.text.fromFile(stopFile).asString()
    def windows = resources.text.fromFile(stopWinFile).asString()

    applicationName = 'stop'
    outputDir = new File(project.buildDir, 'script')
    doLast {
        def stop = new File(outputDir, 'stop')
        stop.write(unix)
        def stopWin = new File(outputDir, 'stop.bat')
        stopWin.write(windows)
        exec {
            commandLine 'cmd', 'dos2unix', stop.absolutePath
        }
    }
}
```

### ✅ 리팩토링 후

```groovy
task stop(type: CreateStartScripts) {
    applicationName = "stop"
    unixStartScriptGenerator.template = resources.text.fromFile('script/stop.sh.template')
    windowsStartScriptGenerator.template = resources.text.fromFile('script/stop.bat.template')
    outputDir = new File(project.buildDir, 'script')
    doLast {
        def stop = new File(outputDir, 'stop')
        exec {
            commandLine 'cmd', 'dos2unix', stop.absolutePath
        }
    }
}
```

------

### ⚠️ 주의할 점

- 배치 스크립트에서 `SET /P VAR=<file` 같은 문법은 반드시 파일 경로를 **`"..."`*로 감싸야 한다.

- `FOR /F` 루프 내 `SET` 변수는 바로 사용하려면 **`setlocal ENABLEDELAYEDEXPANSION`** 설정이 필요하다.

  > Windows 배치 스크립트에서는 FOR, IF 등 블록 내부에서 SET으로 변경된 변수 값을 바로 사용하려면 setlocal ENABLEDELAYEDEXPANSION 설정이 필요하다.
  >
  > 이는 변수 확장이 실행 전에 일어나기 때문이며, 이를 실행 시점 확장으로 변경하려면 `%VAR%` 대신 `!VAR!`를 사용해야 한다.

- Gradle 6.x에서는 `lineSeparator` 옵션이 없으므로, 템플릿 파일을 직접 LF로 저장하거나 `dos2unix`를 통해 변환하는 방식으로 대응해야 한다.

------

### 🧠 느낀 점

- 단순한 실행 스크립트 생성도 Gradle을 활용하면 확장성과 유지보수성이 크게 향상된다.
- OS 간의 줄바꿈 방식(CRLF vs LF)을 간과하면 의외의 실행 오류가 발생할 수 있으므로 주의해야 한다.
- Gradle 버전마다 지원 기능이 다르므로 문서 확인은 필수이며, 하위 호환성이 중요한 프로젝트에서는 특히 신중해야 한다.