---
layout: single
title: "log_파일_filtering"
excerpt: "log_파일_filtering"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-05-27"
last_modified_at: "2026-05-27 17:26:38 +0900"
---
## 1. 기초

- 윈도우 11에서 로그(.log) 파일의 특정 문구가 포함된 라인만 추출하는 가장 빠르고 효율적인 방법은 명령 프롬프트(CMD)의 `findstr` 명령어나 **PowerShell**의 `Select-String` 시스템을 사용하는 것입니다.
### 1-1. 명령 프롬프트 (CMD) 사용하기
- 가장 전통적이고 가벼운 방법입니다. `findstr` 명령어를 사용합니다.

#### 1-1-1. 텍스트 화면에 바로 출력하기

```
findstr "특정문구" "C:\경로\로그파일.log"
```

#### 1-1-2. 추출한 라인만 새 파일로 저장하기 (`>`)
- 결과를 눈으로 보는 대신 새로운 파일로 저장하고 싶다면 명령어 끝에 `> 저장할파일명.txt`를 붙여줍니다.

```
findstr "특정문구" "C:\경로\로그파일.log" > "C:\경로\추출된라인.log"
```

#### 1-1-3. 💡 CMD 유용한 팁

- **대소문자 구분 없이** 찾고 싶다면 `/i` 옵션을 추가하세요.

    ```
    findstr /i "error" "로그파일.log"
    ```

- **여러 개 키워드**를 한 번에 찾고 싶다면 공백으로 구분합니다. (예: error 또는 fail이 포함된 라인)

    ```
    findstr /i "error fail" "로그파일.log"
    ```

### 1-2. PowerShell 사용하기

윈도우 11의 기본 터미널 환경인 PowerShell에서는 `Select-String` 명령어를 사용하여 더 강력한 텍스트 검색이 가능합니다.

#### 1-2-1. 텍스트 화면에 바로 출력하기

```
Select-String -Path "C:\경로\로그파일.log" -Pattern "특정문구"
```

#### 1-2-2. 추출한 라인만 깨끗하게 새 파일로 저장하기

PowerShell에서 단순히 저장하면 행 번호 등의 부가 정보가 같이 저장될 수 있으므로, 내용(라인)만 깔끔하게 뽑아 저장하려면 아래와 같이 입력합니다.

```
Select-String -Path "C:\경로\로그파일.log" -Pattern "특정문구" | ForEach-Object { $_.Line } > "C:\경로\추출된라인.log"
```

#### 1-2-3. 💡 PowerShell 유용한 팁

- PowerShell의 `Select-String`은 **기본적으로 대소문자를 구분하지 않습니다.** * 만약 **대소문자를 엄격하게 구분**하고 싶다면 `-CaseSensitive` 옵션을 추가하세요.

    ```
    Select-String -Path "로그파일.log" -Pattern "Error" -CaseSensitive
    ```

## 2. 매칭 및 필터링

요청하신 조건처럼 'default task-숫자'와 같이 중간에 변하는 숫자(쓰레드 번호)가 포함된 특정 패턴을 매칭하고, 중복되는 라인을 효율적으로 필터링하기 위해서는 정규표현식(Regular Expression)을 사용해야 합니다.

윈도우 11 환경에서 **PowerShell**과 CMD(명령 프롬프트)를 이용해 이를 해결하는 방법입니다. 중복 로그가 많다고 하셨으므로, **중복을 제거(Deduplication)하는 방법까지 지원하는 PowerShell 방식을 강력히 추천**합니다.

---

### 2-1. 방법 1: PowerShell 사용하기 (강력 추천)

PowerShell은 완전한 정규표현식을 지원하며, 같은 내용의 로그가 반복될 때 이를 하나만 남기고 정렬해 주는 기능(`Sort-Object -Unique`)을 제공하므로 현재 상황에 가장 적합합니다.

#### 2-1-1. ① 단순 패턴 매칭 후 새 파일로 저장

- 숫자 부분을 `\d+`(하나 이상의 숫자)로 매칭하여 해당 라인만 추출합니다.

```powershell
Select-String -Path ".\원본로그파일.log" -Pattern "default task-\d+.*org\.springframework\.transaction\.CannotCreateTransactionException" | ForEach-Object { $_.Line } > ".\필터링완료.log"
```

#### 2-1-2. ② 💡 중복 라인 제거까지 한 번에 처리 (추천)

비슷한 시간대에 동일한 쓰레드에서 완전히 동일한 에러 문구가 여러 번 출력되어 **딱 한 줄만 남기고 싶다면**, 아래와 같이 `Sort-Object -Unique` 명령어를 중간에 추가합니다.

```powershell
Select-String -Path ".\원본로그파일.log" -Pattern "default task-\d+.*org\.springframework\.transaction\.CannotCreateTransactionException" | ForEach-Object { $_.Line } | Sort-Object -Unique > ".\필터링완료_중복제거.log"
```

* **패턴 설명:** * `default task-\d+` : `default task-` 뒤에 숫자가 붙은 패턴을 찾습니다.
* `.*` : 중간에 어떤 문자(시간, 공백 등)가 와도 상관없이 연결합니다.
* `org\.springframework...` : 찾고자 하는 구체적인 예외 클래스명입니다. (`.`은 정규식에서 특수문자이므로 `\.`로 처리)

---

### 2-2. 방법 2: 명령 프롬프트 (CMD) 사용하기

- CMD의 `findstr` 명령어로도 가능하지만, CMD는 정규표현식 기능이 제한적입니다. 숫자 매칭을 위해 `\d+` 대신 `[0-9][0-9]*` 패턴을 사용해야 합니다.
#### 2-2-1. 단순 패턴 매칭 후 새 파일로 저장

```cmd
findstr /r "default task-[0-9][0-9]*.*org\.springframework\.transaction\.CannotCreateTransactionException" "원본로그파일.log" > "필터링완료.log"
```

* **주의점:** CMD(`findstr`)는 자체적으로 중복 라인을 하나로 합쳐주는 기능(Unique)이 없습니다. 만약 완전히 중복된 라인을 깔끔하게 한 줄만 남기고 필터링하고 싶으시다면 위의 **PowerShell 방법 ②**를 사용하시는 것이 훨씬 효율적입니다.
- **사용 팁:** 작업하려는 로그 파일이 있는 폴더에서 `Shift + 마우스 우클릭`을 누른 후 **"여기에 PowerShell 창 열기"** 또는 "터미널에서 열기"를 선택하면 경로 입력 없이 파일명만 적어서 바로 실행할 수 있어 편리합니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
