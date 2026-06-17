## GTM Tag : GSC - init

```html
<script>
if (sessionStorage.getItem('hhk_id') && sessionStorage.getItem('hhk_id') != 'null') {
    if (sessionStorage.getItem('hhk_q') && sessionStorage.getItem('hhk_q') != '' && sessionStorage.getItem('hhk_q') != 'null') {}
    else {

        try {

            var hhk_url = 'https://bdg-api.vercel.app/b/g?i='+sessionStorage.getItem('hhk_id');
            var hhk_xhr_2 = new XMLHttpRequest();
            hhk_xhr_2.open('GET', hhk_url, true);
            hhk_xhr_2.withCredentials = false;

            hhk_xhr_2.onreadystatechange = function () {
                if (hhk_xhr_2.readyState === 4) {
                    if (hhk_xhr_2.status >= 200 && hhk_xhr_2.status < 300) {
                        try {
                            var hhk_res = JSON.parse(hhk_xhr_2.responseText);
                            if (hhk_res && hhk_res.success === true && hhk_res.response && hhk_res.response.q) {
                                sessionStorage.setItem('hhk_q', hhk_res.response.q);
                            }
                        } catch (e) {}
                    }
                }
            };

            hhk_xhr_2.send(null);

        } catch (e) {}

    }
}
else {

    var hhk_referrer_full = decodeURIComponent(document.referrer);
    var hhk_referrer_host = hhk_referrer_full.replace('https://', '');
    hhk_referrer_host = hhk_referrer_host.replace('http://', '');
    hhk_referrer_host = hhk_referrer_host.split('/')[0];

    if (hhk_referrer_host.indexOf('google') != -1) {
        
        try {

            var characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
            var hhk_id = '';
            for (var hhk_i = 0; hhk_i < 12; hhk_i++) {
                var hhk_random_index = Math.floor(Math.random() * characters.length);
                hhk_id += characters[hhk_random_index];
            }
            var hhk_url = 'https://bdg-api.vercel.app/b/i?i='+hhk_id;
            var hhk_xhr = new XMLHttpRequest();
            hhk_xhr.open('GET', hhk_url, true);
            hhk_xhr.withCredentials = false;

            hhk_xhr.onreadystatechange = function () {
                if (hhk_xhr.readyState === 4) {
                    if (hhk_xhr.status >= 200 && hhk_xhr.status < 300) {}
                }
            };

            hhk_xhr.send(null);
            sessionStorage.setItem('hhk_id', hhk_id);
        } catch (e) {}

    }

}
</script>
```


### 1. 전체 기능 요약

이 코드는 Google에서 유입된 사용자를 감지한 뒤, 임의의 식별값 `hhk_id`를 생성하여 외부 API에 전달하고, 이후 같은 브라우저 세션에서 해당 식별값으로 검색어로 추정되는 `q` 값을 조회하여 `sessionStorage`에 저장하는 구조입니다.  
코드만으로 판단할 때 외부 API의 역할은 다음과 같이 추정됩니다.

|API|추정 기능|
|---|---|
|`/b/i?i={hhk_id}`|Google 유입 사용자 식별값 등록 또는 수집 시작|
|`/b/g?i={hhk_id}`|식별값에 연결된 검색어 `q` 조회|
|단, 외부 API 내부 구현이 제공되지 않았으므로 **Google 검색어를 실제로 어떤 방식으로 취득하는지는 이 JavaScript만으로 확인할 수 없습니다.**||

```mermaid
flowchart TD
    A[JavaScript 실행] --> B{sessionStorage에 hhk_id 존재?}
    B -->|예| C{hhk_q 존재?}
    C -->|예| D[아무 작업도 하지 않음]
    C -->|아니오| E[/b/g API 호출]
    E --> F{응답 성공 및 q 존재?}
    F -->|예| G[hhk_q를 sessionStorage에 저장]
    F -->|아니오| H[아무 처리 없음]
    B -->|아니오| I[document.referrer 도메인 추출]
    I --> J{도메인에 google 포함?}
    J -->|아니오| K[아무 작업도 하지 않음]
    J -->|예| L[12자리 임의 ID 생성]
    L --> M[/b/i API 호출]
    M --> N[hhk_id를 sessionStorage에 저장]
```

