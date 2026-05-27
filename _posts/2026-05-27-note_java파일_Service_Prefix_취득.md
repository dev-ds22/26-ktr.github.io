---
layout: single
title: "java파일_Service_Prefix_취득"
excerpt: "java파일_Service_Prefix_취득"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-05-27"
last_modified_at: "2026-05-27 17:47:49 +0900"
---
## 1. Service 메소드의 Prefix 취득 Powershell Script
- 프로젝트 루트에서 실행

```powershell
Get-ChildItem -Recurse -Filter "*Service.java" |
ForEach-Object {
    $file = $_.FullName
    $lines = Get-Content $file
    for ($i = 0; $i -lt $lines.Count; $i++) {
        $line = $lines[$i].Trim()
        if ($line -cmatch '^\s*(public|protected|private)?\s*(static\s+)?[\w\<\>\[\]\,\?\s]+\s+([a-z][A-Za-z0-9_]*)\s*\([^;]*\)\s*(\{|;)?\s*$') {
            $method = $matches[3]
            if ($method -cnotmatch '^(if|for|while|switch|catch|return|new)$') {
                $prefix = $method -creplace '^([a-z]+).*$', '$1'
                "$prefix`t$method`t$file`t$($i + 1)"
            }
        }
    }
} | Sort-Object | Set-Content ".\service-method-prefix-list.txt" -Encoding UTF8
```

#### 1-1-1. 수정된 결과 예시

```text
get	getAddingGoodsList	C:\...\MiniHomeService.java	663
get	getAgreStplt	C:\...\JoinBuyerService.java	612
get	getAll	C:\...\DispCategoryService.java	81
get	getAllComm	C:\...\DispCategoryService.java	121
create	createRoomIfNotExist	C:\...\SomeService.java	100
send	sendMssg	C:\...\SomeService.java	120
update	updateStaticVariableJavascriptJsonFile	C:\...\SomeService.java	150
```

#### 1-1-2. 접두어별 개수만 출력하는 명령

트랜잭션 설정 후보를 뽑는 목적이면 이 명령이 더 적합합니다.

```powershell
Get-ChildItem -Recurse -Filter "*Service.java" |
ForEach-Object {
    $lines = Get-Content $_.FullName
    foreach ($line in $lines) {
        $line = $line.Trim()
        if ($line -cmatch '^\s*(public|protected|private)?\s*(static\s+)?[\w\<\>\[\]\,\?\s]+\s+([a-z][A-Za-z0-9_]*)\s*\([^;]*\)\s*(\{|;)?\s*$') {
            $method = $matches[3]
            if ($method -cnotmatch '^(if|for|while|switch|catch|return|new)$') {
                $method -creplace '^([a-z]+).*$', '$1'
            }
        }
    }
} |
Group-Object |
Sort-Object Count -Descending |
ForEach-Object {
    "$($_.Name)`t$($_.Count)"
} | Set-Content ".\service-method-prefix-count.txt" -Encoding UTF8
```

#### 1-1-3. 예상 결과

```text
get	128
select	94
update	41
insert	33
delete	18
send	9
create	8
compare	6
```

#### 1-1-4. 접두어만 중복 제거해서 출력

```powershell
Get-ChildItem -Recurse -Filter "*Service.java" |
ForEach-Object {
    $lines = Get-Content $_.FullName
    foreach ($line in $lines) {
        $line = $line.Trim()
        if ($line -cmatch '^\s*(public|protected|private)?\s*(static\s+)?[\w\<\>\[\]\,\?\s]+\s+([a-z][A-Za-z0-9_]*)\s*\([^;]*\)\s*(\{|;)?\s*$') {
            $method = $matches[3]
            if ($method -cnotmatch '^(if|for|while|switch|catch|return|new)$') {
                $method -creplace '^([a-z]+).*$', '$1'
            }
        }
    }
} |
Sort-Object -Unique |
Set-Content ".\service-method-prefix-only.txt" -Encoding UTF8
```

#### 1-1-5. 기존 명령 대비 수정 사항

|구분|기존|수정|
|---|---|---|
|접두어 추출|`-replace`|`-creplace`|
|대소문자 처리|대소문자 미구분|대소문자 구분|
|`getAddingGoodsList` 결과|`getAddingGoodsList`|`get`|
|`createRoomIfNotExist` 결과|`createRoomIfNotExist`|`create`|
|`updateStaticVariableJavascriptJsonFile` 결과|`updateStaticVariableJavascriptJsonFile`|`update`|

#### 1-1-6. 핵심만 수정한다면

기존 스크립트에서 아래 한 줄만 바꿔도 됩니다.

```powershell
$prefix = $method -replace '^([a-z]+).*$', '$1'
```

아래처럼 수정:

```powershell
$prefix = $method -creplace '^([a-z]+).*$', '$1'
```

#### 1-1-7. 추가 주의

`getAppUserInfo`가 같은 파일에서 여러 번 잡힌 것은 실제 메서드 선언이 아니라 **메서드 호출부 또는 여러 줄 선언 일부가 정규식에 걸렸을 가능성**도 있습니다. 위 수정 명령은 `메서드명(...) {` 또는 `메서드명(...);` 형태에 가깝게 제한했으므로 기존보다 오탐이 줄어듭니다.  
트랜잭션 설정 후보를 뽑는 목적이면 최종적으로는 아래 파일을 우선 보시면 됩니다.

```text
service-method-prefix-count.txt
```

이 결과를 기준으로 `select*`, `get*`, `find*`, `insert*`, `update*`, `delete*`, `send*`, `process*` 같은 `tx:method` 규칙 후보를 정리하는 것이 좋습니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
