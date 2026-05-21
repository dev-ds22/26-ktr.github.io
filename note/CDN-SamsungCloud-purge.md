## 결론

Samsung Cloud CDN 사용 시 **전체 Purge는 운영 장애를 유발할 수 있는 고위험 작업**입니다. 특히 JS/CSS/Image 같은 정적 파일 전체를 purge하면 CDN Edge 캐시가 비워지고, 이후 사용자 요청이 Origin으로 몰려 **TTFB 증가, Origin 부하 급증, 일시적 화면 지연**이 발생할 수 있습니다. Samsung SDS도 Purge를 “Edge 서버에 캐시된 콘텐츠를 삭제하는 기능”으로 설명하며, purge 후 사용자 요청 시 Origin에서 갱신 콘텐츠를 가져와 다시 캐시한다고 설명합니다. ([Samsung SDS](https://www.samsungsds.com/kr/techreport/networking-cdn.html "전보다 더 빠르고 효율적인 콘텐츠 전달을 위한 솔루션, CDN  | 클라우드 기술 백서 | 삼성SDS"))  
또한 `?ver=1.234` 같은 QueryString 방식의 버전 관리는 **CDN Cache Key에 QueryString이 포함되는지 여부**에 따라 효과가 완전히 달라집니다. Samsung SDS 기술 백서도 Cache Key는 보통 도메인, Path, QueryString으로 정해지며, QueryString 포함 여부에 따라 같은 Path라도 서로 다른 캐시 객체로 처리될 수 있다고 설명합니다. ([Samsung SDS](https://www.samsungsds.com/kr/techreport/networking-cdn.html "전보다 더 빠르고 효율적인 콘텐츠 전달을 위한 솔루션, CDN  | 클라우드 기술 백서 | 삼성SDS"))

## 1. Samsung Cloud CDN에서 전체 Purge의 위험성

### 1.1 전체 Purge의 동작 구조

```text
전체 Purge 실행
→ CDN Edge 서버의 캐시 콘텐츠 제거 또는 만료 처리
→ 이후 사용자 요청이 CDN Edge에 도착
→ Edge에 캐시 없음
→ Origin 서버로 요청 전달
→ Origin 응답 후 다시 CDN Edge에 캐시
```

Samsung SDS Global CDN은 전 세계 Edge 서버를 통해 정적 콘텐츠를 빠르게 제공하고 Origin 부하를 분산하는 구조입니다. 즉, 전체 purge는 이 장점을 일시적으로 제거하는 작업입니다. ([Samsung SDS](https://www.samsungsds.com/en/network-global-cdn/global-cdn.html "Global CDN | Cloud Service | Samsung SDS"))

### 1.2 운영 위험

|구분|위험 내용|영향|
|---|---|---|
|Origin 부하|CDN MISS 증가로 Origin 직접 요청 증가|Nginx/WAS/Object Storage 부하 증가|
|TTFB 증가|Edge가 Origin 응답을 기다림|JS/CSS/Image 로딩 지연|
|장애 오인|사용자 화면이 갑자기 느려짐|배포 장애처럼 보일 수 있음|
|비용 증가|Origin egress, CDN refill 트래픽 증가|트래픽 비용 증가 가능|
|캐시 적중률 저하|HIT Ratio 급락|CDN 효율 저하|
|브라우저 캐시 미해결|CDN purge로 브라우저 캐시는 삭제 불가|사용자는 여전히 구버전 파일을 볼 수 있음|
|동시 요청 폭증|인기 리소스가 동시에 MISS|thundering herd 현상 가능|
|Samsung SDS 기술 백서도 CDN purge로는 브라우저 캐시를 제거할 수 없고, 사용자가 새 콘텐츠를 받게 하려면 파일명 변경 같은 조치가 필요하다고 설명합니다. ([Samsung SDS](https://www.samsungsds.com/kr/techreport/networking-cdn.html "전보다 더 빠르고 효율적인 콘텐츠 전달을 위한 솔루션, CDN  \| 클라우드 기술 백서 \| 삼성SDS"))|||

## 2. QueryString 사용 여부 확인 방법

### 2.1 CDN 설정에서 확인할 것

CDN 콘솔 또는 운영사에 아래 항목을 확인해야 합니다.

```text
1. Cache Key에 QueryString을 포함하는가?
2. QueryString 전체를 포함하는가, 특정 파라미터만 포함하는가?
3. QueryString 순서까지 Cache Key에 반영하는가?
4. Origin 요청 시 QueryString을 전달하는가?
5. Purge 시 QueryString 포함 URL을 별도 객체로 purge해야 하는가?
6. /file.js purge가 /file.js?ver=1.234까지 같이 purge하는가?
7. ?ver 값이 달라지면 Edge에서 다른 객체로 캐시되는가?
```

핵심 질문은 아래 1개입니다.

```text
/js/app.js 와 /js/app.js?ver=1.234 는 CDN에서 같은 캐시 객체인가, 다른 캐시 객체인가?
```

Samsung SDS 문서 기준으로 일반적으로 QueryString이 Cache Key에 포함되면 같은 Path라도 QS가 다를 때 다른 요청으로 판단하고 캐시합니다. 다만 실제 서비스 설정은 CDN 정책에 따라 달라지므로 콘솔/운영사 확인이 필요합니다. ([Samsung SDS](https://www.samsungsds.com/kr/techreport/networking-cdn.html "전보다 더 빠르고 효율적인 콘텐츠 전달을 위한 솔루션, CDN  | 클라우드 기술 백서 | 삼성SDS"))

### 2.2 브라우저/헤더로 확인할 것

같은 파일에 대해 아래 URL을 비교합니다.

```text
https://cdn.example.com/js/app.js
https://cdn.example.com/js/app.js?ver=1.234
https://cdn.example.com/js/app.js?ver=1.235
```

확인 헤더:

```http
Cache-Control
Age
ETag
Last-Modified
Content-Length
Content-Encoding
Vary
X-Cache
X-Cache-Key
Via
Server-Timing
```

판단 기준:

|현상|의미|
|---|---|
|`?ver` 값마다 `X-Cache-Key`가 다름|QueryString이 Cache Key에 포함됨|
|`?ver` 변경 시 첫 요청만 느리고 두 번째부터 빠름|새 Cache Key로 MISS 후 HIT 가능성|
|QueryString 제거 URL만 빠름|QueryString별 캐시 객체 상태 차이 가능성|
|`Age`가 URL마다 다름|URL별 캐시 객체가 다를 가능성|
|`X-Cache: MISS` 반복|캐시 미적용 또는 TTL 만료 재검증 가능성|

## 3. `?ver=1.234` 방식의 장점

|장점|설명|
|---|---|
|적용 쉬움|JSP/HTML에서 URL 뒤에 파라미터만 붙이면 됨|
|기존 파일명 유지|`/js/app.js` 파일명을 바꾸지 않아도 됨|
|브라우저 캐시 회피|QueryString이 바뀌면 브라우저가 새 URL로 인식 가능|
|배포 영향 작음|파일명 기반 import 구조를 크게 바꾸지 않아도 됨|
|긴급 갱신 가능|`?ver` 값을 바꿔 새 캐시 객체를 만들 수 있음|
|이 방식은 기존 JSP/Tiles/jQuery 기반 프로젝트에서 빠르게 적용하기 쉽습니다.||

## 4. `?ver=1.234` 방식의 단점

|단점|설명|
|---|---|
|CDN 설정 의존|CDN이 QueryString을 Cache Key에 포함하지 않으면 효과 약함|
|캐시 객체 분산|`ver` 값이 자주 바뀌면 캐시 객체가 계속 새로 생김|
|Origin 부하 증가|새 `ver`마다 최초 요청은 Origin Fetch 가능|
|Purge 복잡성|Path purge와 QueryString purge 범위가 달라질 수 있음|
|오래된 버전 방치|1년 이상 된 `ver` URL이 남아 특정 캐시 객체만 비정상 상태일 수 있음|
|관리 난이도|파일 내용과 `ver` 값의 일치 여부 관리 필요|
|무분별한 변경|모든 리소스에 같은 `ver`를 붙이면 전체 캐시가 동시에 무효화됨|
|특히 `?ver=1.234`를 매 배포마다 전체 JS/CSS에 동일하게 변경하면, 실제로 바뀌지 않은 파일까지 새 URL이 되어 CDN MISS가 대량 발생합니다.||

## 5. 실무에서 주의해야 할 점

### 5.1 QueryString 값은 “파일 내용 변경”과 연결되어야 함

나쁜 예:

```text
/js/common.js?ver=20260515
/js/app.js?ver=20260515
/css/main.css?ver=20260515
```

모든 파일에 같은 전역 버전을 붙이면, 실제 변경되지 않은 파일도 전부 새 캐시 객체가 됩니다.  
권장:

```text
/js/common.js?ver=파일내용해시
/js/app.js?ver=파일내용해시
/css/main.css?ver=파일내용해시
```

### 5.2 `Cache-Control: public`만으로는 부족할 수 있음

정적 파일에는 명확한 TTL이 필요합니다.

```http
Cache-Control: public, max-age=31536000
```

파일 URL이 변경될 때만 내용이 바뀌는 구조라면:

```http
Cache-Control: public, max-age=31536000, immutable
```

Samsung SDS 문서도 `max-age`를 Edge 서버 내 콘텐츠의 최대 캐싱 기간으로 설명하고, TTL 만료 후 Origin과 유효성 검사를 수행할 수 있다고 설명합니다. ([Samsung SDS](https://www.samsungsds.com/kr/techreport/networking-cdn.html "전보다 더 빠르고 효율적인 콘텐츠 전달을 위한 솔루션, CDN  | 클라우드 기술 백서 | 삼성SDS"))

### 5.3 Origin의 `Last-Modified`/`ETag` 확인

TTL 만료 후 CDN이 Origin에 재검증할 때 `Last-Modified` 또는 `ETag`가 중요합니다. Samsung SDS도 캐시 만료 시 IMS 요청으로 Origin과 비교하며, 효율적 캐시 사용을 위해 Origin의 `Last-Modified` 설정을 권장한다고 설명합니다. ([Samsung SDS](https://www.samsungsds.com/kr/techreport/networking-cdn.html "전보다 더 빠르고 효율적인 콘텐츠 전달을 위한 솔루션, CDN  | 클라우드 기술 백서 | 삼성SDS"))  
확인:

```bash
curl -I "https://cdn.example.com/js/app.js?ver=1.234"
```

확인할 값:

```http
Last-Modified: ...
ETag: ...
Cache-Control: ...
Age: ...
```

### 5.4 Purge 대상은 최소화

운영 원칙:

```text
1. 전체 purge 금지에 가깝게 운영
2. URL 단위 purge 기본
3. 디렉토리 purge는 제한적으로 사용
4. 전체 purge는 승인 기반으로만 허용
5. purge 후 preload/warm-up 수행
```

### 5.5 브라우저 캐시는 CDN Purge로 해결되지 않음

CDN purge는 Edge 캐시 삭제이지, 사용자 브라우저 캐시 삭제가 아닙니다. Samsung SDS도 브라우저에 캐시된 콘텐츠는 CDN purge로 제거할 수 없으므로 파일명 변경 등으로 사용자가 다시 받게 해야 한다고 설명합니다. ([Samsung SDS](https://www.samsungsds.com/kr/techreport/networking-cdn.html "전보다 더 빠르고 효율적인 콘텐츠 전달을 위한 솔루션, CDN  | 클라우드 기술 백서 | 삼성SDS"))

## 6. 개선안

## 6.1 최선: 파일명 해시 방식

가장 권장되는 구조입니다.

```text
기존:
selectDigtlKbcGoodsListMng.js?ver=5.035555555
개선:
selectDigtlKbcGoodsListMng.8f3a91c2.js
```

장점:

```text
1. 파일 내용이 바뀌면 URL도 바뀜
2. 브라우저/CDN 장기 캐시 적용 가능
3. QueryString Cache Key 정책 의존 감소
4. 오래된 캐시 객체 문제 감소
5. purge 필요성 감소
```

권장 헤더:

```http
Cache-Control: public, max-age=31536000, immutable
```

## 6.2 차선: QueryString은 유지하되 content hash 사용

파일명을 바꾸기 어렵다면:

```text
/js/app.js?ver=8f3a91c2
```

주의:

```text
1. CDN Cache Key에 QueryString 포함 필수
2. 파일별 개별 hash 사용
3. 전역 ver 사용 지양
4. QueryString 포함 purge 정책 확인
5. 동일 파일 내용이면 ver를 바꾸지 않음
```

## 6.3 기존 JSP/Spring Framework 5.3 환경 개선 방향

Spring MVC 정적 리소스를 사용한다면 `VersionResourceResolver` 또는 파일명 기반 버전 전략을 검토할 수 있습니다.

```text
/javascripts/app-8f3a91c2.js
/css/main-1a2b3c4d.css
```

Spring 5.3 기반 JSP라면 `ResourceUrlEncodingFilter`, `ResourceUrlProvider`를 이용해 렌더링 시 버전 URL을 출력하는 구조가 가능합니다. 단, 기존 정적 리소스 경로, CDN prefix, Tiles/JSP include 방식과 함께 검증해야 합니다.

## 7. 운영 정책 권장안

|영역|권장안|
|---|---|
|캐시 키|정적 파일은 QueryString 정책을 명확히 문서화|
|JS/CSS|파일명 해시 또는 파일별 hash query 사용|
|HTML|짧은 TTL 또는 revalidate|
|Image|파일명 변경이 가능하면 long TTL|
|Purge|URL 단위 purge 기본|
|전체 Purge|승인·이중확인·작업시간 제한|
|배포|변경 파일만 purge 또는 새 URL 배포|
|모니터링|HIT ratio, MISS ratio, TTFB, Origin request count 확인|
|장애 대응|purge 전후 Origin 부하와 CDN 상태 비교|

## 8. 확인용 커맨드

### 8.1 QueryString별 캐시 상태 비교

```bash
curl -I "https://cdn.example.com/js/app.js"
curl -I "https://cdn.example.com/js/app.js?ver=1.234"
curl -I "https://cdn.example.com/js/app.js?ver=1.235"
```

비교:

```text
Age
Cache-Control
ETag
Last-Modified
X-Cache
X-Cache-Key
Content-Length
```

### 8.2 강제 재검증 테스트

```bash
curl -I -H "Cache-Control: no-cache" "https://cdn.example.com/js/app.js?ver=1.234"
```

주의: 이 요청은 테스트 목적이며, 실제 성능 판단 시 일반 요청 결과와 구분해야 합니다.

### 8.3 브라우저 DevTools 확인

```text
Network
→ Disable cache 체크 해제
→ JS 파일 선택
→ Headers / Timing 확인
→ X-Cache, Age, Waiting for server response 확인
```

## 9. Samsung Cloud CDN 운영사에 확인할 질문

```text
1. 현재 CDN 서비스의 Cache Key에 QueryString이 포함되어 있는가?
2. QueryString 전체를 포함하는가, 특정 파라미터만 포함하는가?
3. `ver` 파라미터는 Cache Key에 포함되는가?
4. QueryString 순서가 Cache Key에 영향을 주는가?
5. `/js/app.js` purge 시 `/js/app.js?ver=1.234`도 같이 purge되는가?
6. `/js/app.js?ver=1.234`만 지정 purge 가능한가?
7. 전체 purge 수행 시 Edge 전체 반영까지 예상 시간은?
8. purge 후 preload 또는 cache warming 기능이 있는가?
9. CDN 로그에서 URL별 HIT/MISS/REFRESH_MISS 확인이 가능한가?
10. 특정 URL의 TTFB 지연 시 Edge POP, Origin Fetch Time을 분리해 볼 수 있는가?
11. Origin의 Cache-Control과 CDN TTL 중 무엇을 우선하는가?
12. `Last-Modified`/`ETag` 기반 재검증 정책은 어떻게 동작하는가?
```

## 10. 최종 권장안

현재처럼 `?ver=1.234` 방식으로 운영 중이라면 바로 폐기하기보다 아래 순서로 개선하는 것이 안전합니다.

```text
1. Samsung Cloud CDN의 QueryString Cache Key 포함 여부 확인
2. 현재 느린 URL과 빠른 URL의 X-Cache/Age/TTFB 비교
3. `ver`가 파일별 내용 hash인지, 전역 버전인지 확인
4. 전역 버전이면 파일별 hash 방식으로 전환
5. 장기적으로 queryString 방식보다 파일명 해시 방식으로 전환
6. 전체 purge는 차단하거나 승인 기반으로 제한
7. 배포 후 변경 URL만 purge 또는 preload
```

핵심은 **전체 purge로 문제를 해결하려 하지 말고, 파일 URL 자체가 변경되도록 설계해 purge 필요성을 줄이는 것**입니다. CDN의 목적은 Origin 부하를 줄이는 것인데, 전체 purge는 그 보호막을 한 번에 걷어내는 작업이므로 운영에서는 최후 수단으로만 사용해야 합니다.


# CDN 사용에도 느려지는 이유

## 결론

`?ver=1.234`로 오래전에 배포한 URL도 **영구적으로 CDN에 cache 된다고 보면 안 됩니다.** CDN 캐시는 보통 `TTL`, `Cache-Control`, `CDN 캐시 정책`, `Purge 이력`, `Edge 서버별 보유 상태`, `요청 빈도`, `QueryString Cache Key 정책`에 따라 유지·만료·재검증·삭제됩니다.  
즉, CDN을 사용해도 특정 JS/CSS 파일 로딩이 느려질 수 있습니다. 특히 `?ver=1.234`처럼 오래된 QueryString URL이 느리다면 **브라우저 문제가 아니라 CDN Edge가 즉시 응답하지 못하고 Origin 조회 또는 재검증을 수행하는 상태**일 가능성이 큽니다.

## 1. 오래전에 배포한 `?ver=1.234`는 계속 캐시되는가?

### 답

조건부입니다.

```text
캐시될 수도 있고, 만료될 수도 있고, 특정 Edge에는 없을 수도 있고, 매번 Origin 재검증을 할 수도 있습니다.
```

Samsung SDS Global CDN 설명에는 **캐시 정책, 캐시 만료 시간, TTL 도달 후 Edge 캐시 purge** 기능이 포함되어 있습니다. 즉, CDN은 한 번 캐시했다고 해서 무제한 보관하는 구조가 아닙니다. ([Samsung SDS](https://www.samsungsds.com/en/network-global-cdn/global-cdn.html?utm_source=chatgpt.com "Global CDN | Cloud Service"))

## 2. CDN 캐시가 오래 유지되지 않는 대표 이유

| 구분             | 설명                                                       |
| -------------- | -------------------------------------------------------- |
| TTL 만료         | CDN Edge의 캐시 유효 시간이 지나면 stale 상태가 됨                      |
| `max-age` 없음   | `Cache-Control: public`만 있고 `max-age`가 없으면 장기 캐시 보장 어려움  |
| Purge 이력       | 전체 purge 또는 경로 purge로 캐시가 삭제됐을 수 있음                      |
| Edge별 보유 차이    | 서울 Edge에는 있지만 다른 Edge에는 없을 수 있음                          |
| 요청 빈도 낮음       | 자주 요청되지 않는 객체는 CDN 내부 정책상 제거될 수 있음                       |
| QueryString 분리 | `?ver=1.234`가 별도 객체라 특정 객체만 stale/MISS 상태일 수 있음          |
| Origin 재검증     | TTL 만료 후 `If-Modified-Since`, `If-None-Match`로 Origin 확인 |
| Cache Bypass   | 쿠키, 헤더, CDN Rule 때문에 캐시를 우회할 수 있음                        |

## 3. `?ver=1.234`와 CDN Cache Key 관계

Samsung SDS 기술 문서에서는 **Cache-key에 QueryString 포함 여부에 따라 캐시 동작이 달라지며, QueryString을 인식하면 동일 QS는 같은 객체, 다른 QS는 다른 객체로 인식**한다고 설명합니다. ([Samsung SDS](https://www.samsungsds.com/kr/techreport/networking-cdn.html?utm_source=chatgpt.com "전보다 더 빠르고 효율적인 콘텐츠 전달을 위한 솔루션, CDN"))  
예를 들어 CDN이 QueryString을 Cache Key에 포함하면 아래 3개는 서로 다른 캐시 객체가 됩니다.

```text
/js/app.js
/js/app.js?ver=1.234
/js/app.js?ver=1.235
```

Akamai 문서도 Query String Parameter를 Cache Key에 포함하거나 제외하는 정책을 설정할 수 있다고 설명합니다. ([Akamai TechDocs](https://techdocs.akamai.com/property-mgr/docs/cache-key-query-param?utm_source=chatgpt.com "Cache Key Query Parameters"))  
따라서 `?ver=1.234`만 느리다면 아래 가능성이 높습니다.

```text
/js/app.js?ver=1.234 객체만 TTL 만료, MISS, 재검증, 또는 비정상 캐시 상태
/js/app.js 또는 ?ver=1.235는 다른 Cache Key라 정상 HIT
```

## 4. CDN을 사용해도 로딩 속도가 느려지는 구조

### 4.1 정상적인 CDN HIT

```text
브라우저
→ CDN Edge
→ 캐시 HIT
→ 즉시 응답
→ TTFB 낮음
```

이 경우 JS/CSS는 빠르게 내려옵니다.

### 4.2 CDN MISS 또는 만료 재검증

```text
브라우저
→ CDN Edge
→ 캐시 없음 또는 TTL 만료
→ Origin 서버로 요청
→ Origin 응답 대기
→ CDN이 응답 시작
→ TTFB 증가
```

이때 DevTools에서는 `Waiting for server response`, 즉 TTFB가 길게 보입니다. MDN 기준으로 `Cache-Control: max-age`는 응답이 fresh 상태로 재사용 가능한 시간을 지정하고, `s-maxage`는 공유 캐시에서 `max-age`를 재정의합니다. ([MDN 웹 문서](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control?utm_source=chatgpt.com "Cache-Control header - HTTP - MDN Web Docs"))

## 5. `?ver=1.234`가 오래됐을 때 오히려 느려지는 이유

### 5.1 TTL이 이미 만료됨

`?ver=1.234`가 1년 이상 전 배포 값이라면, CDN에 남아 있더라도 fresh 상태가 아닐 수 있습니다.

```text
캐시 객체 존재
→ TTL 만료
→ 요청 때마다 Origin에 재검증
→ Origin 응답 느림
→ TTFB 증가
```

이 경우 응답 헤더에서 아래처럼 보일 수 있습니다.

```http
Age: 0
X-Cache: TCP_REFRESH_MISS
X-Cache: TCP_MISS
Cache-Control: public
```

특히 `Cache-Control: public`만 있고 `max-age` 또는 CDN TTL override가 약하면 “공유 캐시에 저장 가능”하다는 의미에 가깝고, “1년 동안 빠르게 유지”된다는 의미는 아닙니다.

## 5.2 Edge에 캐시 객체가 없어짐

CDN Edge는 무한 저장소가 아닙니다. 특정 파일이 오래 요청되지 않으면 내부 정책에 따라 제거될 수 있습니다.

```text
오래된 ver URL
→ 최근 요청 적음
→ 일부 Edge에서 객체 제거
→ 다음 요청 때 MISS
→ Origin Fetch
→ 느림
```

즉, 오래된 URL이라고 해서 항상 더 빠른 것이 아니라, **요청 빈도가 낮아진 오래된 객체일수록 Edge에 없을 가능성**도 있습니다.

## 5.3 `?ver=1.234` 객체만 Cache Key 상태가 다름

같은 파일이라도 아래는 서로 다른 객체일 수 있습니다.

```text
/js/app.js?ver=1.234
/js/app.js?ver=1.235
```

그래서 `?ver=1.234`만 느리고 `?ver=1.235`는 빠를 수 있습니다.

```text
?ver=1.234 → stale/MISS/revalidation 반복
?ver=1.235 → 방금 캐시됨/HIT
```

## 5.4 Purge 후 특정 객체만 다시 느려짐

전체 purge나 경로 purge가 과거에 수행되었으면 `?ver=1.234` 객체가 삭제됐을 수 있습니다.

```text
전체 purge
→ 모든 Edge 캐시 삭제
→ 다음 요청은 Origin으로 감
→ Origin이 느리면 TTFB 증가
```

Samsung SDS 문서도 purge는 Edge 서버에 캐시된 콘텐츠를 삭제하고, 이후 사용자가 요청하면 Origin에서 변경된 콘텐츠를 가져와 Edge에 다시 캐시하는 구조로 설명합니다. ([Samsung SDS](https://www.samsungsds.com/kr/techreport/networking-cdn.html?utm_source=chatgpt.com "전보다 더 빠르고 효율적인 콘텐츠 전달을 위한 솔루션, CDN"))

## 5.5 요청 헤더가 재검증을 강제함

브라우저 개발자도구에서 `Disable cache`가 켜져 있거나, 새로고침 방식에 따라 아래 헤더가 붙을 수 있습니다.

```http
Cache-Control: no-cache
Pragma: no-cache
```

이 경우 CDN이 캐시를 갖고 있어도 재검증을 수행할 수 있습니다.

```text
일반 새로고침보다 강력 새로고침/DevTools Disable cache 상태에서 느려질 수 있음
```

## 5.6 Origin이 느리면 CDN도 느려짐

CDN MISS 또는 재검증이 발생하면 CDN은 Origin의 응답을 기다립니다.

```text
CDN이 느린 것이 아니라,
CDN이 Origin 응답을 기다리기 때문에
브라우저 입장에서는 CDN 응답도 느려 보임
```

Origin 지연 원인:

```text
정적 파일이 Nginx 직접 서빙이 아니라 WAS를 탐
파일 시스템 I/O 지연
압축 gzip/br을 요청마다 동적으로 수행
Origin 서버 CPU/디스크 부하
Nginx rewrite/fallback 처리
L4/방화벽/보안장비 지연
Origin과 CDN Edge 간 네트워크 지연
```

## 6. 확인해야 할 응답 헤더

느린 URL과 빠른 URL을 비교해야 합니다.

```bash
curl -I "https://cdn.example.com/js/app.js?ver=1.234"
curl -I "https://cdn.example.com/js/app.js?ver=1.235"
curl -I "https://cdn.example.com/js/app.js"
```

확인 항목:

```http
Cache-Control
Age
Expires
ETag
Last-Modified
Vary
Content-Length
Content-Encoding
X-Cache
X-Cache-Key
Via
Server-Timing
```

판단 기준:

| 헤더                     | 느린 URL에서 의심할 상태                             |
| ---------------------- | ------------------------------------------- |
| `X-Cache`              | `MISS`, `REFRESH_MISS`, `BYPASS`, `EXPIRED` |
| `Age`                  | 없음, `0`, 매번 초기화                             |
| `Cache-Control`        | `public`만 있고 `max-age` 없음                   |
| `ETag`/`Last-Modified` | 매번 재검증 발생                                   |
| `Vary`                 | 과도한 분기, `Accept-Encoding` 외 불필요한 값          |
| `Content-Encoding`     | 느린 URL만 압축 미적용 또는 동적 압축                     |
| `Content-Length`       | 느린 URL만 다른 파일 응답 가능                         |

## 7. `Age`로 보는 캐시 상태

| `Age` 상태        | 의미                             |
| --------------- | ------------------------------ |
| `Age`가 큼        | CDN 또는 공유 캐시에 오래 보관된 응답 가능성    |
| `Age: 0`        | 방금 Origin에서 가져왔거나 재검증된 응답 가능성  |
| `Age` 없음        | 캐시 미적중, bypass, CDN 헤더 비노출 가능성 |
| 반복 요청마다 증가      | 캐시 HIT 가능성 높음                  |
| 반복 요청마다 0으로 초기화 | 매번 재검증 또는 MISS 가능성             |

## 8. 왜 `?ver` 값을 바꾸면 빨라지는가?

`?ver=1.235`로 바꾸면 CDN 입장에서는 새 URL일 수 있습니다.

```text
기존 느린 객체:
app.js?ver=1.234

새 객체:
app.js?ver=1.235
```

가능한 시나리오:

```text
1. 첫 요청은 MISS지만 Origin이 그 순간 빠르게 응답
2. 이후 Edge에 새로 캐시되어 HIT
3. 기존 ?ver=1.234 객체의 stale/revalidation 문제를 우회
4. 브라우저도 새 URL로 인식해 기존 캐시 문제를 회피
```

하지만 이것은 근본 해결이 아닐 수 있습니다. 시간이 지나 `?ver=1.235`도 TTL이 만료되거나 요청 빈도가 줄면 같은 문제가 반복될 수 있습니다.

## 9. 실무에서 가장 많이 생기는 원인 조합

현재 설명된 증상 기준으로는 아래 조합이 가장 유력합니다.

```text
1. ?ver=1.234가 CDN Cache Key에 포함되어 별도 객체가 됨
2. 해당 객체는 오래되어 TTL 만료 또는 Edge에서 제거됨
3. 요청 시 CDN이 Origin 재검증 또는 Origin Fetch 수행
4. Origin 응답이 느려 TTFB가 커짐
5. QueryString 제거 또는 ver 변경 URL은 다른 Cache Key라 빠르게 응답
```

## 10. 대책

### 10.1 단기 대책

```text
1. 느린 URL과 빠른 URL의 X-Cache, Age, Cache-Control 비교
2. CDN 운영사에 ?ver가 Cache Key에 포함되는지 확인
3. ?ver=1.234 URL만 Purge 후 재측정
4. Purge 후 첫 요청/두 번째 요청 시간 비교
5. Origin에서 해당 파일 직접 호출 시 TTFB 확인
6. CDN에 해당 URL Preload 또는 Cache Warming 요청
```

### 10.2 CDN 설정 대책

```text
1. JS/CSS 경로에 명확한 TTL 설정
2. Cache-Control: public, max-age=31536000 또는 s-maxage 적용
3. QueryString Cache Key 정책 문서화
4. 불필요한 QueryString은 Cache Key에서 제외
5. ver 파라미터를 사용할 경우 ver만 Cache Key에 포함
6. Purge 시 QueryString 포함 URL 처리 방식 확인
```

단, `?ver`별로 실제 콘텐츠가 달라질 수 있다면 QueryString을 무조건 제외하면 안 됩니다. Akamai도 콘텐츠 차이가 있는 query parameter를 무시하면 잘못된 캐시 객체가 제공될 수 있다고 주의합니다. ([Akamai TechDocs](https://techdocs.akamai.com/terraform/docs/cache-key-query-params?utm_source=chatgpt.com "cache_​key_​query_​params"))

## 10.3 Origin 대책

```text
1. 정적 파일은 WAS가 아니라 Nginx/Object Storage에서 직접 서빙
2. gzip/br 사전 압축 사용
3. `Last-Modified`, `ETag`, `Cache-Control` 정상 설정
4. Nginx access log에서 request_time 확인
5. CDN MISS 시 Origin 응답 시간이 긴지 분리 측정
```

## 10.4 구조 개선

가장 권장되는 방식은 QueryString 버전이 아니라 **파일명 해시 방식**입니다.

```text
기존:
selectDigtlKbcGoodsListMng.js?ver=1.234

권장:
selectDigtlKbcGoodsListMng.8f3a91c2.js
```

권장 헤더:

```http
Cache-Control: public, max-age=31536000, immutable
```

이 구조는 파일 내용이 바뀌면 URL 자체가 바뀌므로, 브라우저/CDN 장기 캐시를 안전하게 적용할 수 있습니다.

## 11. CDN 업체 또는 인프라팀에 확인할 질문

```text
1. 현재 Samsung Cloud CDN의 Cache Key에 QueryString이 포함되어 있는가?
2. `ver` 파라미터가 Cache Key에 포함되는가?
3. `/js/app.js`와 `/js/app.js?ver=1.234`는 같은 캐시 객체인가?
4. `?ver=1.234` URL의 최근 HIT/MISS/REFRESH/BYPASS 비율은?
5. 해당 URL의 Edge별 Age 값과 Cache 상태는?
6. TTL 만료 후 stale 객체를 바로 제공하는가, Origin 재검증을 기다리는가?
7. 전체 purge 또는 path purge 이력이 있는가?
8. QueryString 포함 URL purge 방식은 어떻게 동작하는가?
9. 특정 Edge POP에서만 느린가?
10. Edge → Origin fetch time을 확인할 수 있는가?
11. Origin 응답 헤더의 Cache-Control을 따르는가, CDN TTL override가 있는가?
12. Preload 또는 Cache Warming 기능을 사용할 수 있는가?
```

## 최종 요약

`?ver=1.234`로 오래전에 배포했다고 해서 CDN에 계속 빠르게 남아 있는 것은 아닙니다. CDN은 TTL이 만료되거나 purge되거나 Edge에서 제거되거나 재검증이 필요하면 Origin을 다시 조회합니다. 이때 Origin 응답이 느리면 CDN을 쓰고 있어도 브라우저에서는 `Waiting for server response`, 즉 TTFB가 길게 보입니다.  
현재처럼 특정 `?ver` 값만 느리다면 핵심은 **QueryString이 Cache Key에 포함되는지, 해당 Cache Key의 TTL/Age/HIT 상태가 어떤지, MISS 또는 재검증 시 Origin 응답이 느린지**를 확인하는 것입니다. 장기적으로는 `?ver=...` 방식보다 **content hash 파일명 + long TTL + immutable + URL 단위 purge 최소화** 구조가 더 안정적입니다.

## 전제

공개된 Samsung SDS/Samsung Cloud Platform 자료만으로는 **CDN 서비스별 실제 설정값, 특정 URL의 HIT/MISS 비율, Edge별 상태, Purge 이력**까지는 확정할 수 없습니다. 공개 자료로 확인 가능한 것은 **Cache Key에 QueryString을 포함할 수 있고, 일반적으로 QueryString 포함 여부가 캐시 객체 구분에 영향을 준다는 원리**와 **TTL/Purge/Monitoring 기능의 존재**입니다. 실제 운영 설정은 **SCP 콘솔 설정, CDN 운영 로그, Samsung Cloud 지원 문의**로 확인해야 합니다.

## 신뢰성 검증

| 구분  | 자료                            |   신뢰도 | 한계                                                                                 |
| --- | ----------------------------- | ----: | ---------------------------------------------------------------------------------- |
| 공식  | Samsung SDS CDN 기술 백서         |    높음 | 원리 설명 중심, 개별 서비스 설정값은 없음                                                           |
| 공식  | Samsung SDS Global CDN 서비스 소개 |    높음 | 기능 개요 중심, 세부 로그 항목은 없음                                                             |
| 공식  | SCP Technical Guide PDF       | 중간~높음 | WordPress 구축 예시 문서이나 SCP Global CDN이 Akamai 기반임을 확인 가능                             |
| 공식  | SCP Well-Architected PDF      | 중간~높음 | 운영/모니터링·서비스 제한 설명, URL별 CDN 로그 수준은 없음                                              |
| 보조  | Akamai 공식 문서                  |    중간 | SCP Global CDN이 Akamai 기반이라는 공식 자료와 연결해서 참고 가능, Samsung Cloud 콘솔 설정과 1:1 동일 보장은 아님 |


- Samsung SDS 기술 백서는 Cache Key가 보통 도메인, Path, QueryString Parameter로 정해지고, QueryString 포함 여부에 따라 캐시 동작이 달라진다고 설명합니다. 또한 QueryString이 Cache Key에 포함되면 동일 Path라도 QueryString에 따라 다른 요청으로 판단될 수 있다고 설명합니다. ([Samsung SDS](https://www.samsungsds.com/kr/techreport/networking-cdn.html "전보다 더 빠르고 효율적인 콘텐츠 전달을 위한 솔루션, CDN  | 클라우드 기술 백서 | 삼성SDS"))
- Samsung SDS의 Global CDN 소개 자료는 Origin/Content 설정에서 Cache Key를 구성할 수 있고, 캐싱 정책과 만료 시간을 설정할 수 있다고 설명합니다. ([Samsung SDS](https://www.samsungsds.com/en/network-global-cdn/global-cdn.html "Global CDN | Cloud Service | Samsung SDS"))
- SCP 기술 가이드에는 SCP Global CDN이 Akamai CDN 서비스를 사용한다고 되어 있어, Akamai Cache Key Query Parameter 문서는 동작 원리 검증용 보조 자료로 사용할 수 있습니다.

## 질문별 답변

|  No | 질문                                                        | 공개자료 기준 답변                                                                                                                                                                                                | 확정 여부       |
| --: | --------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
|   1 | 현재 Samsung Cloud CDN의 Cache Key에 QueryString이 포함되는가?      | **포함될 수 있음.** Samsung SDS 자료는 일반적으로 Cache Key가 도메인, Path, QueryString으로 정해진다고 설명합니다. 다만 현재 귀사 CDN 설정에서 QueryString을 포함하는지는 콘솔 설정 확인이 필요합니다.                                                               | 설정 확인 필요    |
|   2 | `ver` 파라미터가 Cache Key에 포함되는가?                             | QueryString 전체 포함 또는 `ver` 포함 정책이면 포함됩니다. QueryString 무시 또는 특정 파라미터 제외 정책이면 포함되지 않습니다.                                                                                                                    | 설정 확인 필요    |
|   3 | `/js/app.js`와 `/js/app.js?ver=1.234`는 같은 캐시 객체인가?         | QueryString이 Cache Key에 포함되면 **다른 객체**입니다. QueryString을 무시하도록 설정되어 있으면 **같은 객체**로 볼 수 있습니다.                                                                                                               | 설정 확인 필요    |
|   4 | `?ver=1.234` URL의 최근 HIT/MISS/REFRESH/BYPASS 비율은?         | 공개 자료로는 확인 불가합니다. CDN 로그 또는 Samsung Cloud 지원팀의 URL별 캐시 로그가 필요합니다.                                                                                                                                         | 공개자료 확인 불가  |
|   5 | 해당 URL의 Edge별 Age 값과 Cache 상태는?                           | 공개 자료로는 확인 불가합니다. 브라우저/`curl -I`로 현재 응답 헤더는 볼 수 있지만, Edge 전체 상태는 CDN 로그가 필요합니다.                                                                                                                           | 공개자료 확인 불가  |
|   6 | TTL 만료 후 stale 객체를 바로 제공하는가, Origin 재검증을 기다리는가?           | Samsung SDS 자료는 TTL 만료 또는 Origin 콘텐츠 변경 시 Edge의 기존 캐시가 제거되고, 새 콘텐츠를 Origin에서 가져와 캐시한다고 설명합니다. stale 제공 정책은 별도 설정 여부 확인이 필요합니다.                                                                            | 부분 확인       |
|   7 | 전체 purge 또는 path purge 이력이 있는가?                           | 공개 자료로는 확인 불가합니다. SCP Logging & Audit, 콘솔 작업 이력, CDN 운영 로그 확인이 필요합니다.                                                                                                                                     | 공개자료 확인 불가  |
|   8 | QueryString 포함 URL purge 방식은 어떻게 동작하는가?                   | Purge는 Edge에 캐시된 콘텐츠를 삭제하는 기능입니다. QueryString이 Cache Key에 포함되어 있다면 `app.js`와 `app.js?ver=1.234`가 별도 객체일 수 있으므로, purge 범위가 Path 기준인지 QueryString 포함 URL 기준인지 확인해야 합니다.                                     | 설정 확인 필요    |
|   9 | 특정 Edge POP에서만 느린가?                                       | 공개 자료로는 확인 불가합니다. SCP Global CDN은 Akamai Edge 기반이므로 지역/POP별 차이가 있을 수 있으나, 실제 특정 POP 지연 여부는 CDN 로그나 지원팀 확인이 필요합니다.                                                                                         | 공개자료 확인 불가  |
|  10 | Edge → Origin fetch time을 확인할 수 있는가?                      | 공개 자료에서 URL별 Edge→Origin fetch time 조회 가능 여부는 확인되지 않습니다. Samsung Cloud Monitoring은 Global CDN을 모니터링 대상에 포함하지만, URL별 fetch time 제공 여부는 별도 확인이 필요합니다.                                                       | 지원 문의 필요    |
|  11 | Origin 응답 헤더의 Cache-Control을 따르는가, CDN TTL override가 있는가? | Samsung 자료상 캐싱 정책과 만료 시간을 설정할 수 있습니다. `Cache-Control: max-age`, `no-cache`, `no-store`, `Last-Modified`, `ETag`는 캐시 동작에 영향을 주는 헤더로 설명됩니다. 현재 서비스가 Origin 헤더를 따르는지, CDN TTL로 override하는지는 콘솔 설정 확인이 필요합니다. | 설정 확인 필요    |
|  12 | Preload 또는 Cache Warming 기능을 사용할 수 있는가?                   | 공개 Samsung SDS 자료에서 Preload/Cache Warming 기능 제공 여부는 명확히 확인되지 않았습니다. Purge 후 첫 사용자 요청이 Origin에서 콘텐츠를 가져와 Edge에 다시 캐시하는 구조는 확인됩니다.                                                                          | 공식 지원 문의 필요 |

## 1~3번 상세 판단: QueryString과 Cache Key

Samsung SDS 자료상 Cache Key는 보통 도메인, Path, QueryString Parameter로 구성됩니다. 또한 QueryString을 Cache Key에 포함하면 같은 Path라도 QueryString 값이 다를 때 다른 요청으로 판단될 수 있습니다. ([Samsung SDS](https://www.samsungsds.com/kr/techreport/networking-cdn.html "전보다 더 빠르고 효율적인 콘텐츠 전달을 위한 솔루션, CDN  | 클라우드 기술 백서 | 삼성SDS"))  
따라서 아래는 설정에 따라 달라집니다.

```text
Case A. QueryString 포함
/js/app.js
/js/app.js?ver=1.234
/js/app.js?ver=1.235
→ 모두 다른 캐시 객체 가능
```

```text
Case B. QueryString 무시
/js/app.js
/js/app.js?ver=1.234
/js/app.js?ver=1.235
→ 같은 캐시 객체 가능
```

Akamai 공식 문서도 기본적으로 전체 QueryString을 포함해 Cache Key를 만들 수 있고, 특정 파라미터만 포함하거나 제외하거나 전체 QueryString을 무시하는 설정이 가능하다고 설명합니다. SCP Global CDN이 Akamai 기반이라는 Samsung SDS 문서와 함께 보면, Samsung Cloud CDN도 유사한 Cache Key 정책 확인이 핵심입니다.

## 4~5번 상세 판단: HIT/MISS, Edge별 Age는 공개자료로 알 수 없음

`?ver=1.234` URL의 최근 HIT/MISS/REFRESH/BYPASS 비율과 Edge별 Age 값은 **테넌트별 운영 데이터**입니다. 공개 문서에는 나오지 않습니다.  
다만 SCP Well-Architected 문서는 Samsung Cloud Monitoring이 인프라 리소스의 사용 상태, 변경 정보, 로그를 수집하고 이벤트를 생성한다고 설명하며, Cloud Monitoring 대상 목록에 Networking의 Global CDN이 포함됩니다. 따라서 최소한 Global CDN 단위의 모니터링 가능성은 공식 자료로 확인됩니다.  
확인 요청 항목:

```text
1. URL별 HIT/MISS/REFRESH/BYPASS 로그 제공 가능 여부
2. Edge POP별 캐시 상태 확인 가능 여부
3. Age 또는 캐시 생성 시각 확인 가능 여부
4. 특정 URL의 최근 24시간/7일 HIT Ratio
5. 특정 QueryString URL의 Cache Key 확인
```

브라우저 또는 서버에서 즉시 확인할 수 있는 값은 아래 정도입니다.

```bash
curl -I "https://CDN도메인/js/app.js"
curl -I "https://CDN도메인/js/app.js?ver=1.234"
curl -I "https://CDN도메인/js/app.js?ver=1.235"
```

비교 항목:

```text
Cache-Control
Age
ETag
Last-Modified
Content-Length
Content-Encoding
Vary
X-Cache
X-Cache-Key
Via
Server-Timing
```

단, `X-Cache`, `X-Cache-Key`, `Server-Timing`은 CDN 설정에 따라 응답에 노출되지 않을 수 있습니다.

## 6번 상세 판단: TTL 만료 후 동작

Samsung SDS Global CDN 소개는 TTL 만료 또는 Origin 콘텐츠 변경 시 기존 Edge 캐시가 제거되고, 새 콘텐츠를 Origin에서 가져와 사용자의 요청을 처리하면서 Edge에 캐시한다고 설명합니다. ([Samsung SDS](https://www.samsungsds.com/en/network-global-cdn/global-cdn.html "Global CDN | Cloud Service | Samsung SDS"))  
Samsung SDS 기술 백서도 `max-age`를 Edge 서버 내 콘텐츠의 최대 캐싱 기간으로 설명하고, `no-cache`는 캐시 파일 생성은 가능하지만 매번 Origin 유효성 검사를 요청한다고 설명합니다. `Last-Modified`는 CDN 캐시 서버의 IMS 요청 시 유효성 검사 수단으로 사용된다고 설명합니다. ([Samsung SDS](https://www.samsungsds.com/kr/techreport/networking-cdn.html "전보다 더 빠르고 효율적인 콘텐츠 전달을 위한 솔루션, CDN  | 클라우드 기술 백서 | 삼성SDS"))  
따라서 기본적인 해석은 아래입니다.

```text
TTL 유효
→ Edge 캐시 HIT 가능
TTL 만료
→ Origin 재검증 또는 Origin Fetch 가능
Origin 응답 지연
→ CDN을 사용해도 TTFB 증가 가능
```

다만 `stale-if-error`, `serve stale on origin error`, `stale-while-revalidate` 같은 구체 정책은 공개 Samsung SDS 자료에서 명확히 확인되지 않았습니다. 이 부분은 CDN 서비스 설정 또는 지원팀 확인이 필요합니다.

## 7번 상세 판단: Purge 이력

Purge 이력은 공개 자료로는 확인할 수 없습니다. Samsung Cloud Platform 문서는 Logging & Audit이 콘솔 사용자 활동 같은 클라우드/사용자 활동 기록을 남기는 용도로 설명하고, Cloud Monitoring도 로그/이벤트 처리를 제공한다고 설명합니다. 따라서 purge가 콘솔/API 작업으로 수행됐다면 작업 이력 또는 감사 로그 확인 대상입니다.  
확인 위치:

```text
1. SCP Console > Global CDN > 해당 CDN 서비스 > Purge 이력
2. SCP Logging & Audit > 사용자/API 작업 이력
3. Jenkins purge job 실행 이력
4. OpenAPI 호출 로그
5. Samsung Cloud 지원팀 CDN purge audit 요청
```

## 8번 상세 판단: QueryString 포함 URL Purge 방식

Samsung SDS 자료에서 Purge는 Edge 서버에 캐시된 콘텐츠를 삭제하는 기능입니다. Origin 콘텐츠가 갱신되어도 Edge 캐시가 만료되기 전까지는 Origin의 갱신 콘텐츠를 다시 가져오지 않기 때문에, 즉시 갱신이 필요하면 Purge로 기존 캐시를 제거한다고 설명합니다. ([Samsung SDS](https://www.samsungsds.com/kr/techreport/networking-cdn.html "전보다 더 빠르고 효율적인 콘텐츠 전달을 위한 솔루션, CDN  | 클라우드 기술 백서 | 삼성SDS"))  
하지만 `QueryString 포함 URL`을 Purge할 때 아래 중 어느 방식인지는 공개 자료로 확정할 수 없습니다.

```text
방식 A. Path 기준 purge
/js/app.js purge 시 /js/app.js?ver=1.234까지 제거
```

```text
방식 B. Exact URL 기준 purge
/js/app.js만 제거하고 /js/app.js?ver=1.234는 별도 제거 필요
```

```text
방식 C. QueryString 포함 exact purge
/js/app.js?ver=1.234만 제거 가능
```

반드시 Samsung Cloud 측에 아래 질문으로 확인해야 합니다.

```text
/js/app.js purge 시 /js/app.js?ver=1.234 객체도 함께 삭제되는가?
/js/app.js?ver=1.234를 exact purge할 수 있는가?
QueryString이 Cache Key에 포함된 경우 purge도 QueryString 단위로 수행되는가?
```

## 9~10번 상세 판단: Edge POP 지연과 Edge→Origin fetch time

SCP 기술 가이드는 SCP Global CDN이 Akamai CDN 서비스를 사용하며, 여러 국가/네트워크의 Akamai 리소스를 통해 Origin 콘텐츠를 전 세계에 배포할 수 있다고 설명합니다.  
이 구조에서는 특정 지역/POP/Edge에서만 Cache Miss, Origin 연결 지연, 특정 Edge의 stale 상태가 발생할 수 있습니다. 그러나 특정 Edge POP 지연 여부와 Edge→Origin fetch time은 공개 자료로는 확인할 수 없고 CDN 로그 또는 지원팀 확인이 필요합니다.  
확인 요청:

```text
1. 느린 요청의 Edge POP ID
2. Edge POP별 HIT/MISS 상태
3. Edge→Origin fetch time
4. Origin response time
5. 특정 POP에서만 REFRESH_MISS 또는 MISS 반복 여부
6. Origin Shield 또는 상위 캐시 계층 사용 여부
```

## 11번 상세 판단: Origin Cache-Control vs CDN TTL Override

Samsung SDS 자료는 `Cache-Control`, `Expires`, `Last-Modified`, `ETag` 같은 캐시 관련 HTTP 헤더를 설명합니다. 특히 `max-age`는 Edge 서버 내 콘텐츠의 최대 캐싱 기간으로 설명됩니다. ([Samsung SDS](https://www.samsungsds.com/kr/techreport/networking-cdn.html "전보다 더 빠르고 효율적인 콘텐츠 전달을 위한 솔루션, CDN  | 클라우드 기술 백서 | 삼성SDS"))  
또한 Global CDN 서비스 소개는 캐시 콘텐츠 정책과 캐시 만료 시간을 구성할 수 있다고 설명합니다. ([Samsung SDS](https://www.samsungsds.com/en/network-global-cdn/global-cdn.html "Global CDN | Cloud Service | Samsung SDS"))  
따라서 확인해야 할 설정은 아래입니다.

```text
1. Origin Cache-Control을 Honor하는가?
2. Origin Expires를 Honor하는가?
3. CDN Console의 Cache Expiration 설정이 Origin 헤더보다 우선하는가?
4. 특정 경로 /js/** 에 별도 TTL Rule이 있는가?
5. QueryString URL에도 동일 TTL Rule이 적용되는가?
```

SCP Well-Architected 자료에는 Global CDN의 Cache Expiration time 제한이 1시간~30일로 표시됩니다. 이 수치는 서비스 제한 자료 기준이므로 현재 사용하는 상품/계약/콘솔 설정과 일치하는지 확인이 필요합니다.

## 12번 상세 판단: Preload / Cache Warming

공개 Samsung SDS 자료에서는 **Preload 또는 Cache Warming 기능 제공 여부를 명확히 확인하지 못했습니다.** 확인 가능한 공식 설명은 Purge 후 사용자의 요청이 발생하면 Origin에서 새 콘텐츠를 가져오면서 Edge에 다시 캐시한다는 구조입니다. ([Samsung SDS](https://www.samsungsds.com/kr/techreport/networking-cdn.html "전보다 더 빠르고 효율적인 콘텐츠 전달을 위한 솔루션, CDN  | 클라우드 기술 백서 | 삼성SDS"))  
따라서 선택지는 2가지입니다.

```text
1. Samsung Cloud 지원팀에 공식 Preload 기능 지원 여부 확인
2. 공식 Preload가 없다면 배포 후 주요 URL을 지역별로 호출해 수동 warming 수행
```

단, 수동 warming은 Edge POP 전체를 보장하지 못합니다. Akamai 기반 CDN은 사용자 위치·DNS·라우팅에 따라 다른 Edge가 선택될 수 있기 때문에, 단일 Jenkins 서버에서 `curl`을 실행해도 일부 Edge만 warming될 수 있습니다. SCP Global CDN이 Akamai 기반이라는 점은 Samsung SDS 기술 가이드에서 확인됩니다.

## 최종 확인 요청서 초안

Samsung Cloud 운영사 또는 인프라 담당자에게 아래처럼 요청하는 것이 가장 정확합니다.

```text
대상 URL:
https://CDN도메인/js/app.js?ver=1.234
확인 요청:
1. 현재 CDN 서비스의 Cache Key 정책에서 QueryString 전체가 포함되는지 확인 부탁드립니다.
2. ver 파라미터가 Cache Key에 포함되는지 확인 부탁드립니다.
3. /js/app.js와 /js/app.js?ver=1.234가 동일 캐시 객체인지 확인 부탁드립니다.
4. 최근 7일 기준 해당 URL의 HIT/MISS/REFRESH/BYPASS 비율 확인 부탁드립니다.
5. Edge POP별 캐시 상태와 Age 확인 가능 여부 확인 부탁드립니다.
6. TTL 만료 후 stale 제공인지 Origin 재검증/Fetch 대기인지 확인 부탁드립니다.
7. 최근 전체 purge, path purge, URL purge 이력 확인 부탁드립니다.
8. QueryString 포함 URL의 purge가 exact URL 기준인지 path variant 전체 기준인지 확인 부탁드립니다.
9. 특정 Edge POP에서만 TTFB가 높은지 확인 부탁드립니다.
10. Edge → Origin fetch time 또는 Origin response time 확인 가능 여부 확인 부탁드립니다.
11. Origin Cache-Control과 CDN TTL override 중 어떤 정책이 우선되는지 확인 부탁드립니다.
12. Preload 또는 Cache Warming 기능 지원 여부 확인 부탁드립니다.
```

## 실무 판단

현재 공개 자료 기준으로 가장 신뢰도 높은 판단은 아래입니다.

```text
1. Samsung Cloud CDN은 Cache Key 설정을 제공한다.
2. QueryString은 Cache Key에 포함될 수 있고, 포함 여부에 따라 app.js와 app.js?ver=1.234는 같은 객체일 수도, 다른 객체일 수도 있다.
3. 특정 URL의 HIT/MISS/Edge 상태/Purge 이력은 공개자료로 확인할 수 없다.
4. TTL 만료 또는 Purge 후에는 Origin Fetch가 발생할 수 있고, 이때 Origin이 느리면 CDN을 써도 TTFB가 길어진다.
5. ver 파라미터 방식은 CDN Cache Key 정책 확인 없이는 안전한 버전 관리 방식이라고 단정할 수 없다.
```

따라서 우선순위는 **Cache Key의 QueryString 포함 여부 확인 → 해당 URL의 HIT/MISS/Age 확인 → Purge 이력 확인 → Origin Fetch 시간 확인 → 파일명 해시 기반 버전 관리 전환 검토**입니다.