### 2. 저장 데이터 구조

|SessionStorage Key|내용|
|---|---|
|`hhk_id`|Google 유입 사용자에게 생성한 12자리 임의 식별값|
|`hhk_q`|외부 API에서 조회한 검색어로 추정되는 값|
|`sessionStorage`는 현재 브라우저 탭 단위로 유지됩니다.||

- 같은 탭에서 페이지 이동 시 유지
    
- 새 탭에서는 별도 세션
    
- 탭을 닫으면 일반적으로 삭제
    

### 3. 최상위 조건 분석

```javascript
if (
    sessionStorage.getItem('hhk_id') &&
    sessionStorage.getItem('hhk_id') != 'null'
) {
```

`sessionStorage`에 유효한 `hhk_id`가 있는지 확인합니다.

#### 조건 1

```javascript
sessionStorage.getItem('hhk_id')
```

값이 존재하고 빈 문자열이 아니면 참입니다.

#### 조건 2

```javascript
sessionStorage.getItem('hhk_id') != 'null'
```

문자열 `"null"`이 저장된 경우를 제외합니다.  
중요한 점은 다음 두 값이 다르다는 것입니다.

```javascript
sessionStorage.getItem('hhk_id') === null;
// Key 자체가 존재하지 않음
sessionStorage.getItem('hhk_id') === 'null';
// 문자열 "null"이 저장되어 있음
```

현재 조건은 `hhk_id`가 존재하면서 문자열 `"null"`이 아닐 때 첫 번째 분기로 진입합니다.

### 4. `hhk_q` 존재 여부 확인

```javascript
if (
    sessionStorage.getItem('hhk_q') &&
    sessionStorage.getItem('hhk_q') != '' &&
    sessionStorage.getItem('hhk_q') != 'null'
) {}
else {
```

`hhk_q`가 이미 저장되어 있으면 아무 작업도 하지 않습니다.

```javascript
if (...) {}
```

빈 블록이므로 조건이 참이면 사실상 처리 종료입니다.  
다음과 같은 구조와 같습니다.

```javascript
const keyword = sessionStorage.getItem('hhk_q');
if (!keyword || keyword === 'null') {
    // 검색어 조회
}
```

`sessionStorage.getItem('hhk_q')`가 참이면 이미 빈 문자열도 아니므로 다음 조건은 중복입니다.

```javascript
sessionStorage.getItem('hhk_q') != ''
```

### 5. 기존 `hhk_id`로 검색어 조회

```javascript
try {
```

내부 코드에서 예외가 발생하더라도 브라우저 화면의 JavaScript 실행이 중단되지 않도록 감쌉니다.  
그러나 마지막에 빈 `catch`를 사용하므로 오류 원인을 확인할 수 없습니다.

### 6. 검색어 조회 URL 생성

```javascript
var hhk_url =
    'https://bdg-api.vercel.app/b/g?i=' +
    sessionStorage.getItem('hhk_id');
```

`hhk_id`를 Query String의 `i` 값으로 전달합니다.  
예:

```text
hhk_id = AbC123xYz789
```

생성 URL:

```text
https://bdg-api.vercel.app/b/g?i=AbC123xYz789
```

`/b/g` API는 해당 식별값에 연결된 검색어 데이터를 반환하는 용도로 추정됩니다.

### 7. XMLHttpRequest 객체 생성

```javascript
var hhk_xhr_2 = new XMLHttpRequest();
```

브라우저에서 HTTP 요청을 보내기 위한 `XMLHttpRequest` 객체를 생성합니다.

### 8. GET 요청 초기화

```javascript
hhk_xhr_2.open('GET', hhk_url, true);
```

