# VS Code + PMD 6.45.0 + eGovFramework PMD Rule 설치 가이드
현재 환경에서는 다음 구성을 권장합니다.
```text
VS Code
 └─ PMD for Java Extension
      ↓
   PMD 6.45.0      ← 버전 고정
      ↓
eGovFramework 표준 Inspection Ruleset
      ↓
Java 11 / Spring 5.3 Source 검사
```
이 구성은 **SonarQube Server와 완전히 독립적으로 Local에서만 PMD 검사를 수행**합니다.
현재 SonarQube Server 로그에서 확인된 `pmd:SystemPrintln`, `pmd:EmptyCatchBlock`, `pmd:UnusedPrivateField` 등은 전자정부 표준프레임워크가 선정한 39개 PMD 표준 Inspection Rule과 상당 부분 일치합니다. eGovFrame 공식 문서도 표준 Inspection Rule을 39개로 정의하고 있습니다. 
## 1. 최종 권장 버전
|구성요소|권장 버전|이유|
|---|---:|---|
|Java|JDK 11|현재 프로젝트 환경 그대로|
|PMD Engine|**6.45.0**|2023년 말 SonarQube 9.9 + sonar-pmd 환경 재현 후보|
|VS Code Extension|**PMD for Java (`cracrayol.pmd-java`)**|외부 PMD와 Custom Ruleset 지원|
|eGov Rule|**eGovFrame Inspection Rules 3.8**|기존 39개 표준 Rule|
|SonarQube for IDE|삭제 또는 Disable|Local PMD 검사에는 불필요|
PMD 6.45.0은 Java 11 소스를 지원합니다. PMD의 Java 11 지원 자체는 6.6.0부터 제공됐으므로 현재 JDK 11 프로젝트에 문제가 없습니다. 
# 2. 인터넷 가능한 PC에서 준비해야 할 파일 3개
폐쇄망 PC로 다음 세 가지를 반입하면 됩니다.
### ① VS Code `PMD for Java`
Extension:
```text
PMD for Java
Publisher : cracrayol
Extension ID : cracrayol.pmd-java
```
Marketplace:
[PMD for Java - VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=cracrayol.pmd-java&utm_source=chatgpt.com)
이 Extension은 공식 설명상 다음 기능을 지원합니다.
- 개별 Java 파일 검사
- Workspace 전체 검사
- 파일 Open/Save/Change 시 자동 검사
- Custom Ruleset
- 외부 PMD 설치 경로 지정
- 별도 JRE 지정 
현재 저장소의 Extension 버전은 `0.7.3`이며 `javaPMD.pmdBinPath`, `javaPMD.rulesets`, `javaPMD.jrePath` 등의 설정을 제공합니다. 
### ② PMD 6.45.0
공식 PMD 6.45.0 Release:
[PMD 6.45.0 공식 Release](https://github.com/pmd/pmd/releases/tag/pmd_releases%2F6.45.0?utm_source=chatgpt.com)
다운로드 대상:
```text
pmd-bin-6.45.0.zip
```
PMD 6.45.0은 2022년 4월 30일 공식 릴리스입니다. 
### ③ eGovFramework 표준 Inspection Rules
현재 환경에서는 **3.8 Rule Set**을 추천합니다.
공식 다운로드:
[eGovFramework Inspection Rules 3.8 ZIP](https://egovframe.go.kr/wiki/lib/exe/fetch.php?media=egovframework%3Adev3.8%3Aimp%3Aegovinspectionrules-3.8.zip&utm_source=chatgpt.com)
공식 설명:
[eGovFrame Code Inspection 공식 문서](https://egovframe.go.kr/wiki/doku.php?id=egovframework%3Adev4.0%3Aimp%3Ainspection&utm_source=chatgpt.com)
eGovFrame 공식 문서에서도 개발환경 3.8 이상용 표준 Inspection Rules 압축파일을 제공하며, 총 **39개 PMD Rule**을 표준으로 정의합니다. 
# 3. 폐쇄망 PC에 PMD 설치
예를 들어 다음 위치를 사용하겠습니다.
```text
C:\DevTools\
 ├─ pmd-bin-6.45.0\
 └─ pmd-rules\
```
PMD ZIP:
```text
pmd-bin-6.45.0.zip
```
압축 해제:
```text
C:\DevTools\pmd-bin-6.45.0\
 ├─ bin\
 ├─ lib\
 └─ ...
```
**중요:** VS Code 설정에서 지정할 경로는:
```text
C:\DevTools\pmd-bin-6.45.0
```
입니다.
다음처럼 `bin`까지 지정하지 않습니다.
```text
C:\DevTools\pmd-bin-6.45.0\bin    ← X
```
PMD for Java Extension의 `pmdBinPath`는 `bin`과 `lib` 디렉터리를 포함하는 **PMD Root 디렉터리**를 요구합니다. 
# 4. PMD가 JDK 11에서 실행되는지 먼저 확인
CMD:
```cmd
java -version
```
현재 환경이라면:
```text
java version "11..."
```
이어야 합니다.
필요하면:
```cmd
C:\DevTools\jdk-11\bin\java.exe -version
```
확인합니다.
PMD 자체 실행에는 Java 8 이상이면 충분하므로 JDK 11은 문제가 없습니다. 
# 5. eGovFramework Rule 압축 해제
다운로드한:
```text
egovinspectionrules-3.8.zip
```
을 압축 해제합니다.
예:
```text
C:\DevTools\pmd-rules\
```
또는 프로젝트 안에 넣어도 됩니다.
저는 **프로젝트 내에 Ruleset XML만 복사하는 방식**을 더 추천합니다.
예:
```text
bk-fo\
 ├─ src\
 ├─ pom.xml
 ├─ config\
 │   └─ pmd\
 │       └─ EgovInspectionRules_kor-xxxx.xml
 └─ .vscode\
     └─ settings.json
```
압축파일 안의 실제 파일명은 배포본에 따라 다를 수 있습니다. eGovFrame 공식 문서의 예에서는:
```text
EgovInspectionRules_kor-3.5.xml
```
을 사용하고 있으며, 3.8 패키지 계열에서도 유사한 한글/영문 Ruleset XML이 포함됩니다. 
**한글 Ruleset XML을 선택**하면 됩니다.
# 6. eGov Rule이 오래된 Rule 문법인데 PMD 6.45에서 사용할 수 있는가?
여기가 중요합니다.
eGovFramework의 기존 Inspection XML은 과거 PMD Rule 경로를 사용하는 경우가 있습니다.
예:
```text
rulesets/java/basic.xml
rulesets/java/empty.xml
rulesets/java/braces.xml
```
등입니다.
PMD 6에서는 Rule이 새로운 Category 체계로 이동했지만 **PMD 6 전체에서는 기존 Rule reference에 대한 backward compatibility를 유지했습니다.**
PMD 공식 문서에서도 PMD 6에서는 deprecated된 옛 Rule 이름을 계속 사용할 수 있고, 호환성은 PMD 7 이전까지 유지된다고 설명합니다. 
따라서 PMD 6.45를 사용하는 현재 선택이 중요합니다.
```text
eGovFramework Legacy Ruleset
          ↓
PMD 6.45 compatibility layer
          ↓
실제 PMD 6 Rule
```
형태로 처리됩니다.
실행 중:
```text
The rule "EmptyCatchBlock" has been moved...
```
같은 Warning이 나올 수 있습니다.
이는 **반드시 검사 실패를 의미하는 것은 아닙니다.**
실제로 eGovFramework 3.5 Ruleset을 PMD 6에서 실행하면 moved-rule compatibility warning이 발생하는 사례도 확인됩니다. 
따라서 현재 목적에서는 **PMD 7을 사용하지 말고 PMD 6.45에 고정**하는 편이 안전합니다.
# 7. 먼저 CMD에서 eGov Ruleset 정상 여부 검증
VS Code에 붙이기 전에 PMD 자체가 Ruleset을 읽을 수 있는지 검사하면 문제 해결이 훨씬 쉽습니다.
예:
```cmd
cd C:\DevTools\pmd-bin-6.45.0\bin
```
PMD 6.x Windows에서는:
```cmd
pmd.bat
```
를 사용합니다. PMD 6.45에서도 `pmd.bat`가 공식 실행 스크립트입니다. 
예를 들어:
```cmd
pmd.bat -d D:\workspace\bk-fo\src\main\java ^
-R D:\workspace\bk-fo\config\pmd\EgovInspectionRules_kor-3.5.xml ^
-f text
```
즉:
```text
-d
→ 검사할 Java Source

-R
→ Ruleset XML

-f text
→ 출력 형식
```
입니다.
### 정상
다음처럼 위반사항이 나오면 됩니다.
```text
TestService.java:25:
AvoidReassigningParameters: ...
TestController.java:37:
SystemPrintln: ...
```
또는 이동된 Rule에 대한 Warning과 검사결과가 함께 나올 수 있습니다.
### 비정상
다음이 발생하면 아직 VS Code 설정으로 넘어가지 마십시오.
```text
Cannot load ruleset
Cannot find rule
ClassNotFoundException
Unable to find referenced rule
```
이 경우 Ruleset과 PMD 버전 문제를 먼저 해결해야 합니다.
# 8. 기존 SonarQube for IDE 처리
현재 목적이라면 **삭제해도 됩니다.**
VS Code:
```text
Ctrl + Shift + X
→ SonarQube for IDE
→ Uninstall
```
또는:
```cmd
code --uninstall-extension SonarSource.sonarlint-vscode
```
우선 PMD 테스트만 하려면 삭제 대신:
```text
SonarQube for IDE
→ Disable (Workspace)
```
도 괜찮습니다.
제가 권장하는 순서는:
```text
SonarQube for IDE
→ Disable (Workspace)

PMD 정상 확인

→ 이후 Sonar 필요 없으면 Uninstall
```
입니다.
# 9. 폐쇄망에서 PMD for Java VSIX 설치
인터넷 PC에서 Marketplace에서 VSIX를 내려받아 폐쇄망 PC로 복사합니다.
VS Code:
```text
Ctrl + Shift + P
```
입력:
```text
Extensions: Install from VSIX...
```
다운받은:
```text
cracrayol.pmd-java-xxxx.vsix
```
선택.
설치 확인:
```cmd
code --list-extensions --show-versions | findstr /I pmd
```
예:
```text
cracrayol.pmd-java@0.7.3
```
# 10. VS Code 프로젝트 설정
프로젝트:
```text
D:\workspace\bk-fo
```
에:
```text
.vscode\settings.json
```
을 생성 또는 수정합니다.
## 권장 설정
```json
{
    "javaPMD.pmdBinPath": "C:\\DevTools\\pmd-bin-6.45.0",
    "javaPMD.jrePath": "C:\\DevTools\\jdk-11",
    "javaPMD.rulesets": [
        "config/pmd/EgovInspectionRules_kor-3.5.xml"
    ],
    "javaPMD.runOnFileOpen": false,
    "javaPMD.runOnFileSave": false,
    "javaPMD.runOnFileChange": false,
    "javaPMD.enableCache": false
}
```
PMD for Java가 공식적으로 지원하는 설정 이름입니다. 
각 항목 의미:
|설정|의미|추천|
|---|---|---|
|`javaPMD.pmdBinPath`|사용할 PMD Engine|**6.45.0 지정 필수**|
|`javaPMD.jrePath`|PMD 실행 Java|JDK 11|
|`javaPMD.rulesets`|사용할 Ruleset|eGov Ruleset|
|`runOnFileOpen`|파일 열 때 검사|OFF|
|`runOnFileSave`|저장 시 검사|OFF|
|`runOnFileChange`|수정할 때마다 검사|OFF|
|`enableCache`|`.pmdCache` 생성|초기에는 OFF|
이 Extension 자체에도 PMD가 포함되어 있지만, 현재 목적은 **서버 환경 재현을 위해 6.45.0을 고정하는 것**이므로 반드시:
```json
"javaPMD.pmdBinPath": "C:\\DevTools\\pmd-bin-6.45.0"
```
을 명시하는 것을 권장합니다.
# 11. 왜 자동검사를 모두 OFF하는가
이전부터 원하셨던 사용 방식이:
> 평소에는 검사하지 않고 내가 원할 때만 검사
이므로:
```json
"javaPMD.runOnFileOpen": false,
"javaPMD.runOnFileSave": false,
"javaPMD.runOnFileChange": false
```
가 가장 맞습니다.
필요할 때만 수동 실행합니다.
# 12. 원하는 Java 파일 하나만 검사
Java 파일:
```text
JoinSellerApplicationService.java
```
를 엽니다.
해당 파일에서:
```text
마우스 우클릭
```
하면 PMD 메뉴가 나타납니다.
Extension의 실제 등록 명령은:
```text
PMD for Java: Static analysis on file
```
입니다. 
또는:
```text
Ctrl + Shift + P
```
입력:
```text
PMD
```
선택:
```text
PMD for Java: Static analysis on file
```
실행.
# 13. 프로젝트 전체 검사
```text
Ctrl + Shift + P
→ PMD
```
선택:
```text
PMD for Java: Static analysis on workspace
```
실행합니다.
이 명령 역시 Extension에 공식 등록되어 있습니다. 
## 결과 확인
```text
Ctrl + Shift + M
```
또는:
```text
View
→ Problems
```
여기에:
```text
PMD
```
진단이 표시됩니다.
예:
```text
SystemPrintln
EmptyCatchBlock
UnusedPrivateField
AvoidReassigningParameters
```
등.
# 14. 검사 결과 삭제
Command Palette:
```text
Ctrl + Shift + P
→ PMD for Java: Clear problems
```
Extension이 공식적으로:
```text
Static analysis on workspace
Static analysis on file
Clear problems
```
세 명령을 등록합니다. 
# 15. 정상동작 확인용 테스트 코드
eGovFrame의 대표 Rule을 한 번에 확인하려면 다음 정도가 좋습니다.
```java
package app.sonarcheck;
public class PmdCheckSample {
    private String unusedField;
    public void test(String value) {
        System.out.println("PMD TEST");
        if (value != null) {
        }
        try {
            Integer.parseInt(value);
        } catch (Exception e) {
        }
    }
}
```
예상 가능한 eGov Rule:
|코드|대표 Rule|
|---|---|
|`unusedField`|`UnusedPrivateField`|
|`System.out.println`|`SystemPrintln`|
|빈 `if`|`EmptyIfStmt`|
|빈 `catch`|`EmptyCatchBlock`|
eGovFrame 공식 39개 목록에도 이 Rule들이 포함되어 있습니다. 
실행:
```text
파일 우클릭
→ PMD for Java
→ Static analysis on file
```
Problems에 여러 건이 나오면 기본 구성은 성공입니다.
# 16. 추천 디렉터리 구조
최종적으로 다음 구조를 권장합니다.
```text
C:\DevTools\
└─ pmd-bin-6.45.0\
   ├─ bin\
   ├─ lib\
   └─ ...
D:\workspace\bk-fo\
├─ pom.xml
├─ src\
│   └─ main\
│       └─ java\
├─ config\
│   └─ pmd\
│       └─ EgovInspectionRules_kor-3.5.xml
└─ .vscode\
    └─ settings.json
```
`settings.json`:
```json
{
    "javaPMD.pmdBinPath": "C:\\DevTools\\pmd-bin-6.45.0",
    "javaPMD.jrePath": "C:\\DevTools\\jdk-11",
    "javaPMD.rulesets": [
        "config/pmd/EgovInspectionRules_kor-3.5.xml"
    ],
    "javaPMD.runOnFileOpen": false,
    "javaPMD.runOnFileSave": false,
    "javaPMD.runOnFileChange": false,
    "javaPMD.enableCache": false
}
```
# 17. 한 가지 중요한 확인사항
eGovFramework의 3.8 Ruleset은 **전자정부 표준 39개 Rule이라는 기준에는 적합**하지만, 현재 SonarQube Server에 설정된 PMD Rule과 완전히 1:1 동일하다고 아직 보장할 수는 없습니다.
현재 서버 로그에는 예를 들어:
```text
pmd:UncommentedEmptyMethodBody
```
가 있었는데 eGovFrame 공식 39개 표에는 일부 버전에 따라:
```text
UncommentedEmptyMethod
```
형태로 표현되기도 합니다. 
PMD 버전이 올라가면서 Rule 이름 이동/변경이 있었기 때문입니다.
따라서 최종적으로는:
```text
Server에서 PMD Issue가 많이 발생하는 Java 파일 1개 선정
       ↓
Local PMD 6.45 + eGov Ruleset 검사
       ↓
Server 결과와 비교
```
하는 과정이 필요합니다.
## 현재 환경에서 최종 권장 절차
```text
[인터넷 PC]
1. PMD for Java VSIX 다운로드
2. PMD 6.45.0 ZIP 다운로드
3. egovinspectionrules-3.8.zip 다운로드
        ↓
[폐쇄망 반입]
4. PMD 6.45 압축 해제
   C:\DevTools\pmd-bin-6.45.0
5. eGov Ruleset 프로젝트 config/pmd에 복사
6. SonarQube for IDE Disable
7. PMD for Java VSIX 설치
8. .vscode/settings.json 설정
9. CMD에서 PMD + eGov Ruleset 먼저 테스트
10. VS Code 재시작
11. Java 파일 우클릭
12. PMD for Java: Static analysis on file
13. Problems 확인
14. Workspace 전체가 필요할 때만
    Static analysis on workspace
```
현재 요구사항인 **“서버 환경 수정 없이, 폐쇄망 VS Code에서 내가 원할 때만 eGovFramework PMD Rule로 검사”**에는 이 구성이 가장 단순합니다. 특히 `pmdBinPath`를 명시하여 **Extension 내장 PMD 버전이 아니라 PMD 6.45.0을 강제 사용**하는 것이 핵심입니다. 