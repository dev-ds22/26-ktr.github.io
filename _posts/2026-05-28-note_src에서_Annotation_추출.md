---
layout: single
title: "src에서_Annotation_추출"
excerpt: "src에서_Annotation_추출"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-05-28"
last_modified_at: "2026-05-28 12:08:10 +0900"
---
### 1. java 소스 내 어노테이션 및 메서드명 추출 명령어

**결론:** 정규표현식(Regex)의 완전한 지원 여부에 따라 **PowerShell**에서는 정밀한 패턴 매칭(`Select-String`)을 사용하고, **CMD**에서는 제한된 정규식 기능(`findstr`)을 활용하여 소스 코드 내의 `@어노테이션` 및 `메서드 시그니처` 라인을 추출할 수 있습니다.

어노테이션과 멤버 변수(Field)의 혼선을 줄이고 메서드만 정확하게 골라내기 위해 **PowerShell 방식을 사용하는 것을 강력히 권장**합니다.

### 2. PowerShell을 이용한 정밀 검색 (추천)

PowerShell은 하위 폴더 재귀 탐색(`Get-ChildItem -Recurse`)과 다중 정규식 패턴을 지원하므로, 변수 선언문(`private String name;`)을 배제하고 괄호(`(`)가 포함된 메서드 정의부와 어노테이션만 정확히 찾아낼 수 있습니다.

#### 2-1. 화면 출력 및 파일 저장 명령어

PowerShell

```
Get-ChildItem -Recurse -Filter *.java | Select-String -Pattern "^\s*@\w+", "^\s*(public|protected|private|static)\s+[\w\<\>\[\]]+\s+\w+\s*\(" | ForEach-Object { "$($_.Path) [Line $($_.LineNumber)]: $($_.Line.Trim())" } > ".\소스분석_결과.txt"
```

#### 2-2. 🔍 정규표현식 패턴 매커니즘 설명

- `^\s*@\w+` : 라인 시작 부분에 공백이 있고 그 뒤에 `@`와 단어(어노테이션명)가 오는 라인을 매칭합니다.

- `^\s*(public|...)\s+[\w\<\>\[\]]+\s+\w+\s*\(` : 접근 제어자 뒤에 리턴 타입(제네릭 및 배열 포함)이 오고, 메서드명 뒤에 **열는 괄호 `(`** 가 반드시 존재하는 라인만 필터링하여 멤버 변수 선언문과 메서드를 구별합니다.

### 3. 명령 프롬프트 (CMD)를 이용한 약식 검색

CMD의 `findstr`은 정규식 기능이 극히 제한적(그룹화 `()`, 수량자 `+`, `?` 등 사용 불가)이므로, 다중 와일드카드 매칭 조건`/c:`를 여러 번 지정하는 방식으로 근사치를 추출합니다.

#### 3-1. 화면 출력 및 파일 저장 명령어

DOS

```
findstr /s /r /c:"^[ ]*@" /c:"^[ ]*public.*(" /c:"^[ ]*private.*(" /c:"^[ ]*protected.*(" *.java > 소스분석_결과_cmd.txt"
```

#### 3-2. ⚠️ CMD 방식의 한계점

- 리턴 타입이 없는 생성자(Constructor) 중 `public 클래스명()` 구조는 메서드와 함께 추출됩니다.

- 인터페이스 내의 제어자가 생략된 메서드(`String getName();`) 등은 패턴에서 누락될 수 있습니다.

### 4. 도구별 기능 및 추출 정확도 비교

|**비교 항목**|**PowerShell (Select-String)**|**명령 프롬프트 (findstr)**|
|---|---|---|
|**정확도**|**높음** (메서드와 변수의 정밀 분리 가능)|**보통** (괄호가 포함된 제어자 라인 전체 매칭)|
|**하위 폴더 탐색**|`Get-ChildItem -Recurse` 조합|`/s` 옵션으로 자체 지원|
|**제네릭 타입 인식**|`List<String>` 등 정규식 매칭 가능|`.*` 와일드카드로 매칭 (오탐 가능성 있음)|
|**결과 가독성**|파일 절대경로, 라인 번호, 공백 제거 정렬 가능|파일 상대경로와 텍스트 원본 그대로 출력|

### 5. 💡 [난이도 상] 아키텍처 관점의 정적 분석 한계와 대안

상기 셸 스크립트 방식은 **텍스트 기반 정적 매칭**입니다. 따라서 아래와 같은 아키텍처적 예외 케이스에서는 정확도가 떨어질 수 있습니다.

1. **멀티라인 어노테이션:** `@GetMapping(value = "/list", produces = "application/json")` 과 같이 하나의 어노테이션이 여러 줄에 걸쳐 선언된 경우 첫 줄만 잡히게 됩니다.