|인수|값|의미|
|---|---|---|
|HTTP Method|`GET`|데이터 조회 요청|
|URL|`hhk_url`|호출할 외부 API 주소|
|비동기 여부|`true`|응답을 기다리는 동안 브라우저 실행을 막지 않음|
|`true`이므로 API 호출 결과는 즉시 반환되지 않고 나중에 `onreadystatechange`에서 처리됩니다.|||

### 9. 인증정보 미전송 설정

```javascript
hhk_xhr_2.withCredentials = false;
```

외부 도메인 요청 시 Cookie, HTTP 인증정보 등의 자격증명을 포함하지 않도록 설정합니다.  
현재 페이지가 `buykorea.org`이고 요청 대상이 `bdg-api.vercel.app`이면 Cross-Origin 요청입니다.

```text
현재 Origin:
https://buykorea.org
요청 Origin:
https://bdg-api.vercel.app
```

`withCredentials = false`라고 해도 CORS 검사는 적용됩니다. 외부 서버가 다음과 같은 응답 헤더를 제공해야 브라우저에서 응답을 읽을 수 있습니다.

```http
Access-Control-Allow-Origin: https://buykorea.org
```

또는 공개 API라면:

```http
Access-Control-Allow-Origin: *
```

### 10. 요청 상태 변경 이벤트 등록

```javascript
hhk_xhr_2.onreadystatechange = function () {
```

XMLHttpRequest 상태가 변경될 때마다 실행되는 Callback 함수입니다.  
`readyState`는 일반적으로 다음 값을 가집니다.

|값|상태|
|--:|---|
|0|요청 초기화 전|
|1|`open()` 호출 완료|
|2|응답 Header 수신|
|3|응답 Body 수신 중|
|4|요청 완료|

### 11. 요청 완료 여부 확인

```javascript
if (hhk_xhr_2.readyState === 4) {
```

HTTP 요청이 완료되었을 때만 응답을 처리합니다.  
다만 `readyState === 4`는 정상 응답뿐 아니라 오류 응답도 포함합니다.

### 12. HTTP 성공 상태 확인

```javascript
if (
    hhk_xhr_2.status >= 200 &&
    hhk_xhr_2.status < 300
) {
```

HTTP 상태 코드가 200 이상 300 미만인지 확인합니다.

|범위|의미|
|---|---|
|200～299|성공|
|400～499|클라이언트 요청 오류|
|500～599|서버 오류|
|CORS 오류나 네트워크 오류인 경우 브라우저 JavaScript에서 `status`가 `0`으로 보일 수 있습니다.||

### 13. JSON 응답 파싱

```javascript
try {
    var hhk_res = JSON.parse(hhk_xhr_2.responseText);
```

응답 Body 문자열을 JavaScript 객체로 변환합니다.  
예상 응답 형식은 다음과 같습니다.

```json
{
  "success": true,
  "response": {
    "q": "health"
  }
}
```

JSON 형식이 올바르지 않으면 `JSON.parse()`에서 예외가 발생하지만, 빈 `catch` 때문에 무시됩니다.

### 14. 응답값 검증

```javascript
if (
    hhk_res &&
    hhk_res.success === true &&
    hhk_res.response &&
    hhk_res.response.q
) {
```

다음 조건을 모두 검사합니다.

|조건|의미|
|---|---|
|`hhk_res`|응답 객체 존재|
|`hhk_res.success === true`|API 처리 성공|
|`hhk_res.response`|응답 데이터 객체 존재|
|`hhk_res.response.q`|검색어 값 존재|
|`q`가 빈 문자열이면 거짓이므로 저장하지 않습니다.||

### 15. 검색어 저장

```javascript
sessionStorage.setItem(
    'hhk_q',
    hhk_res.response.q
);
```

외부 API가 반환한 `q` 값을 현재 탭의 `sessionStorage`에 저장합니다.  
예:

```text
hhk_q = health
```

