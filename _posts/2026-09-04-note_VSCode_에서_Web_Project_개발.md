---
layout: single
title: "VSCode_에서_Web_Project_개발"
excerpt: "VSCode_에서_Web_Project_개발"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-09-04"
last_modified_at: "2026-09-04 14:25:53 +0900"
mermaid: false
---
# 1. VS Code + Tomcat + Maven 프로젝트 배포/기동 방법
현재 26_KTR 환경처럼 **Spring Framework 5.3 + JDK 11 + Maven + WAR 배포** 구조라면 다음 구성을 권장합니다.
```text
VS Code
 ├─ Extension Pack for Java
 ├─ Maven for Java
 └─ Community Server Connectors
        ↓
Apache Tomcat 9
        ↓
mvn clean package
        ↓
target/*.war
        ↓
Tomcat webapps 배포
        ↓
Tomcat Start
```
Spring Framework 5.3은 Java EE의 `javax.*` Servlet API 기반이고 Tomcat 8/9와 호환됩니다. 반면 Tomcat 10부터 Servlet API 패키지가 `javax.* → jakarta.*`로 변경되므로, 기존 Spring 5.3 프로젝트의 로컬 테스트 서버로는 **Tomcat 9를 사용하는 것이 안전합니다.** 
## 1-1. VS Code Tomcat 플러그인 선택
### 1-1-1. 권장
```text
Community Server Connectors
Publisher: Red Hat
Extension ID:
redhat.vscode-community-server-connector
```
VS Code 공식 Java 문서에서도 Tomcat 등의 Application Server 연동 방법으로 Community Server Connectors를 안내하고 있으며, Tomcat 5.5~9.0을 지원합니다. 
기존:
```text
Tomcat for Java
adashen.vscode-tomcat
```
도 Marketplace에 남아 있지만 프로젝트 자체에서 **deprecated** 되었고 Community Server Connectors 사용을 권장하고 있습니다. 신규 환경에서는 사용하지 않는 편이 좋습니다. 
## 1-2. 필요한 VS Code Extension
최소 다음 3개를 권장합니다.

| Extension | 용도 | 필수도 |
|---|---|---:|
| Language Support for Java by Red Hat | Java 프로젝트 인식/컴파일 | 필수 |
| Maven for Java | Maven Goal 실행 | 필수 |
| Community Server Connectors | Tomcat 등록/기동/배포 | 권장 |
JUnit 테스트까지 수행한다면:
```text
Test Runner for Java
Debugger for Java
```
도 설치하는 것이 좋습니다.
사실상:
```text
Extension Pack for Java
```
를 설치하면 Java 관련 주요 확장들이 같이 설치됩니다. Maven for Java는 Maven 프로젝트 탐색과 Maven Goal 실행을 지원합니다. 
# 2. 온라인 환경에서 설치
VS Code:
```text
Ctrl + Shift + X
```
검색:
```text
Community Server Connectors
```
선택:
```text
Install
```
그리고:
```text
Maven for Java
```
도 설치합니다.
# 3. 현재처럼 폐쇄망이라면 VSIX 설치
현재 로컬이 인터넷 연결이 불가능한 폐쇄망이라면 **인터넷 가능한 PC에서 `.vsix` 파일을 받아 반입하는 방법**이 가장 현실적입니다.
인터넷 PC에서 다음 Extension의 VSIX를 확보합니다.
```text
Community Server Connectors
Runtime Server Protocol UI
```
Community Server Connectors는 RSP UI에 의존합니다. 온라인 설치에서는 자동 설치되지만 폐쇄망에서는 **의존 Extension도 같이 가져오는 것이 안전합니다.** 
그리고 VS Code에서:
```text
Ctrl + Shift + P
```
입력:
```text
Extensions: Install from VSIX...
```
파일 선택:
```text
redhat.vscode-rsp-ui-xxxx.vsix
redhat.vscode-community-server-connector-xxxx.vsix
```
순으로 설치합니다.
CLI에서도 가능합니다.
```powershell
code --install-extension rsp-ui-버전.vsix
code --install-extension vscode-community-server-connector-버전.vsix
```
설치 후 VS Code를 재시작합니다.
# 4. Tomcat 설치
현재 프로젝트에는 **Tomcat 9.x**를 권장합니다.
예를 들어:
```text
C:\dev\
 ├─ java\
 │   └─ jdk-11.0.21
 │
 ├─ maven\
 │   └─ apache-maven-3.8.x
 │
 └─ tomcat\
     └─ apache-tomcat-9.0.xx
```
Tomcat 디렉토리는 최소 다음 구조여야 합니다.
```text
apache-tomcat-9.0.xx
├─ bin
├─ conf
├─ lib
├─ logs
├─ temp
├─ webapps
└─ work
```
# 5. JAVA_HOME 확인
PowerShell 또는 CMD에서:
```powershell
java -version
```
예:
```text
java version "11.0.21"
```
확인:
```powershell
echo $env:JAVA_HOME
```
CMD:
```cmd
echo %JAVA_HOME%
```
예:
```text
C:\dev\java\jdk-11.0.21
```
Maven도 확인합니다.
```powershell
mvn -version
```
중요한 부분:
```text
Apache Maven 3.x.x
Java version: 11.0.xx
Java home: C:\dev\java\jdk-11.0.xx
```
**Maven과 Tomcat이 동일한 JDK 11을 사용하는 것이 좋습니다.**
# 6. VS Code에서 Tomcat 등록
Community Server Connectors 설치 후:
```text
Ctrl + Shift + P
```
입력:
```text
Servers: Add Local Server
```
또는 VS Code 좌측:
```text
SERVERS
```
에서:
```text
Add Server
```
선택합니다.
Tomcat Home:
```text
C:\dev\tomcat\apache-tomcat-9.0.xx
```
를 지정합니다.
RSP UI는 `Add Local Server`, `Start`, `Stop`, `Restart`, `Debug`, `Add Deployment to Server`, `Run on Server` 등의 기능을 제공합니다. 
정상 등록되면 대략:
```text
SERVERS
└─ Community Server Connector
   └─ Tomcat 9.0
```
형태로 나타납니다.
# 7. JDK 11 경로가 제대로 잡히지 않는 경우
`.vscode/settings.json`에 명시할 수 있습니다.
Windows:
```json
{
    "rsp-ui.rsp.java.home": "C:\\dev\\java\\jdk-11.0.21"
}
```
RSP UI는 Tomcat 등의 Java Runtime을 실행하기 위한 Java Home 설정을 제공합니다. Windows에서는 `\`를 `\\`로 작성해야 합니다. 
# 8. Maven 프로젝트인지 먼저 확인
프로젝트 루트:
```text
bk-fo
├─ pom.xml
├─ src
│  ├─ main
│  │  ├─ java
│  │  ├─ resources
│  │  └─ webapp
│  │     └─ WEB-INF
│  └─ test
│     └─ java
└─ target
```
`pom.xml`에서 반드시:
```xml
<packaging>war</packaging>
```
인지 확인합니다.
예:
```xml
<groupId>bk</groupId>
<artifactId>bk-fo</artifactId>
<version>1.0.0</version>
<packaging>war</packaging>
```
# 9. WAR 이름 지정
`pom.xml`에서:
```xml
<build>
    <finalName>bk-fo</finalName>