2. **주석 처리된 코드:** `// @Override` 또는 `/* public void test() */` 처럼 주석 처리된 코드도 정규식에 매칭되어 결과 파일에 포함됩니다.

**🛠️ 아키텍처적 대안 권장:**

프로젝트 내 전체 API 엔드포인트 맵핑 현황이나 빈(Bean) 구조 파악이 목적이라면, 런타임 시점에 Spring의 `ApplicationContext` 내부 인터페이스를 활용하거나 Spring Boot Actuator (`/mappings` 엔드포인트)를 활성화하여 JSON 데이터로 정밀하게 수집하는 것이 아키텍처 관리 측면에서 가장 확실한 방법입니다.

#### 5-1. Annotation  취득

Spring 5.3 기반 Java 소스에서 아래 3가지를 파일로 추출합니다.

|파일|내용|
|---|---|
|`java-annotations.csv`|전체 어노테이션 목록|
|`java-methods.csv`|전체 메서드명 목록|
|`java-method-annotations.csv`|메서드명과 바로 위에 선언된 어노테이션 매핑|
|`java-annotation-count.txt`|어노테이션별 사용 개수|
|`java-method-prefix-count.txt`|메서드 접두어별 사용 개수|

#### 5-2. PowerShell 권장 방식

프로젝트 루트에서 아래 파일을 생성합니다.

```text
scan-java-spring.ps1
```

```powershell
$root = Get-Location
$outDir = Join-Path $root "java-scan-output"
New-Item -ItemType Directory -Force -Path $outDir | Out-Null
$javaFiles = Get-ChildItem -Path $root -Recurse -Filter "*.java" -File |
    Where-Object {
        $_.FullName -cnotmatch '\\(target|build|out|\.git|\.gradle)\\'
    }
$annotationRegex = '^\s*@([A-Za-z_][A-Za-z0-9_\.]*)'
$methodRegex = '^\s*(?:(?:public|protected|private)\s+)?(?:(?:static|final|synchronized|abstract|native|strictfp)\s+)*(?:<[^>]+>\s+)?(?:[\w$<>\[\]?.,]+\s+)+([A-Za-z_$][A-Za-z0-9_$]*)\s*\([^;]*\)\s*(?:throws\s+[\w$.,\s]+)?\s*(?:\{|;)\s*$'
$annotations = New-Object System.Collections.Generic.List[object]
$methods = New-Object System.Collections.Generic.List[object]
$methodAnnotations = New-Object System.Collections.Generic.List[object]
foreach ($file in $javaFiles) {
    $lines = Get-Content -LiteralPath $file.FullName
    $pendingAnnotations = New-Object System.Collections.Generic.List[string]
    $inAnnotationBlock = $false
    for ($i = 0; $i -lt $lines.Count; $i++) {
        $lineNo = $i + 1
        $line = $lines[$i]
        $trim = $line.Trim()
        if ($trim -cmatch $annotationRegex) {
            $annotationName = $matches[1]
            $annotations.Add([PSCustomObject]@{
                File = $file.FullName
                Line = $lineNo
                Annotation = $annotationName
                Text = $trim
            })
            $pendingAnnotations.Add($annotationName)
            if ($trim -cmatch '\(' -and $trim -cnotmatch '\)') {
                $inAnnotationBlock = $true
            }
            continue
        }
        if ($inAnnotationBlock) {
            if ($trim -cmatch '\)') {
                $inAnnotationBlock = $false
            }
            continue
        }
        if ($trim -eq "" -or $trim -cmatch '^(//|\*|/\*)') {
            continue
        }
        if ($trim -cmatch $methodRegex) {
            $methodName = $matches[1]
            $prefix = $methodName -creplace '^([a-z]+).*$', '$1'
            $methods.Add([PSCustomObject]@{
                File = $file.FullName
                Line = $lineNo
                Method = $methodName
                Prefix = $prefix
                Text = $trim
            })
            $methodAnnotations.Add([PSCustomObject]@{
                File = $file.FullName
                Line = $lineNo
                Method = $methodName
                Prefix = $prefix
                Annotations = ($pendingAnnotations -join ",")
                Text = $trim
            })
            $pendingAnnotations.Clear()
            continue
        }
        $pendingAnnotations.Clear()
    }
}
$annotations |
    Export-Csv -Path (Join-Path $outDir "java-annotations.csv") -NoTypeInformation -Encoding UTF8
$methods |
    Export-Csv -Path (Join-Path $outDir "java-methods.csv") -NoTypeInformation -Encoding UTF8
$methodAnnotations |
    Export-Csv -Path (Join-Path $outDir "java-method-annotations.csv") -NoTypeInformation -Encoding UTF8
$annotations |
    Group-Object Annotation |
    Sort-Object Count -Descending |
    ForEach-Object { "$($_.Name)`t$($_.Count)" } |
    Set-Content -Path (Join-Path $outDir "java-annotation-count.txt") -Encoding UTF8