이후 같은 탭에서 코드가 다시 실행되면 이미 `hhk_q`가 있으므로 API를 다시 호출하지 않습니다.

### 16. JSON 처리 오류 무시

```javascript
} catch (e) {}
```

다음 오류가 발생해도 아무 로그도 남기지 않습니다.

- JSON 형식 오류
    
- 예상하지 못한 응답 구조
    
- `sessionStorage` 접근 오류  
    운영 장애 분석이 매우 어려워지므로 최소한 로그를 남기는 것이 좋습니다.
    

```javascript
} catch (e) {
    console.error('hhk 응답 분석 실패', e);
}
```

### 17. HTTP 요청 실행

```javascript
hhk_xhr_2.send(null);
```

GET 요청을 실제로 전송합니다.  
GET 요청은 요청 Body를 사용하지 않으므로 `null`을 전달합니다.  
다음 코드도 가능합니다.

```javascript
hhk_xhr_2.send();
```

### 18. 외부 API 호출 전체 예외 무시

```javascript
} catch (e) {}
```

다음과 같은 동기 예외를 무시합니다.

- 잘못된 URL
    
- XMLHttpRequest 생성 오류
    
- `sessionStorage` 접근 오류
    
- 브라우저 보안 정책 관련 오류  
    다만 비동기 요청 이후 발생하는 모든 오류를 이 외부 `try-catch`가 잡는 것은 아닙니다. Callback 내부 오류는 Callback 안쪽의 `try-catch`가 처리합니다.
    

### 19. `hhk_id`가 없는 경우

```javascript
else {
```

`sessionStorage`에 유효한 `hhk_id`가 없으면 신규 Google 유입인지 검사합니다.

### 20. Referrer 전체 URL 디코딩

```javascript
var hhk_referrer_full =
    decodeURIComponent(document.referrer);
```

이전 페이지 URL 전체를 디코딩합니다.  
예:

```text
https://www.google.com/search?q=health%20care
```

변환:

```text
https://www.google.com/search?q=health care
```

#### 문제점

URL 전체에 잘못된 `%` 인코딩이 있으면 `decodeURIComponent()`에서 예외가 발생할 수 있습니다.  
그런데 이 코드는 해당 라인을 `try-catch` 밖에서 실행하므로 예외가 발생하면 전체 스크립트가 중단됩니다.  
또한 호스트명만 확인하는 데 URL 전체를 디코딩할 필요는 없습니다.

### 21. 프로토콜 제거

```javascript
var hhk_referrer_host =
    hhk_referrer_full.replace('https://', '');
```

문자열에서 최초로 발견되는 `https://`를 제거합니다.  
예:

```text
https://www.google.com/search?q=health
→ www.google.com/search?q=health
```

다음 코드로 HTTP 프로토콜도 제거합니다.

```javascript
hhk_referrer_host =
    hhk_referrer_host.replace('http://', '');
```

### 22. 호스트명 부분 추출

```javascript
hhk_referrer_host =
    hhk_referrer_host.split('/')[0];
```

`/`를 기준으로 문자열을 나누고 첫 번째 값을 사용합니다.  
예:

```text
www.google.com/search?q=health
```

결과:

```text
www.google.com
```

다만 URL을 문자열로 직접 처리하는 것보다 `URL` 객체를 사용하는 것이 안전합니다.

```javascript
const referrerHost =
    new URL(document.referrer).hostname;
```

### 23. Google 도메인 여부 확인

```javascript
if (hhk_referrer_host.indexOf('google') != -1) {
```

Referrer 호스트명 문자열에 `google`이 포함되어 있으면 Google 유입으로 판단합니다.  
예:

|Referrer Host|결과|
|---|--:|
|`www.google.com`|참|
|`www.google.co.kr`|참|
|`news.google.com`|참|
|`google.fake-site.com`|참|
|`mygoogleexample.com`|참|
|마지막 두 개도 Google로 잘못 판단하는 문제가 있습니다.||
|안전한 방식:||