</build>
```
로 지정하면:
```text
target/bk-fo.war
```
가 생성됩니다.
URL은 기본적으로:
```text
http://localhost:8080/bk-fo/
```
가 됩니다.
## 9-1. ROOT Context로 실행하려면
WAR를:
```text
ROOT.war
```
라는 이름으로 배포하면:
```text
http://localhost:8080/
```
로 접근할 수 있습니다.
다만 개발환경에서도 운영 context 구조와 맞추는 것이 좋으므로 기존 Eclipse/Tomcat에서 사용하던 Context Path를 먼저 확인하는 것을 권장합니다.
# 10. Maven으로 프로젝트 빌드
VS Code Terminal:
```text
Ctrl + `
```
프로젝트 루트에서:
```powershell
mvn clean package
```
현재처럼 폐쇄망이면:
```powershell
mvn -o clean package
```
를 권장합니다.
`-o`:
```text
Offline Mode
```
입니다.
즉:
```text
인터넷/Nexus 접근하지 않고
로컬 .m2 repository의 dependency만 사용
```
합니다.
# 11. JUnit 포함 빌드
현재 작성한:
```text
JoinSellerTransactionTest
```
까지 실행하려면:
```powershell
mvn -o clean test package
```
또는 일반적으로:
```powershell
mvn -o clean package
```
만 해도 Maven lifecycle에서 테스트가 실행됩니다.
성공하면:
```text
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0
...
BUILD SUCCESS
```
이 나오고:
```text
target\bk-fo.war
```
가 생성됩니다.
# 12. 테스트를 생략하고 WAR만 생성
서버 기동 테스트만 빠르게 하고 싶다면:
```powershell
mvn -o clean package -DskipTests
```
주의:
```text
-DskipTests
```
는:
- 테스트 소스 컴파일: 수행
- 테스트 실행: 생략
입니다.
테스트 컴파일까지 완전히 생략하려면:
```powershell
mvn -o clean package -Dmaven.test.skip=true
```
다만 일반적인 개발에서는:
```text
-DskipTests
```
를 더 권장합니다.
# 13. WAR 생성 확인
정상적인 경우:
```text
bk-fo
└─ target
   ├─ classes
   ├─ test-classes
   ├─ bk-fo
   └─ bk-fo.war
```
확인 대상:
```text
target\bk-fo.war
```
# 14. 방법 A - VS Code에서 WAR를 Tomcat에 배포
가장 직관적인 방법입니다.
VS Code Explorer에서:
```text
target
└─ bk-fo.war
```
우클릭 후:
```text
Run on Server
```
또는:
```text
Add Deployment to Server
```
선택.
그리고:
```text
Tomcat 9
```
선택합니다.
RSP UI는 WAR 파일에 대해 `Run on Server`와 `Add Deployment to Server`를 지원합니다. 
결과적으로:
```text
target\bk-fo.war
        ↓
Tomcat
        ↓
webapps\bk-fo.war
        ↓
webapps\bk-fo\
```
형태로 배포됩니다.
# 15. Tomcat 기동
SERVERS:
```text
Tomcat 9
```
우클릭:
```text
Start
```
정상 시작 시 로그:
```text
INFO [main] org.apache.catalina.startup.Catalina.start
Server startup in xxxx ms
```
브라우저:
```text
http://localhost:8080/
```
애플리케이션:
```text
http://localhost:8080/bk-fo/
```
# 16. 방법 B - 플러그인 없이 Maven + Tomcat 직접 배포
저는 **폐쇄망 개발환경에서는 이 방법도 반드시 알아두는 것을 권장**합니다.
VS Code Extension 오류와 완전히 독립적입니다.
### 16-1-1. ① Maven WAR 생성
```powershell
mvn -o clean package -DskipTests
```
생성:
```text
target\bk-fo.war
```
### 16-1-2. ② Tomcat으로 복사
Windows:
```cmd
copy /Y target\bk-fo.war C:\dev\tomcat\apache-tomcat-9.0.xx\webapps\
```
### 16-1-3. ③ Tomcat 시작
```cmd
C:\dev\tomcat\apache-tomcat-9.0.xx\bin\startup.bat
```
### 16-1-4. ④ 로그 확인
```text
C:\dev\tomcat\apache-tomcat-9.0.xx\logs\
```
주요 로그:
```text
catalina.YYYY-MM-DD.log
localhost.YYYY-MM-DD.log
```
또는 CMD에서 직접 로그를 보면서 시작하려면:
```cmd
C:\dev\tomcat\apache-tomcat-9.0.xx\bin\catalina.bat run
```
개발환경에서는 오히려:
```cmd
catalina.bat run
```
을 추천합니다.
Tomcat 로그가 현재 Terminal에 바로 나타나기 때문입니다.
# 17. Tomcat 종료
```cmd
C:\dev\tomcat\apache-tomcat-9.0.xx\bin\shutdown.bat
```
또는 VS Code:
```text
SERVERS
→ Tomcat 9
→ Stop
```
# 18. 가장 실무적인 개발 반복 절차
현재 프로젝트에서는 다음 순서가 가장 안전합니다.
```text
① Java 수정
      ↓
② JUnit 실행
      ↓
③ Maven package
      ↓
④ WAR 생성
      ↓
⑤ 기존 WAR / exploded directory 제거
      ↓
⑥ WAR 배포
      ↓
⑦ Tomcat 기동
      ↓
⑧ 로그 확인
      ↓
⑨ 브라우저 테스트
```
명령으로 보면:
```powershell
mvn -o clean package
```
성공:
```text
BUILD SUCCESS
```
배포:
```cmd
copy /Y target\bk-fo.war C:\dev\tomcat\apache-tomcat-9.0.xx\webapps\
```
실행:
```cmd
C:\dev\tomcat\apache-tomcat-9.0.xx\bin\catalina.bat run
```
# 19. 재배포 시 주의
기존 WAR가 이미 있는 상태에서 새 WAR만 계속 덮어쓰면 Tomcat의 exploded directory가 남아 문제가 발생할 수 있습니다.
예:
```text
webapps
├─ bk-fo.war      ← 새 WAR
└─ bk-fo           ← 이전 배포의 exploded 디렉토리
```
개발 중 이상 동작이 나타난다면 Tomcat을 정지한 후:
```text
webapps\bk-fo.war
webapps\bk-fo\
work\Catalina\localhost\bk-fo\
```
를 삭제하고 다시 배포하는 것이 안전합니다.
예:
```cmd
del C:\dev\tomcat\apache-tomcat-9.0.xx\webapps\bk-fo.war
rmdir /S /Q C:\dev\tomcat\apache-tomcat-9.0.xx\webapps\bk-fo
rmdir /S /Q C:\dev\tomcat\apache-tomcat-9.0.xx\work\Catalina\localhost\bk-fo
```
그 후:
```cmd
copy target\bk-fo.war C:\dev\tomcat\apache-tomcat-9.0.xx\webapps\
```
# 20. Maven Tomcat Plugin 방식은 우선 추천하지 않음
인터넷에 다음 예제가 많이 나옵니다.
```xml
<plugin>
    <groupId>org.apache.tomcat.maven</groupId>
    <artifactId>tomcat7-maven-plugin</artifactId>
    <version>2.2</version>
</plugin>
```
그리고:
```powershell
mvn tomcat7:run
```
형태입니다.
VS Code Java Debug 문서에도 과거 이 방식의 예제가 존재합니다. 
그러나 현재 26_KTR 환경에는 **우선 추천하지 않습니다.**

