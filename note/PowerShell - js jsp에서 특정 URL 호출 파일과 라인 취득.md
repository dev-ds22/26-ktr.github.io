# Windows 11에서 JSP/JS 파일의 `/mypage/seller` 호출 위치 검색
프로젝트 루트 디렉터리에서 실행하면 됩니다.
## 1. PowerShell — 가장 추천
```powershell
Get-ChildItem -Path . -Recurse -File -Include *.jsp,*.js |
    Select-String -SimpleMatch "/mypage/seller"
```
출력 예:
```text
src\main\webapp\WEB-INF\views\mypage\sellerInfo.jsp:125:    url: "/mypage/seller/info.do",
src\main\webapp\js\mypage\seller.js:47:    location.href = "/mypage/seller/list.do";
```
형식은 다음과 같습니다.
```text
파일명:라인번호:해당라인
```
### 결과를 보기 좋게 정리
```powershell
Get-ChildItem -Path . -Recurse -File -Include *.jsp,*.js |
    Select-String -SimpleMatch "/mypage/seller" |
    ForEach-Object {
        "$($_.Path):$($_.LineNumber): $($_.Line.Trim())"
    }
```
예:
```text
D:\workspace\bk-fo\src\main\webapp\WEB-INF\views\seller.jsp:125: url: "/mypage/seller/info.do",
D:\workspace\bk-fo\src\main\webapp\js\seller.js:47: location.href = "/mypage/seller/list.do";
```
## 2. 결과를 TXT 파일로 저장
실무에서는 이 방법이 편합니다.
```powershell
Get-ChildItem -Path . -Recurse -File -Include *.jsp,*.js |
    Select-String -SimpleMatch "/mypage/seller" |
    ForEach-Object {
        "$($_.Path):$($_.LineNumber): $($_.Line.Trim())"
    } |
    Out-File ".\mypage_seller_search.txt" -Encoding utf8
```
결과:
```text
mypage_seller_search.txt
```
## 3. CMD에서 `findstr` 사용
프로젝트 루트에서:
```cmd
findstr /S /N /I /C:"/mypage/seller" *.jsp *.js
```
옵션 의미:
|옵션|의미|
|---|---|
|`/S`|하위 디렉터리까지 검색|
|`/N`|라인 번호 출력|
|`/I`|대소문자 무시|
|`/C:"..."`|문자열 그대로 검색|
출력 예:
```text
src\main\webapp\seller.jsp:125:    url: "/mypage/seller/info.do"
src\main\webapp\js\seller.js:47:    location.href="/mypage/seller/list.do";
```
파일로 저장:
```cmd
findstr /S /N /I /C:"/mypage/seller" *.jsp *.js > mypage_seller_search.txt
```
## 4. 특정 디렉터리만 검색
예를 들어 실제 화면 소스가 다음 경로라면:
```text
D:\workspace\bk-fo\src\main\webapp
```
PowerShell:
```powershell
Get-ChildItem "D:\workspace\bk-fo\src\main\webapp" -Recurse -File -Include *.jsp,*.js |
    Select-String -SimpleMatch "/mypage/seller"
```
## 5. `.jsp`, `.js` 외 `.html`, `.tag`, `.properties`까지 검사
URI 변경 영향도를 조사하는 목적이라면 JSP/JS만 검색하는 것보다 아래처럼 확대하는 것을 권장합니다.
```powershell
Get-ChildItem -Path . -Recurse -File -Include *.jsp,*.js,*.html,*.tag,*.properties,*.xml |
    Select-String -SimpleMatch "/mypage/seller" |
    ForEach-Object {
        "$($_.Path):$($_.LineNumber): $($_.Line.Trim())"
    }
```
## 6. `target`, `.git`, `node_modules` 제외
Java/Spring 프로젝트라면 불필요한 빌드 결과물이 검색되는 것을 피하는 것이 좋습니다.
```powershell
Get-ChildItem -Path . -Recurse -File -Include *.jsp,*.js |
    Where-Object {
        $_.FullName -notmatch '\\target\\|\\\.git\\|\\node_modules\\'
    } |
    Select-String -SimpleMatch "/mypage/seller" |
    ForEach-Object {
        "$($_.FullName):$($_.LineNumber): $($_.Line.Trim())"
    }
```
### 26_KTR 프로젝트 기준 추천 명령
현재처럼 Spring 5.3 + JSP 프로젝트에서 **URI 변경 영향도 조사** 목적이면 이 명령을 추천합니다.
```powershell
Get-ChildItem -Path . -Recurse -File -Include *.jsp,*.js |
    Where-Object {
        $_.FullName -notmatch '\\target\\|\\\.git\\'
    } |
    Select-String -SimpleMatch "/mypage/seller" |
    Sort-Object Path, LineNumber |
    ForEach-Object {
        "{0}:{1}: {2}" -f $_.Path, $_.LineNumber, $_.Line.Trim()
    } |
    Tee-Object -FilePath ".\mypage_seller_search.txt"
```
이렇게 하면 **화면에도 출력되고 동시에 `mypage_seller_search.txt`에도 저장**됩니다.

추가로 URI가 `"/mypage/seller"`, `"${contextPath}/mypage/seller"`, `"<c:url value='/mypage/seller...'>"` 등 여러 형태로 작성되어 있어도 `/mypage/seller` 문자열 자체가 포함되어 있으면 모두 검출됩니다.