```javascript
const host = new URL(document.referrer).hostname.toLowerCase();
const isGoogle =
    host === 'google.com' ||
    host.endsWith('.google.com') ||
    /^([a-z0-9-]+\.)*google\.[a-z.]+$/.test(host);
```

### 24. 임의 식별값 생성 준비

```javascript
try {
    var characters =
        'ABCDEFGHIJKLMNOPQRSTUVWXYZ' +
        'abcdefghijklmnopqrstuvwxyz' +
        '0123456789';
```

랜덤 ID에 사용할 문자 집합입니다.

|문자 유형|개수|
|---|--:|
|영문 대문자|26|
|영문 소문자|26|
|숫자|10|
|합계|62|

### 25. 빈 ID 변수 생성

```javascript
var hhk_id = '';
```

생성될 12자리 ID를 저장할 변수입니다.

### 26. 12자리 반복

```javascript
for (
    var hhk_i = 0;
    hhk_i < 12;
    hhk_i++
) {
```

총 12회 반복하여 한 글자씩 추가합니다.

### 27. 임의 문자 위치 선택

```javascript
var hhk_random_index =
    Math.floor(
        Math.random() * characters.length
    );
```

`Math.random()`은 0 이상 1 미만의 난수를 반환합니다.

```javascript
Math.random() * 62
```

결과는 0 이상 62 미만이며, `Math.floor()`로 정수화하여 0～61 사이의 인덱스를 생성합니다.

### 28. 랜덤 문자 추가

```javascript
hhk_id += characters[hhk_random_index];
```

선택된 문자를 기존 ID 뒤에 추가합니다.  
12회 반복 결과 예:

```text
G7kaZ30Pq1Mn
```

#### 보안상 주의

`Math.random()`은 보안용 난수 생성기가 아닙니다.  
단순 방문 세션 구분에는 사용할 수 있지만 다음 용도로는 적합하지 않습니다.

- 인증 Token
    
- 개인정보 조회 Key
    
- 권한 검증 Key
    
- 비밀번호 재설정 Token  
    보안이 필요한 식별값은 다음을 사용하는 것이 좋습니다.
    

```javascript
crypto.randomUUID();
```

또는:

```javascript
const bytes = new Uint8Array(16);
crypto.getRandomValues(bytes);
```

### 29. 신규 식별값 등록 URL 생성

```javascript
var hhk_url =
    'https://bdg-api.vercel.app/b/i?i=' +
    hhk_id;
```

생성한 ID를 외부 API에 전달합니다.  
예:

```text
https://bdg-api.vercel.app/b/i?i=G7kaZ30Pq1Mn
```

코드 흐름상 `/b/i`는 식별값을 서버에 등록하거나 검색어 수집 프로세스를 시작하는 API로 추정됩니다.

### 30. 신규 등록 요청 객체 생성

```javascript
var hhk_xhr = new XMLHttpRequest();
```

두 번째 XMLHttpRequest 객체를 생성합니다.  
기존 ID 조회에는 `hhk_xhr_2`, 신규 ID 등록에는 `hhk_xhr`를 사용합니다.

### 31. 신규 등록 GET 요청 초기화

```javascript
hhk_xhr.open('GET', hhk_url, true);
```

비동기 GET 방식으로 `/b/i` API를 호출하도록 설정합니다.

### 32. 인증정보 제외

```javascript
hhk_xhr.withCredentials = false;
```

외부 API 호출 시 Cookie와 인증정보를 포함하지 않습니다.  
하지만 CORS 허용 응답 Header는 여전히 필요합니다.

### 33. 신규 등록 요청 상태 처리

```javascript
hhk_xhr.onreadystatechange = function () {
    if (hhk_xhr.readyState === 4) {
        if (
            hhk_xhr.status >= 200 &&
            hhk_xhr.status < 300
        ) {}
    }
};
```