| 이유 | 설명 |
|---|---|
| 오래된 Plugin | `tomcat7-maven-plugin` 기반 |
| 폐쇄망 | Plugin dependency 추가 확보 필요 |
| 실제 서버와 차이 | 별도 Tomcat 9 환경과 실행 방식 차이 발생 |
| 운영 재현성 | 개발/운영 모두 WAR external deployment 구조가 더 유사 |
| 기존 Maven 문제 | 현재 `maven-war-plugin` dependency 문제도 이미 존재했음 |
따라서:
```text
Maven = Build 담당
Tomcat = Runtime 담당
```
으로 분리하는 것이 좋습니다.
# 21. 현재 프로젝트에 추천하는 최종 구성
```text
C:\KTR\
├─ workspace\
│  └─ bk-fo\
│     ├─ pom.xml
│     ├─ src\
│     └─ target\
│        └─ bk-fo.war
│
├─ tools\
│  ├─ jdk-11.0.21\
│  ├─ apache-maven-3.x\
│  └─ apache-tomcat-9.0.xx\
│     ├─ bin
│     ├─ conf
│     ├─ lib
│     ├─ logs
│     ├─ webapps
│     └─ work
│
└─ maven-repository\
```
VS Code:
```text
Extension Pack for Java
Maven for Java
Community Server Connectors
```
개발 명령:
```powershell
mvn -o clean package
```
배포:
```text
target/bk-fo.war
        ↓
Tomcat 9 / webapps
```
실행:
```cmd
catalina.bat run
```
또는:
```text
VS Code SERVERS
→ Tomcat 9
→ Start
```
## 21-1. 추천 우선순위
| 방식 | 추천도 | 용도 |
|---|---:|---|
| `mvn package → WAR → VS Code Run on Server` | ★★★★★ | 평상시 개발 |
| `mvn package → WAR 직접 복사 → catalina.bat run` | ★★★★★ | 폐쇄망/문제 원인 분석 |
| VS Code Server Connector Start/Stop | ★★★★☆ | 편의성 |
| `mvn tomcat7:run` | ★★☆☆☆ | 현재 프로젝트에는 비추천 |
| Tomcat for Java 구 확장 | ★☆☆☆☆ | Deprecated |
**현재 26_KTR 로컬 개발에서는 `Community Server Connectors + Tomcat 9 + Maven WAR 배포`를 기본으로 하고, 문제가 생길 때는 플러그인을 배제한 `mvn -o package → WAR 직접 복사 → catalina.bat run` 방식으로 원인을 분리하는 구조가 가장 안정적입니다.**