$methods |
    Group-Object Prefix |
    Sort-Object Count -Descending |
    ForEach-Object { "$($_.Name)`t$($_.Count)" } |
    Set-Content -Path (Join-Path $outDir "java-method-prefix-count.txt") -Encoding UTF8
Write-Host "완료: $outDir"
```

#### 5-3. PowerShell 실행

프로젝트 루트에서 실행합니다.

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\scan-java-spring.ps1
```

PowerShell 7을 사용한다면 아래도 가능합니다.

```powershell
pwsh -NoProfile -ExecutionPolicy Bypass -File .\scan-java-spring.ps1
```

#### 5-4. CMD에서 실행

CMD에서는 PowerShell 스크립트를 호출하는 방식이 가장 안전합니다.

```cmd
powershell -NoProfile -ExecutionPolicy Bypass -File .\scan-java-spring.ps1
```

#### 5-5. 출력 결과 예시

##### 5-5-1. `java-annotations.csv`

```csv
"File","Line","Annotation","Text"
"C:\project\src\main\java\com\test\OrderService.java","25","Service","@Service"
"C:\project\src\main\java\com\test\OrderService.java","60","Transactional","@Transactional"
```

##### 5-5-2. `java-methods.csv`

```csv
"File","Line","Method","Prefix","Text"
"C:\project\src\main\java\com\test\OrderService.java","61","selectOrderList","select","public List<OrderVO> selectOrderList(OrderVO vo) {"
"C:\project\src\main\java\com\test\OrderService.java","90","insertOrder","insert","public int insertOrder(OrderVO vo) {"
```

##### 5-5-3. `java-method-annotations.csv`

```csv
"File","Line","Method","Prefix","Annotations","Text"
"C:\project\src\main\java\com\test\OrderService.java","61","selectOrderList","select","Override,Transactional","public List<OrderVO> selectOrderList(OrderVO vo) {"
"C:\project\src\main\java\com\test\OrderService.java","90","insertOrder","insert","Transactional","public int insertOrder(OrderVO vo) {"
```

#### 5-6. CMD 단독 검색용 간단 명령

CMD만으로는 정교한 메서드명 추출이 어렵습니다. 단순히 어노테이션 라인과 메서드 선언 후보 라인을 찾는 정도로만 사용하는 것이 좋습니다.

##### 5-6-1. 전체 어노테이션 라인 검색

```cmd
findstr /S /N /R /C:"^[ ]*@[A-Za-z_]" *.java > java-annotation-lines.txt
```

##### 5-6-2. 메서드 선언 후보 라인 검색

```cmd
findstr /S /N /R /C:"^[ ]*public .*(" /C:"^[ ]*protected .*(" /C:"^[ ]*private .*(" *.java > java-method-candidate-lines.txt
```

이 방식은 `public`, `protected`, `private`이 없는 package-private 메서드나 여러 줄로 나뉜 메서드 선언은 누락될 수 있습니다. 따라서 실제 분석용으로는 PowerShell 스크립트를 권장합니다.

#### 5-7. 검증 포인트

|검증 항목|적용 내용|
|---|---|
|대소문자 구분|`-cmatch`, `-cnotmatch`, `-creplace` 사용|
|접두어 추출|`getAddingGoodsList → get`, `updateOrderStatus → update`|
|불필요 폴더 제외|`target`, `build`, `out`, `.git`, `.gradle` 제외|
|어노테이션 추출|`@Service`, `@Transactional`, `@Autowired`, `@Override` 등 추출|
|메서드 매핑|메서드 바로 위에 붙은 어노테이션을 `Annotations` 컬럼에 표시|
|트랜잭션 검토|`Prefix`, `Annotations` 기준으로 `select*`, `insert*`, `update*`, `delete*` 분류 가능|

#### 5-8. 샘플 검증

|입력 메서드명|기대 접두어|스크립트 결과|
|---|---|---|
|`getAddingGoodsList`|`get`|`get`|
|`selectMemberList`|`select`|`select`|
|`insertOrder`|`insert`|`insert`|
|`updateStaticVariableJavascriptJsonFile`|`update`|`update`|
|`sendMssg`|`send`|`send`|

#### 5-9. AA 관점 활용 방법

`java-method-prefix-count.txt`를 먼저 보고 트랜잭션 룰 후보를 정리하는 것이 좋습니다.

```text
select    120
get       90
insert    40
update    35
delete    12
send      8
process   5
```

이후 `java-method-annotations.csv`에서 `@Transactional`, `@Cacheable`, `@Async`, `@Scheduled`, `@Override`가 붙은 메서드를 따로 확인하면 됩니다. 특히 `send*`, `mail*`, `process*`, `save*` 계열은 단순 CRUD가 아닐 가능성이 있으므로 자동 트랜잭션 룰에 넣기 전에 별도 검토가 필요합니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