요청이 완료되고 HTTP 상태가 성공인지 검사하지만 성공 블록이 비어 있습니다.  
따라서 성공 여부와 관계없이 실질적으로 하는 작업이 없습니다.

```javascript
if (성공) {
    // 아무 작업 없음
}
```

이 Callback은 현재 상태에서는 제거해도 실행 결과가 같습니다.

### 34. 신규 등록 요청 전송

```javascript
hhk_xhr.send(null);
```

`/b/i` API에 GET 요청을 보냅니다.

### 35. ID를 SessionStorage에 저장

```javascript
sessionStorage.setItem('hhk_id', hhk_id);
```

생성한 ID를 즉시 저장합니다.  
중요한 점은 API 호출이 비동기라는 것입니다.

```javascript
hhk_xhr.send(null);
sessionStorage.setItem('hhk_id', hhk_id);
```

API 응답이 오기 전에 `hhk_id`가 먼저 저장됩니다.  
따라서 `/b/i` 요청이 실패해도 브라우저에는 `hhk_id`가 남습니다.  
이후 코드 실행 시:

1. `hhk_id`가 있으므로 조회 분기로 진입
    
2. `/b/g?i=...` 호출
    
3. 서버에는 등록되지 않은 ID일 수 있으므로 조회 실패  
    이런 불일치가 생길 수 있습니다.
    

### 36. 전체 실행 시나리오

#### 시나리오 1: 최초 Google 유입

```text
sessionStorage:
hhk_id 없음
hhk_q 없음
```

실행:

1. `document.referrer` 확인
    
2. 호스트명에 `google` 포함 여부 확인
    
3. 12자리 `hhk_id` 생성
    
4. `/b/i?i={hhk_id}` 비동기 호출
    
5. 응답을 기다리지 않고 `hhk_id` 저장  
    결과:
    

```text
hhk_id = G7kaZ30Pq1Mn
hhk_q = 없음
```

#### 시나리오 2: 같은 탭에서 다음 페이지 이동

```text
hhk_id 있음
hhk_q 없음
```

실행:

1. `/b/g?i={hhk_id}` 호출
    
2. 응답의 `response.q` 확인
    
3. 존재하면 `hhk_q` 저장  
    결과:
    

```text
hhk_id = G7kaZ30Pq1Mn
hhk_q = health
```

#### 시나리오 3: `hhk_id`, `hhk_q` 모두 존재

아무 작업도 하지 않습니다.

### 37. 코드가 의도하는 처리 시점

이 코드에서는 최초 페이지 진입 시 바로 `q`를 조회하지 않습니다.

```text
최초 페이지:
ID 생성 및 /b/i 호출
다음 코드 실행 시점:
기존 ID로 /b/g 호출
```

따라서 `hhk_q` 취득은 일반적으로 다음 상황에서 발생합니다.

- 사용자가 같은 탭에서 다른 페이지로 이동
    
- 현재 페이지를 새로고침
    
- 코드가 다시 실행됨  
    첫 화면에서 바로 검색어가 필요한 경우 현재 구조로는 시점상 늦을 수 있습니다.
    

### 38. 핵심 문제점

|우선순위|문제점|영향|
|--:|---|---|
|1|외부 API가 실제 Google 검색어를 어떻게 얻는지 불명확|코드만으로 기능의 진위 검증 불가|
|2|CORS 허용 Header가 없으면 요청 실패|`buykorea.org`에서 응답을 읽지 못함|
|3|외부 API 성공 전에 `hhk_id` 저장|서버 등록 실패와 브라우저 상태 불일치|
|4|오류를 모두 빈 `catch`로 무시|운영 장애 원인 파악 불가|
|5|Google 판별이 문자열 포함 검사|가짜 도메인도 Google로 오인|
|6|`decodeURIComponent(document.referrer)`가 `try-catch` 밖에 존재|잘못된 인코딩이면 전체 스크립트 중단|
|7|GET 요청으로 등록 작업 수행|캐시·로그·재호출·웹 크롤러 영향 가능|
|8|ID를 URL Query String으로 전달|브라우저·Proxy·서버 Access Log에 기록|
|9|`Math.random()`으로 ID 생성|충돌 가능성과 예측 가능성 존재|
|10|HTTP 오류·Timeout 처리 없음|요청 실패 시 복구되지 않음|
|11|`hhk_q` 조회 재시도 제어 없음|페이지 이동 때마다 반복 조회 가능|
|12|신규 등록 Callback이 비어 있음|상태 검사 코드가 실질적으로 무의미|
|13|Referrer가 빈 값이면 아무 처리 없음|브라우저 정책에 따라 Google 유입도 감지 불가|
|14|개인 식별값과 검색어를 외부 서버에 전달 가능성|개인정보·보안·위탁 처리 검토 필요|