# 22. 폐쇄망 VS Code + Spring Framework 5.3 Web 프로젝트 권장 확장 구성
현재 설치 상태라면 **Java/Maven/JUnit 개발 기반은 이미 거의 완성**되어 있습니다. 추가 설치는 `XML`, `Tomcat`, `JSP` 중심으로 최소화하는 것이 좋습니다.
현재 프로젝트 기준 권장 구조는 다음입니다.
```text
VS Code
├─ Java 개발                           ← 현재 구축 완료
│  ├─ Language Support for Java by Red Hat
│  ├─ Debugger for Java
│  ├─ Test Runner for Java
│  ├─ Maven for Java
│  ├─ Project Manager for Java
│  └─ Decompiler for Java
│
├─ Spring/XML                          ← 추가 권장
│  └─ XML Language Support by Red Hat
│
├─ WEB/JSP                             ← 추가 권장
│  └─ JSP Language Support
│
├─ Tomcat                              ← 추가 필수
│  ├─ Runtime Server Protocol UI
│  └─ Community Server Connectors
│
└─ 개발 편의                           ← 선택
   ├─ REST Client
   ├─ Checkstyle for Java
   ├─ SonarQube for IDE
   └─ JBoss Toolkit
```
## 22-1. 현재 설치된 확장 평가
현재 설치된 확장은:
| 현재 설치 확장 | 필요도 | 역할 | 조치 |
|---|---:|---|---|
| Extension Pack for Java | ★★★★★ | Java 개발 기본 패키지 | 유지 |
| Language Support for Java by Red Hat | ★★★★★ | Java IntelliSense, 컴파일, Refactoring | 유지 |
| Debugger for Java | ★★★★★ | Java 디버깅 | 유지 |
| Test Runner for Java | ★★★★★ | JUnit/TestNG | 유지 |
| Maven for Java | ★★★★★ | Maven 실행/프로젝트 관리 | 유지 |
| Project Manager for Java | ★★★★★ | Java Project/Dependency 탐색 | 유지 |
| Decompiler for Java | ★★★★☆ | JAR의 `.class` 소스 확인 | 유지 |
| Java | 확인 필요 | Oracle Java일 가능성 | 아래 설명 확인 |
Microsoft의 Java Extension Pack은 Java Language Support, Debugger, Test Runner, Maven, Project Manager 등 핵심 Java 기능을 제공하므로 현재 설치 구성만으로 일반 Java 개발·Maven·JUnit은 충분합니다. 
## 22-2. `Java`라는 별도 Extension은 확인 필요
VS Code Extension 목록에서 단순히:
```text
Java
```
라는 확장이 별도로 설치되어 있다면 Extension 상세화면에서 **ID**를 확인하십시오.
만약:
```text
Oracle.oracle-java
```
라면 Oracle의 별도 Java Language Server입니다. Oracle Java 자체도 Maven, Debugger, Test Explorer, Java Language Server 기능을 제공합니다. 
현재 이미:
```text
redhat.java
vscjava.vscode-java-debug
vscjava.vscode-java-test
vscjava.vscode-maven
```
조합을 사용하고 있으므로 `Oracle.oracle-java`까지 동시에 활성화할 실익은 크지 않습니다.
### 22-2-1. 권장
```text
Extension Pack for Java + Red Hat Java 계열로 통일
```
따라서 실제 ID가:
```text
Oracle.oracle-java
```
이면 **Disable**을 권장합니다.
삭제하지 않고:
```text
Extensions
→ Java
→ 톱니바퀴
→ Disable
```
만 해도 됩니다.
이유는 Java Language Server, Test Explorer, Debug 기능이 중복되기 때문입니다.
---
# 23. 추가 설치 1순위: XML Language Support by Red Hat
## 23-1. 확장
```text
Name : XML
Publisher : Red Hat
ID : redhat.vscode-xml
```
### 23-1-1. 필요도
**★★★★★ 필수에 가까움**
Spring Framework 5.3의 전통적인 Web 프로젝트에서는 XML 사용 비중이 높습니다.
현재 프로젝트에서도 다음 파일을 다룰 가능성이 큽니다.
```text
pom.xml
web.xml
applicationContext.xml
dispatcher-servlet.xml
context-*.xml
tiles.xml
ehcache.xml
logback.xml
jboss-web.xml
```
XML Extension은:
- XML syntax validation
- 자동 태그 닫기
- Formatting
- XSD validation
- DTD validation
- 자동완성
- XML symbol 탐색
을 지원합니다. 
### 23-1-2. 사용 예
잘못 작성하면:
```xml
<bean id="cryptoUtil"
      class="app.common.util.CryptoUtil">
```
종료 태그 누락 등을 바로 검출할 수 있습니다.
또한:
```text
Shift + Alt + F
```
로 XML Formatting이 가능합니다.
---
# 24. 추가 설치 2순위: Tomcat Server Connector
Spring 5.3 WAR 프로젝트를 VS Code에서 실행하려면 이것이 가장 중요합니다.
## 24-1. 설치해야 할 2개
```text
Runtime Server Protocol UI
ID: redhat.vscode-rsp-ui
Community Server Connectors
ID: redhat.vscode-community-server-connector
```
`Community Server Connectors`는 RSP UI를 기반으로 동작합니다. Tomcat 5.5~9.0, Jetty 등을 관리할 수 있으며 Tomcat Start/Stop/Publish 기능을 제공합니다. 
### 24-1-1. 현재 프로젝트에는 Tomcat 9 권장
Spring Framework 5.3 프로젝트라면:
```text
Spring 5.3
   ↓
javax.servlet.*
   ↓
Tomcat 9
```
구성을 권장합니다.
Tomcat 10 이상은 `jakarta.servlet.*` 기반이라 기존 Spring 5.3/JSP/Servlet 프로젝트를 그대로 올리는 용도로는 적절하지 않습니다.
## 24-2. 서버 등록
설치 후:
```text
Ctrl + Shift + P
```
입력:
```text
Add Local Server
```
Tomcat 디렉토리:
```text
C:\dev\apache-tomcat-9.0.xx
```
선택.
정상적으로 등록되면:
```text
SERVERS
└─ Tomcat 9.0
```
형태로 표시됩니다.
### 24-2-1. Start
```text
SERVERS
→ Tomcat
→ 우클릭
→ Start
```
### 24-2-2. Stop
```text
SERVERS
→ Tomcat
→ 우클릭
→ Stop
```
### 24-2-3. WAR 실행
Maven:
```bash
mvn -o clean package
```
생성:
```text
target/bk-fo.war
```
WAR 우클릭:
```text
Run on Server
```
또는:
```text
Add Deployment to Server
```
RSP UI는 WAR 파일의 Run on Server 및 Debug on Server를 지원합니다. 
---
# 25. 추가 설치 3순위: JSP Language Support
현재 프로젝트가 **JSP + Tiles**를 사용하므로 권장합니다.
## 25-1. 확장
예:
```text
JSP Language Support
ID: samuel-weinhardt.vscode-jsp-lang
```
### 25-1-1. 필요도
**★★★★☆**
기본 VS Code도 HTML은 잘 처리하지만:
```jsp
<%@ page ... %>
<c:if>
<c:forEach>
<jsp:include>
${variable}
```
같은 JSP 구문을 완전히 이해하지는 못합니다.
JSP Language Support는 JSP syntax highlighting을 제공합니다. 다만 JSP 전체 의미 분석까지 제공하는 Java IDE 수준의 확장은 아니라는 점은 주의해야 합니다. 
### 25-1-2. 설정 추천
`.vscode/settings.json`
```json
{
    "emmet.includeLanguages": {
        "jsp": "html"
    }
}
```
그러면 JSP 내부에서:
```text
div
table
input
form
```
등 HTML Emmet 기능도 사용할 수 있습니다.
---
# 26. REST Client
## 26-1. 확장
```text
REST Client
ID: humao.rest-client
```
### 26-1-1. 필요도
**★★★★☆ 선택 권장**
Controller/AJAX/REST 테스트에 상당히 유용합니다.
Postman 없이 VS Code에서 직접:
```http
GET http://localhost:8080/bk-fo/main.do
```
또는:
```http
POST http://localhost:8080/bk-fo/none/mm/resultSuccessFromInicis.do
Content-Type: application/x-www-form-urlencoded
txId=TEST001&resultCode=0000
```
형태로 요청할 수 있습니다.
`.http` 파일 하나에 여러 요청을 저장할 수도 있습니다. 
예:
```text
test/
└─ http/
   ├─ seller.http
   ├─ login.http
   └─ inicis.http
```
`seller.http`:
```http
### 업체 조회
GET http://localhost:8080/bk-fo/seller/search.do
### 업체 등록
POST http://localhost:8080/bk-fo/seller/save.do
Content-Type: application/json
{
    "sellerId": "TEST01"
}
```
폐쇄망에서도 **localhost나 내부 개발 WAS 호출에는 문제가 없습니다.**
---
# 27. Checkstyle for Java
## 27-1. 확장
```text
Checkstyle for Java
ID: shengchen.vscode-checkstyle
```
### 27-1-1. 필요도
**★★★☆☆ 프로젝트에 Checkstyle 정책이 있을 때**
소스 작성 중:
- naming rule
- indentation
- import order
- JavaDoc
- line length
등을 검사합니다. 
다만 현재 26_KTR 프로젝트처럼 기존 소스 포맷이 이미 확립되어 있다면 **팀에서 Checkstyle XML을 사용하고 있을 때만 설치**하는 것이 좋습니다.
예:
```text
config/checkstyle/checkstyle.xml
```
가 있다면 유용합니다.
반대로 Checkstyle 정책이 없다면 굳이 설치할 필요 없습니다.
### 27-1-2. 폐쇄망 주의
일부 Checkstyle 버전은 필요한 JAR를 자동 다운로드하려 할 수 있습니다. 따라서 폐쇄망에서는:
```text
Checkstyle Extension
+
필요 Checkstyle JAR
+
checkstyle.xml
```
까지 함께 반입해야 안전합니다. 
---
# 28. SonarQube for IDE
현재 프로젝트가 SonarQube를 사용하므로 선택적으로 추천할 수 있습니다.
### 28-1-1. 용도
코딩하는 순간:
```text
Bug
Vulnerability
Code Smell
Null 가능성
Resource 미반환
잘못된 Exception 처리
```
등을 확인하는 용도입니다.
### 28-1-2. 추천도
```text
★★★☆☆
```
CI에서 이미 SonarQube를 돌린다면 필수는 아닙니다.
하지만 개발단계에서 사전에 문제를 잡고 싶다면 유용합니다.
폐쇄망에서도 IDE 자체 분석은 가능하지만, **Connected Mode로 사내 SonarQube 서버와 연동하려면 폐쇄망 PC에서 해당 내부 SonarQube 주소에 접근 가능해야 합니다.**
---
# 29. JBoss Toolkit
현재 실제 개발/운영 환경에 JBoss EAP 7.2가 있기 때문에 별도 선택지입니다.
## 29-1. Extension
```text
JBoss Toolkit
ID: redhat.vscode-server-connector
```
Red Hat의 JBoss Toolkit은 WildFly 및 JBoss EAP를 제어하며 EAP 8 이하를 지원 범위로 명시하고 있으므로 EAP 7.2도 대상입니다. 
### 29-1-1. 하지만 현재 로컬 Tomcat 개발만 한다면
```text
설치 불필요
```
입니다.
즉:

| 로컬 실행환경 | Extension |
|---|---|
| Tomcat 9 | Community Server Connectors |
| JBoss EAP 7.2 | JBoss Toolkit |
| 둘 다 | 둘 다 설치 가능 |
둘 모두 RSP UI를 사용합니다.
---
# 30. Spring Boot Extension Pack은 설치할 필요 없음
인터넷 검색을 하면 거의 항상:
```text
Spring Boot Extension Pack
Spring Boot Tools
Spring Initializr
Spring Boot Dashboard
```
가 추천됩니다.
하지만 현재 프로젝트는:
```text
Spring Framework 5.3
WAR
web.xml/XML
외부 Tomcat/JBoss
```
구조이므로 우선 설치할 이유가 없습니다.
Spring Boot Tools는 Spring Boot의:
```text
application.properties
application.yml
Boot configuration
Boot process
```
등에 특화되어 있습니다. Spring Initializr와 Dashboard 역시 Boot 프로젝트 생성/기동용입니다. 
따라서:
```text
Spring Boot 프로젝트가 아님
        ↓
Spring Boot Extension Pack 불필요
```
로 판단합니다.
---
# 31. YAML Extension도 현재는 선택
```text
redhat.vscode-yaml
```
다음 파일이 많다면 설치:
```text
application.yml
docker-compose.yml
GitLab CI YAML
Kubernetes YAML
```
하지만 현재 Spring 5.3 XML 기반 Web 프로젝트만 개발한다면 우선순위가 낮습니다.
---
# 32. Gradle Extension도 필요 없음
현재:
```text
pom.xml
Maven
```
프로젝트이므로:
```text
Gradle for Java
```
는 사용하지 않아도 됩니다.
최신 Extension Pack for Java 구성에는 Gradle 지원도 포함될 수 있지만 Maven 프로젝트에서 굳이 사용하지 않습니다. 
---
# 33. 폐쇄망 Extension 설치 방법
폐쇄망에서는 **VSIX 방식으로 통일하는 것을 권장**합니다.
## 33-1. 인터넷 가능 PC
필요한 `.vsix` 파일을 준비합니다.
추천 반입 세트:
```text
vscode-xml-x.x.x.vsix
vscode-rsp-ui-x.x.x.vsix
vscode-community-server-connector-x.x.x.vsix
vscode-jsp-lang-x.x.x.vsix
rest-client-x.x.x.vsix
```
선택:
```text
vscode-checkstyle-x.x.x.vsix
sonarlint-vscode-x.x.x.vsix
vscode-server-connector-x.x.x.vsix
```
VS Code는 공식적으로 `.vsix` 파일의 수동 설치를 지원합니다. 
## 33-2. 폐쇄망 PC
### 33-2-1. GUI 설치
```text
Ctrl + Shift + P
```
입력:
```text
Extensions: Install from VSIX...
```
VSIX 선택.
### 33-2-2. 명령어 설치
```cmd
code --install-extension redhat.vscode-xml-x.x.x.vsix
```
여러 개:
```cmd
code --install-extension redhat.vscode-rsp-ui-x.x.x.vsix
code --install-extension redhat.vscode-community-server-connector-x.x.x.vsix
code --install-extension samuel-weinhardt.vscode-jsp-lang-x.x.x.vsix
```
VS Code CLI도 VSIX 경로를 직접 받아 설치할 수 있습니다. 
---
# 34. 폐쇄망에서는 의존 Extension을 반드시 함께 반입
온라인 Marketplace에서는:
```text
Community Server Connectors
        ↓
Runtime Server Protocol UI
```
의존성을 자동으로 설치합니다. 
하지만 폐쇄망에서는 자동 다운로드가 불가능할 수 있으므로:
```text
① RSP UI VSIX
② Community Server Connectors VSIX
```
순서로 직접 설치하는 것을 권장합니다.
```text
redhat.vscode-rsp-ui
        ↓
redhat.vscode-community-server-connector
```
---
# 35. Extension 버전까지 기록하는 것이 중요
폐쇄망에서는 한번 정상 동작한 환경을 **그대로 보존**하는 것이 매우 중요합니다.
인터넷 PC 또는 정상 개발 PC에서:
```cmd
code --list-extensions --show-versions
```
실행.
예:
```text
redhat.java@x.x.x
vscjava.vscode-java-debug@x.x.x
vscjava.vscode-java-test@x.x.x
vscjava.vscode-maven@x.x.x
vscjava.vscode-java-dependency@x.x.x
redhat.vscode-xml@x.x.x
redhat.vscode-rsp-ui@x.x.x
redhat.vscode-community-server-connector@x.x.x
```
이 목록을:
```text
vscode-extensions-version.txt
```
로 프로젝트 개발환경 문서에 보관하는 것을 추천합니다.
---
# 36. JDK 11 프로젝트에서 특히 주의할 점
현재 프로젝트는:
```text
JDK 11
Spring Framework 5.3
```
입니다.
여기서 **프로젝트 실행 JDK와 VS Code Extension 자체를 구동하는 JDK 요구사항이 동일하다고 생각하면 안 됩니다.**
최신 Red Hat Java Extension은 Java Language Server 자체의 실행 환경 요구사항이 상향될 수 있지만 프로젝트 자체는 JDK 8 이상을 지원합니다. 
현재 JUnit까지 정상 실행되고 있으므로 **현재 Red Hat Java Extension 버전을 굳이 업데이트하지 않는 것**을 권장합니다.
폐쇄망에서는:
```text
VS Code 버전
+
Extension 버전
+
JDK 버전
+
Maven 버전
```
을 하나의 검증된 세트로 관리하십시오.
---
# 37. JDK 11 설정
프로젝트용 JDK:
```json
{
    "java.configuration.runtimes": [
        {
            "name": "JavaSE-11",
            "path": "C:\\dev\\jdk-11.0.21",
            "default": true
        }
    ]
}
```
`java.configuration.runtimes`는 Java 프로젝트에서 사용할 JDK 경로를 지정하는 공식 설정입니다. 
확인:
```text
Ctrl + Shift + P
→ Java: Configure Java Runtime
```
다음처럼 보여야 합니다.
```text
JavaSE-11
C:\dev\jdk-11.0.21
```
---
# 38. Maven 폐쇄망 설정
현재 Maven Extension은 그대로 사용하면 됩니다.
Maven 실행파일을 명시적으로 지정하는 것을 추천합니다.
`.vscode/settings.json`:
```json
{
    "maven.executable.path": "C:\\dev\\apache-maven-3.8.8\\bin\\mvn.cmd",
    "maven.executable.preferMavenWrapper": false,
    "maven.settingsFile": "C:\\Users\\사용자\\.m2\\settings.xml"
}
```
`Maven for Java`는 `maven.executable.path`, `maven.settingsFile` 등의 설정을 공식 지원합니다. 
### 38-1-1. 폐쇄망 실행
```bash
mvn -o clean compile
```
JUnit:
```bash
mvn -o test
```
WAR:
```bash
mvn -o clean package
```
테스트 생략:
```bash
mvn -o clean package -DskipTests
```
핵심:
```text
-o = Offline
```
입니다.
---
# 39. Maven Extension 사용법
VS Code 왼쪽:
```text
MAVEN
└─ bk-fo
   ├─ Lifecycle
   │  ├─ clean
   │  ├─ validate
   │  ├─ compile
   │  ├─ test
   │  ├─ package
   │  └─ install
   └─ Plugins
```
각 Goal을 클릭하면 실행됩니다. Maven for Java는 Maven Lifecycle과 Plugin goal 실행을 지원합니다. 
하지만 폐쇄망에서는 Terminal에서:
```bash
mvn -o clean package
```
를 직접 실행하는 것이 가장 명확합니다.
---
# 40. Tomcat용 설정
`.vscode/settings.json`:
```json
{
    "rsp-ui.rsp.java.home": "C:\\dev\\jdk-11.0.21"
}
```
RSP UI는 서버 실행용 Java Home 설정을 지원하며 JDK 11 이상을 사용할 수 있습니다. 
Tomcat:
```text
C:\dev\apache-tomcat-9.0.xx
```
등록:
```text
Ctrl + Shift + P
→ Add Local Server
→ C:\dev\apache-tomcat-9.0.xx
```
---
# 41. 프로젝트용 권장 `.vscode/settings.json`
현재 환경을 기준으로 하면 다음 정도면 충분합니다.
```json
{
    "java.configuration.runtimes": [
        {
            "name": "JavaSE-11",
            "path": "C:\\dev\\jdk-11.0.21",
            "default": true
        }
    ],
    "java.configuration.updateBuildConfiguration": "automatic",
    "maven.executable.path": "C:\\dev\\apache-maven-3.8.8\\bin\\mvn.cmd",
    "maven.executable.preferMavenWrapper": false,
    "maven.settingsFile": "C:\\Users\\사용자\\.m2\\settings.xml",
    "rsp-ui.rsp.java.home": "C:\\dev\\jdk-11.0.21",
    "emmet.includeLanguages": {
        "jsp": "html"
    }
}
```
`path` 부분만 실제 로컬 환경에 맞게 변경하면 됩니다.
---
# 42. 최종 추천 Extension 목록
## 42-1. A. 현재 유지
| Extension                 | ID                               |   판정 |
| ------------------------- | -------------------------------- | ---: |
| Extension Pack for Java   | `vscjava.vscode-java-pack`       | ✅ 유지 |
| Language Support for Java | `redhat.java`                    | ✅ 유지 |
| Debugger for Java         | `vscjava.vscode-java-debug`      | ✅ 유지 |
| Test Runner for Java      | `vscjava.vscode-java-test`       | ✅ 유지 |
| Maven for Java            | `vscjava.vscode-maven`           | ✅ 유지 |
| Project Manager for Java  | `vscjava.vscode-java-dependency` | ✅ 유지 |
| Decompiler for Java       | `dgileadi.java-decompiler`       | ✅ 유지 |
Decompiler는 Red Hat Java Extension과 연계해 source가 없는 클래스의 decompiled source를 보여줍니다. 
## 42-2. B. 지금 추가 권장
|  우선순위 | Extension                   | ID                                         | 이유                         |
| ----: | --------------------------- | ------------------------------------------ | -------------------------- |
| **1** | XML                         | `redhat.vscode-xml`                        | Spring XML/web.xml/pom.xml |
| **2** | Runtime Server Protocol UI  | `redhat.vscode-rsp-ui`                     | Server UI                  |
| **3** | Community Server Connectors | `redhat.vscode-community-server-connector` | Tomcat 9 실행/배포             |
| **4** | JSP Language Support        | `samuel-weinhardt.vscode-jsp-lang`         | JSP/Tiles                  |
| **5** | REST Client                 | `humao.rest-client`                        | Controller/API 테스트         |
## 42-3. C. 필요 시
| Extension           | 사용 조건                      |
| ------------------- | -------------------------- |
| Checkstyle for Java | 프로젝트 Checkstyle 정책 존재      |
| SonarQube for IDE   | IDE에서 사전 품질검사              |
| JBoss Toolkit       | VS Code에서 EAP 7.2 직접 기동/배포 |
| YAML                | YAML을 많이 사용하는 경우           |
## 42-4. D. 현재는 불필요
| Extension                  | 이유                        |
| -------------------------- | ------------------------- |
| Spring Boot Extension Pack | Spring Boot 프로젝트가 아님      |
| Spring Initializr          | 신규 Boot 프로젝트 생성용          |
| Spring Boot Dashboard      | Boot Application 실행용      |
| Gradle for Java            | Maven 사용 중                |
| Lombok Extension           | Lombok 사용 시에만             |
| Docker Extension           | 현재 로컬 WAR/Tomcat 개발에는 불필요 |
# 43. 제가 현재 환경이라면 이렇게 구성합니다
가장 안정적인 최소 세트는:
```text
[현재 설치]
Extension Pack for Java
Decompiler for Java
        +
[추가]
XML by Red Hat
Runtime Server Protocol UI
Community Server Connectors
JSP Language Support
REST Client
```
그리고 개발 흐름을:
```text
Java 수정
 ↓
JUnit Test Runner
 ↓
mvn -o test
 ↓
mvn -o clean package
 ↓
target/bk-fo.war
 ↓
Community Server Connectors
 ↓
Tomcat 9
 ↓
JSP / Controller 확인
```
으로 가져가는 것을 권장합니다.
특히 **폐쇄망이라는 조건 때문에 “많은 Extension 설치”보다 검증된 최소 Extension의 버전을 고정하는 것이 훨씬 중요합니다.** 지금 상태에서 우선 추가할 것은 **`XML + RSP UI + Community Server Connectors + JSP Language Support` 4개**면 충분하며, REST Client까지 넣으면 Spring 5.3 Web 개발환경으로 상당히 완성도가 높습니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