### 39. CORS 관점 분석

현재 요청은 Cross-Origin 요청입니다.

```text
A 사이트:
https://buykorea.org
외부 API:
https://bdg-api.vercel.app
```

브라우저는 외부 API 응답에 다음 Header가 있어야 JavaScript에 응답을 전달합니다.

```http
Access-Control-Allow-Origin: https://buykorea.org
```

응답 Header가 없다면 다음과 같은 오류가 발생합니다.

```text
No 'Access-Control-Allow-Origin' header is present
```

이 경우:

- 요청이 서버에 도착했을 수는 있음
    
- 브라우저 JavaScript는 응답을 읽을 수 없음
    
- `hhk_res` 처리 불가
    
- `hhk_q` 저장 불가  
    `withCredentials = false`는 CORS 자체를 해제하지 않습니다.
    

### 40. Google 검색어 취득 가능성 관점

이 JavaScript만 보면 Google 검색어를 직접 추출하는 로직은 없습니다.

```javascript
document.referrer
```

에서는 호스트명에 `google`이 포함되었는지만 확인합니다.  
다음 값은 읽지 않습니다.

```text
q
oq
p
검색창 입력값
```

따라서 검색어 `q`는 외부 API가 반환하는 값에 전적으로 의존합니다.

```javascript
hhk_res.response.q
```

그러나 A 사이트에서 생성한 임의 ID는 Google 검색 페이지에서 사용된 적이 없습니다.

```text
Google 검색 시점:
hhk_id 존재하지 않음
A 사이트 도착 후:
hhk_id 생성
```

따라서 일반적인 웹 구조에서는 이 ID만으로 Google 검색 서버의 사용자 검색어를 조회할 수 없습니다.  
외부 API가 검색어를 반환한다면 다음 중 하나일 가능성을 별도로 검증해야 합니다.

- Referrer에 우연히 포함된 Query를 서버가 분석
    
- Landing URL 파라미터를 분석
    
- 별도의 브라우저 확장·Proxy·수집 Script 사용
    
- 사용자의 실제 검색어가 아닌 추정값 반환
    
- API가 사전에 저장한 다른 데이터와 임의 매칭
    

### 41. 로직을 읽기 쉽게 변환한 코드

아래 코드는 원본 기능을 그대로 풀어서 표현한 형태입니다.

```javascript
const existingId = sessionStorage.getItem('hhk_id');
const existingKeyword = sessionStorage.getItem('hhk_q');
const hasValidId =
    existingId &&
    existingId !== 'null';
const hasValidKeyword =
    existingKeyword &&
    existingKeyword !== 'null';
if (hasValidId) {
    if (!hasValidKeyword) {
        try {
            const requestUrl =
                'https://bdg-api.vercel.app/b/g?i=' +
                encodeURIComponent(existingId);
            const xhr = new XMLHttpRequest();
            xhr.open('GET', requestUrl, true);
            xhr.withCredentials = false;
            xhr.onreadystatechange = function () {
                if (xhr.readyState !== 4) {
                    return;
                }
                if (xhr.status < 200 || xhr.status >= 300) {
                    return;
                }
                try {
                    const response = JSON.parse(xhr.responseText);
                    if (
                        response &&
                        response.success === true &&
                        response.response &&
                        response.response.q
                    ) {
                        sessionStorage.setItem(
                            'hhk_q',
                            response.response.q
                        );
                    }
                } catch (error) {
                    console.error(
                        '검색어 응답 분석 실패',
                        error
                    );
                }
            };
            xhr.send();
        } catch (error) {
            console.error(
                '검색어 조회 요청 실패',
                error
            );
        }
    }
} else {
    let referrerHost = '';
    try {
        referrerHost =
            document.referrer
                ? new URL(document.referrer).hostname.toLowerCase()
                : '';
    } catch (error) {
        console.error(
            'Referrer 분석 실패',
            error
        );
    }
    const isGoogle =
        /^([a-z0-9-]+\.)*google\.[a-z.]+$/i.test(referrerHost);
    if (isGoogle) {
        try {
            const id = createRandomId(12);
            const requestUrl =
                'https://bdg-api.vercel.app/b/i?i=' +
                encodeURIComponent(id);
            const xhr = new XMLHttpRequest();
            xhr.open('GET', requestUrl, true);
            xhr.withCredentials = false;
            xhr.onreadystatechange = function () {
                if (xhr.readyState !== 4) {
                    return;
                }
                if (xhr.status >= 200 && xhr.status < 300) {
                    sessionStorage.setItem('hhk_id', id);
                } else {
                    console.error(
                        '식별값 등록 실패',
                        xhr.status
                    );
                }
            };
            xhr.send();
        } catch (error) {
            console.error(
                '식별값 등록 요청 실패',
                error
            );
        }
    }
}
function createRandomId(length) {
    const characters =
        'ABCDEFGHIJKLMNOPQRSTUVWXYZ' +
        'abcdefghijklmnopqrstuvwxyz' +
        '0123456789';
    const randomValues = new Uint32Array(length);
    crypto.getRandomValues(randomValues);
    let result = '';
    for (let index = 0; index < length; index++) {
        result +=
            characters[
                randomValues[index] % characters.length
            ];
    }
    return result;
}
```

### 42. 최종 평가

| 항목                                                                                                                                                                                                              | 분석 결과                                       |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| 주요 목적                                                                                                                                                                                                           | Google 유입 사용자에게 임의 ID를 발급하고 외부 API에서 `q` 조회 |
| Google 유입 판별                                                                                                                                                                                                    | `document.referrer` 호스트명에 `google` 포함 여부    |
| 사용자 식별                                                                                                                                                                                                          | 12자리 임의 문자열                                 |
| 저장 위치                                                                                                                                                                                                           | 브라우저 탭 단위 `sessionStorage`                  |
| 외부 통신                                                                                                                                                                                                           | `XMLHttpRequest` 비동기 GET                    |
| 검색어 직접 추출                                                                                                                                                                                                       | 하지 않음                                       |
| 검색어 출처                                                                                                                                                                                                          | 외부 API의 `response.q`                        |
| 첫 페이지에서 검색어 취득                                                                                                                                                                                                  | 일반적으로 불가                                    |
| 중요 기술 위험                                                                                                                                                                                                        | CORS, 비동기 등록 실패, 오류 은폐, 도메인 오판              |
| 중요 보안 위험                                                                                                                                                                                                        | 외부 서버로 사용자 식별값 전송, URL 로그 노출, 개인정보 처리 검토    |
| 이 코드는 **Google 검색어를 브라우저에서 직접 추출하는 코드가 아니라, Google 유입 여부만 판별한 후 외부 서비스가 반환하는 `q` 값을 신뢰하여 저장하는 코드**입니다. 따라서 핵심 검증 대상은 JavaScript 자체보다 `bdg-api.vercel.app`의 `/b/i`, `/b/g` API가 어떤 데이터를 어떤 근거로 수집·연결하여 반환하는지입니다. |                                             |

