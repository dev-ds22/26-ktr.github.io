---
layout: single
title: "RequestFilter"
excerpt: "RequestFilter"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-05-28"
last_modified_at: "2026-05-28 12:19:00 +0900"
---
## 1. RequestFilter

```java
import java.io.IOException;
import java.io.UnsupportedEncodingException;
import java.util.Enumeration;
import java.util.HashMap;
import java.util.Map;
import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.util.ContentCachingRequestWrapper;
import org.springframework.web.util.ContentCachingResponseWrapper;
import org.springframework.web.util.WebUtils;

public class RequestFilter implements Filter {

  private static final Logger LOGGER = LoggerFactory.getLogger(RequestFilter.class);

  @Override
  public void init(FilterConfig filterConfig) throws ServletException {
    // 필터 초기화 로직 (필요하면 사용)
  }

  @Override
  public void destroy() {
    // 필터 종료 시 처리할 로직 (필요시 구현)
  }

  public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
      throws IOException, ServletException {

    ContentCachingRequestWrapper requestWrapper =
        new ContentCachingRequestWrapper((HttpServletRequest) request);
    ContentCachingResponseWrapper responseWrapper =
        new ContentCachingResponseWrapper((HttpServletResponse) response);
    long start = System.currentTimeMillis();
    chain.doFilter(requestWrapper, responseWrapper);
    long end = System.currentTimeMillis();

    String contentType = responseWrapper.getContentType();
    String responseBody = " - ";
    String uri = requestWrapper.getRequestURI();

    // 상품 전시 카테고리 목록 (/comm/ajax/comm/getDispCategoryAllFront.do) 의 경우 응답값이 너무 길어 생략함
    if ((contentType == null || !contentType.contains("text/html"))
        && !uri.contains("/comm/ajax/comm/getDispCategoryAllFront.do")) {
      responseBody = this.getResponseBody(responseWrapper);
    }

    LOGGER.debug(
        "\n[REQUEST] {} - {} {} - {}\nHeaders : {}\nRequest : {}\nResponse : {}\n",
        new Object[] {
          ((HttpServletRequest) request).getMethod(),
          ((HttpServletRequest) request).getRequestURI(),
          responseWrapper.getStatus(),
          (double) (end - start) / 1000.0D,
          this.getHeaders((HttpServletRequest) request),
          this.getRequestBody(requestWrapper),
          responseBody
        });
    responseWrapper.copyBodyToResponse();
  }

  private Map getHeaders(HttpServletRequest request) {
    Map headerMap = new HashMap();
    Enumeration headerArray = request.getHeaderNames();

    if (headerArray != null) {
      while (headerArray.hasMoreElements()) {
        String headerName = (String) headerArray.nextElement();
        headerMap.put(headerName, request.getHeader(headerName));
      }
    }

    return headerMap;
  }

  private String getRequestBody(ContentCachingRequestWrapper request)
      throws UnsupportedEncodingException {
    ContentCachingRequestWrapper wrapper =
        (ContentCachingRequestWrapper)
            WebUtils.getNativeRequest(request, ContentCachingRequestWrapper.class);
    if (wrapper != null) {
      byte[] buf = wrapper.getContentAsByteArray();
      if (buf.length > 0) {
        try {
          String requestBody = new String(buf, 0, buf.length, "UTF-8");

          // 정규식을 사용하여 mberPwd 값 마스킹 처리
          requestBody = maskPasswordInRequestBody(requestBody);

          return requestBody;
        } catch (UnsupportedEncodingException var5) {
          return " - ";
        }
      }
    }

    return " - ";
  }

  private String getResponseBody(HttpServletResponse response) throws IOException {
    String payload = null;
    ContentCachingResponseWrapper wrapper =
        (ContentCachingResponseWrapper)
            WebUtils.getNativeResponse(response, ContentCachingResponseWrapper.class);
    if (wrapper != null) {
      byte[] buf = wrapper.getContentAsByteArray();
      if (buf.length > 0) {
        payload = new String(buf, 0, buf.length, "UTF-8");
      }
    }

    return null == payload ? " - " : payload;
  }

  private String maskPasswordInRequestBody(String requestBody) {
    // 정규식으로 mberPwd 값을 찾아서 마스킹 처리
    return requestBody.replaceAll("(\"mberPwd\"\\s*:\\s*\")([^\"]+)(\")", "$1**********$3");
  }
}

```

### 1-1. 핵심 기능 요약

제시된 `RequestFilter`는 Spring MVC 애플리케이션으로 인입되는 **HTTP 요청(Request)과 응답(Response)의 모든 내용(헤더, 바디, 수행 시간 등)을 가로채어 DEBUG 레벨의 로그로 기록**하고, 특정 보안 데이터(`mberPwd`)를 마스킹 처리하는 **디버깅 및 감사(Audit) 목적의 서블릿 필터**입니다.

### 1-2. 주요 기능 상세 분석

- **HTTP 메시지 바디 캐싱**: `ServletInputStream`은 단발성이므로 한 번 읽으면 다시 읽을 수 없습니다. 이를 해결하기 위해 Spring이 제공하는 `ContentCachingRequestWrapper` 및 `ContentCachingResponseWrapper`를 사용하여 바디 데이터를 메모리에 캐싱하고, 비즈니스 로직과 로깅 양쪽에서 바디를 재사용할 수 있도록 지원합니다.

- **성능 모니터링**: `chain.doFilter()` 전후의 `System.currentTimeMillis()` 차이를 계산하여 해당 요청의 총 처리 시간을 초(Seconds) 단위로 기록합니다.

- **특정 조건 기반 로깅 제어**: 응답의 `ContentType`이 HTML이 아니거나, 텍스트 데이터가 매우 길 것으로 예상되는 특정 전시 API (`/comm/ajax/comm/getDispCategoryAllFront.do`)인 경우 응답 바디 로깅을 제외하여 로그 폭증을 예방합니다.

- **민감 정보 마스킹**: `maskPasswordInRequestBody` 메서드 내에서 정규표현식을 활용하여 JSON 형태의 비밀번호 파라미터값(`"mberPwd":"..."`)을 `**********`로 치환하여 로그에 노출되지 않도록 방지합니다.

### 1-3. 코드 구조적 개선점 (Refactoring Points)

제시된 코드는 정상 동작하지만, 대규모 트래픽이 발생하는 실무 환경에서는 성능 저하 및 런타임 에러를 유발할 수 있는 요소들이 존재합니다.

|**분류**|**기존 코드의 문제점**|**개선 제안 및 대안 코드**|
|---|---|---|
|**성능 저하 위험**|`LOGGER.debug()` 문 내부에서 `getHeaders()`, `getRequestBody()`를 직접 호출하여, **로그 레벨이 INFO나 WARN일 때도 매번 바디 파싱 및 정규식 연산이 무조건 실행됨**|로깅 조건문(`if (LOGGER.isDebugEnabled())`)으로 감싸거나 가벼운 람다식 인터페이스 구조로 변경해야 함|
|**타입 안전성 결여**|`Map headerMap = new HashMap();` 등 Generic 타입이 생략된 Raw Type 구조 사용|`Map<String, String> headerMap = new HashMap<>();` 형태로 명확한 타입 정의 필요|
|**하드코딩 및 확장성**|특정 URI와 콘텐츠 타입이 필터 내부에 하드코딩되어 있어 대상이 추가될 때마다 소스 수정 필요|무시할 URL 패턴 리스트를 `FilterConfig`나 Spring `@Value` 프로퍼티로 주입받도록 개선|
|**문자열 상수 미활용**|`"UTF-8"`을 문자열로 하드코딩하여 예외 처리(`UnsupportedEncodingException`) 강제됨|`StandardCharsets.UTF_8.name()` 또는 `StandardCharsets.UTF_8` 객체를 사용하여 예외 처리 최소화|

#### 1-3-1. 🛠️ 개선된 `doFilter` 핵심 로직 예시

Java

```
@Override
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
    HttpServletRequest httpRequest = (HttpServletRequest) request;
    HttpServletResponse httpResponse = (HttpServletResponse) response;

    // 대용량 및 바이너리 요청(파일 업로드 등)은 캐싱에서 제외하는 방어 로직 필요
    if (isExcludeUri(httpRequest.getRequestURI()) || httpRequest.getContentType() != null && httpRequest.getContentType().contains("multipart")) {
        chain.doFilter(request, response);
        return;
    }

    ContentCachingRequestWrapper requestWrapper = new ContentCachingRequestWrapper(httpRequest);
    ContentCachingResponseWrapper responseWrapper = new ContentCachingResponseWrapper(httpResponse);
    long start = System.currentTimeMillis();
    
    try {
        chain.doFilter(requestWrapper, responseWrapper);
    } finally {
        long end = System.currentTimeMillis();
        // ★ 핵심 개선: DEBUG 레벨이 활성화된 경우에만 무거운 문자열 조작 로직 실행
        if (LOGGER.isDebugEnabled()) {
            logRequestAndResponse(requestWrapper, responseWrapper, end - start);
        }
        // 캐싱된 응답 바디를 실제 클라이언트 스트림으로 복사 (필수)
        responseWrapper.copyBodyToResponse();
    }
}
```

### 1-4. 실무 적용 시 치명적인 주의점 (Architecture & Production Hazards)

> [!CAUTION]
> 
> **1. 대용량 파일 업로드/다운로드 시 OutOfMemoryError (OOME) 발생**
> 
> `ContentCachingWrapper` 계열은 바디 데이터를 JVM Heap 메모리(`ByteArrayOutputStream`)에 전부 적재합니다. e-commerce 시스템에서 사용자가 몇 백 메가바이트(MB) 이상의 첨부 파일이나 상품 대량 등록 엑셀 파일을 업로드/다운로드할 경우, 동시 요청 수에 비례하여 힙 메모리가 고갈되어 서버가 다운(OOME)될 수 있습니다. 멀티파트 요청 타입은 필터 진입 단계에서 반드시 우회시켜야 합니다.

> [!WARNING]
> 
> **2. 바이너리 데이터 파일 깨짐 현상 (이미지, PDF 등)**
> 
> 응답 타입이 JSON/텍스트가 아닌 이미지 파일 스트림(`image/png`)이나 엑셀 다운로드일 때, 이를 무조건 `new String(buf, "UTF-8")`로 변환하려고 시도하면 불필요한 메모리 낭비는 물론 이진 데이터 오염이 발생할 수 있으므로 `ContentType` 체크를 엄격하게 수행해야 합니다.

> [!NOTE]
> 
> **3. 마스킹 로직의 한계 (보안 취약점)**
> 
> 현재 정규식은 JSON 형태(`"mberPwd" : "..."`)만 커버하고 있습니다. 만약 클라이언트가 `application/x-www-form-urlencoded` 형식(Form Submit)으로 `mberPwd=password123&id=user`와 같이 요청을 보낼 경우 정규식이 매칭되지 않아 **비밀번호가 평문으로 로그 파일에 그대로 기록되는 보안 사고**가 발생할 수 있습니다. 폼 데이터 형식에 대한 마스킹 규칙도 보완되어야 합니다.

> [!IMPORTANT]
> 
> **4. 비동기 요청(Async Request) 처리 시 데이터 유실 위험**
> 
> Spring MVC에서 `DeferredResult`나 `Callable` 등을 이용해 비동기 응답을 처리하는 아키텍처 구조라면, 일반 `Filter` 인터페이스는 요청이 끝날 때 서블릿 컨테이너에 의해 응답이 복사되어 버리거나 서블릿 재진입 시 중복 로깅이 발생할 수 있습니다. 따라서 실무에서는 일반 Filter 대신 Spring이 제공하는 **`OncePerRequestFilter`**를 상속받아 구현하는 것이 아키텍처 표준에 부합합니다.

## 2. 참고 - GPT
#### 2-1-1. 결론

해당 `RequestFilter`는 **Spring Servlet 요청/응답을 감싸서 HTTP Header, Request Body, Response Body, Status, 처리 시간을 debug 로그로 남기는 필터**입니다. 기능 자체는 장애 분석에 유용하지만, 커머스 운영 환경에서는 **개인정보·비밀번호·토큰·쿠키·주문/결제 응답값 노출, 대용량 응답 메모리 사용, 파일 다운로드/바이너리 응답 로깅, `copyBodyToResponse()` 누락 시 응답 유실 위험** 때문에 그대로 운영 적용하기에는 위험합니다.  
Spring의 `ContentCachingRequestWrapper`는 요청 body를 캐싱해 byte array로 다시 조회할 수 있게 하고, `ContentCachingResponseWrapper`는 응답 output stream/writer에 기록된 내용을 캐싱해 다시 조회할 수 있게 하는 wrapper입니다. 단, 응답 wrapper는 캐시된 본문을 실제 응답으로 다시 복사해야 하므로 `copyBodyToResponse()` 호출이 중요합니다. ([Javadoc](https://www.javadoc.io/doc/org.springframework/spring-web/5.3.13/org/springframework/web/util/ContentCachingRequestWrapper.html?utm_source=chatgpt.com "ContentCachingRequestWrapper (spring-web 5.3.13 API) - javadoc.io"))

#### 2-1-2. 필터의 전체 기능

```text
HTTP Request 진입
→ ContentCachingRequestWrapper 생성
→ ContentCachingResponseWrapper 생성
→ 다음 Filter/DispatcherServlet/Controller 실행
→ 처리 시간 계산
→ Header / Request Body / Response Body 조회
→ LOGGER.debug 출력
→ responseWrapper.copyBodyToResponse()
→ 클라이언트 응답 전송
```

|항목|현재 코드 기능|
|---|---|
|요청 감싸기|`ContentCachingRequestWrapper`로 request body 재조회 가능하게 처리|
|응답 감싸기|`ContentCachingResponseWrapper`로 response body 로깅 가능하게 처리|
|처리 시간|`chain.doFilter()` 전후 시간 차이 계산|
|Header 로그|전체 요청 header를 `Map`으로 수집|
|Request Body 로그|request body를 UTF-8 문자열로 변환 후 `mberPwd` 마스킹|
|Response Body 로그|HTML이 아니고 특정 URI가 아니면 응답 본문 로그 출력|
|응답 복원|`copyBodyToResponse()`로 캐시된 응답을 실제 response에 복사|

#### 2-1-3. 핵심 동작 상세

##### 2-1-3-1. Request Body 로깅

```java
ContentCachingRequestWrapper requestWrapper = new ContentCachingRequestWrapper((HttpServletRequest)request);
```

이 wrapper는 request body를 여러 번 읽기 위한 목적이 아니라, **하위 로직에서 읽힌 body를 캐시에 보관한 뒤 나중에 조회**하는 방식입니다. 즉, Controller나 MessageConverter가 request body를 읽지 않는 GET 요청, 일반 조회 요청에서는 `getContentAsByteArray()`가 비어 있을 수 있습니다. Spring Javadoc도 request input stream/reader에서 읽힌 content를 캐싱한다고 설명합니다. ([Javadoc](https://www.javadoc.io/doc/org.springframework/spring-web/5.3.13/org/springframework/web/util/ContentCachingRequestWrapper.html?utm_source=chatgpt.com "ContentCachingRequestWrapper (spring-web 5.3.13 API) - javadoc.io"))

현재 코드는 `chain.doFilter()` 이후에 request body를 조회하므로 위치 자체는 맞습니다.

```java
chain.doFilter(requestWrapper, responseWrapper);
...
this.getRequestBody(requestWrapper)
```

##### 2-1-3-2. Response Body 로깅

```java
ContentCachingResponseWrapper responseWrapper = new ContentCachingResponseWrapper((HttpServletResponse)response);
```

응답 wrapper는 response output stream/writer에 기록되는 내용을 중간에 캐싱합니다. 그래서 필터에서 response body를 읽을 수 있지만, 마지막에 아래 호출을 반드시 해야 클라이언트로 응답이 정상 전달됩니다.

```java
responseWrapper.copyBodyToResponse();
```

Spring Javadoc에서도 `copyBodyToResponse()`는 캐시된 body 전체를 response로 복사하는 메서드라고 설명합니다. 또한 `flushBuffer()`만으로는 underlying response에 content가 복사되지 않으므로 `copyBodyToResponse()`를 호출해야 한다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/util/ContentCachingResponseWrapper.html "ContentCachingResponseWrapper (Spring Framework 7.0.7 API)"))

#### 2-1-4. 현재 코드의 가장 큰 실무 위험

|위험|현재 코드|문제|
|---|---|---|
|응답 유실 가능성|`copyBodyToResponse()`가 `finally`에 없음|`chain.doFilter()` 이후 로직에서 예외 발생 시 응답 복사가 안 될 수 있음|
|개인정보 노출|`mberPwd`만 마스킹|`password`, `pwd`, `Authorization`, `Cookie`, `JSESSIONID`, 주문/결제정보 노출 가능|
|대용량 응답 로깅|HTML 외 대부분 body 로깅|JSON 대량 목록, Excel, PDF, 이미지, 첨부파일 응답에서 메모리/로그 폭증 가능|
|성능 저하|debug 여부와 무관하게 body 추출 수행|`LOGGER.debug`가 꺼져 있어도 body 변환 비용 발생 가능|
|인코딩 문제|UTF-8 고정|EUC-KR, MS949, 응답 charset이 다른 경우 깨짐|
|URI 예외 하드코딩|특정 URI만 제외|비슷한 대용량 API가 늘어나면 관리 어려움|
|Header 수집|`getHeader()`만 사용|동일 header multi-value 누락 가능|
|Raw 타입|`Map`, `HashMap` raw 사용|타입 안정성 낮음|
|HTTP 전제 캐스팅|바로 `HttpServletRequest` 캐스팅|비HTTP 요청이면 `ClassCastException` 가능|
|로그 인젝션|body/header 원문 로그|개행/제어문자 포함 시 로그 변조 가능|

#### 2-1-5. 특히 위험한 부분

##### 2-1-5-1. `copyBodyToResponse()`가 `finally`에 없음

현재 구조:

```java
chain.doFilter(requestWrapper, responseWrapper);
...
LOGGER.debug(...);
responseWrapper.copyBodyToResponse();
```

`getResponseBody()`, `getRequestBody()`, `LOGGER.debug()` 처리 중 예외가 발생하면 `copyBodyToResponse()`가 실행되지 않을 수 있습니다. 응답 wrapper는 body를 캐싱하고 있으므로, 복사가 안 되면 클라이언트 응답이 비정상 또는 빈 응답처럼 보일 수 있습니다.  
개선 방향:

```java
try {
    chain.doFilter(requestWrapper, responseWrapper);
} finally {
    responseWrapper.copyBodyToResponse();
}
```

다만 로그를 남기려면 `try-finally` 안에서 **로그 처리 후 반드시 복사**되도록 구성해야 합니다.

##### 2-1-5-2. `LOGGER.debug`가 꺼져 있어도 비용 발생

현재 코드는 debug 로그 출력 전에 이미 response body, request body, header를 모두 만듭니다.

```java
responseBody = this.getResponseBody(responseWrapper);
this.getHeaders(...)
this.getRequestBody(...)
```

운영에서 `DEBUG`가 꺼져 있어도 문자열 생성, byte array → String 변환, Map 생성 비용이 발생할 수 있습니다.  
개선 방향:

```java
if (LOGGER.isDebugEnabled()) {
    // body 추출 및 로그 생성
}
```

##### 2-1-5-3. 응답 body 로깅 조건이 위험함

현재 조건:

```java
if ((contentType == null || !contentType.contains("text/html"))
    && !uri.contains("/comm/ajax/comm/getDispCategoryAllFront.do")) {
    responseBody = this.getResponseBody(responseWrapper);
}
```

이 조건은 **HTML만 제외하고 나머지는 대부분 로깅**합니다.

|Content-Type|현재 동작|위험|
|---|---|---|
|`application/json`|로깅|개인정보/주문/결제 응답 노출|
|`application/xml`|로깅|외부 연계 데이터 노출|
|`text/plain`|로깅|토큰/오류 상세 노출 가능|
|`application/pdf`|로깅 시도|바이너리 깨짐/로그 오염|
|`application/vnd.ms-excel`|로깅 시도|대용량/바이너리 로그 위험|
|`image/png`|로깅 시도|의미 없는 바이너리 로그|
|`null`|로깅 시도|예외/불명확 응답까지 로깅|

운영에서는 **허용 목록 방식**이 안전합니다.

```text
application/json 중에서도 특정 URI, 특정 크기 이하만 로깅
```

##### 2-1-5-4. 마스킹이 `mberPwd` JSON 하나만 처리

현재 마스킹:

```java
return requestBody.replaceAll("(\"mberPwd\"\\s*:\\s*\")([^\"]+)(\")", "$1**********$3");
```

이 방식은 아래 값들을 마스킹하지 못합니다.

|값|현재 마스킹 여부|
|---|---|
|`"mberPwd":"1234"`|가능|
|`"password":"1234"`|불가|
|`"pwd":"1234"`|불가|
|`"newPassword":"1234"`|불가|
|`mberPwd=1234` form data|불가|
|`Authorization: Bearer ...`|불가|
|`Cookie: JSESSIONID=...`|불가|
|응답 body의 개인정보|불가|
|대소문자 다른 필드|불가|

커머스 프로젝트에서는 최소한 아래 항목은 마스킹해야 합니다.

```text
password, pwd, mberPwd, newPassword, confirmPassword
Authorization, Cookie, Set-Cookie, JSESSIONID
accessToken, refreshToken
email, phone, mobile, tel, cardNo, accountNo
주문자명, 수취인명, 주소, 결제 승인번호
```

#### 2-1-6. 운영 적용 여부 판단

|환경|적용 판단|
|---|---|
|로컬 개발|사용 가능|
|개발/QA|제한적 사용 가능|
|운영 DEBUG OFF|body 추출 자체를 막아야 함|
|운영 장애 임시 분석|특정 URI/사용자/시간대 제한 후 사용|
|상시 운영|현재 형태로는 비권장|

#### 2-1-7. 개선 방향

##### 2-1-7-1. `OncePerRequestFilter` 사용 권장

Spring 환경에서는 일반 `javax.servlet.Filter`보다 `OncePerRequestFilter`가 더 안전합니다. 동일 요청에서 필터가 중복 실행되는 상황을 줄이기 쉽고, Spring Bean으로 관리하기도 좋습니다.

##### 2-1-7-2. `try-finally`로 응답 복사 보장

`ContentCachingResponseWrapper`를 쓴다면 `copyBodyToResponse()`는 사실상 필수 후처리입니다. 예외 상황에서도 최대한 보장해야 합니다.

##### 2-1-7-3. DEBUG 활성화 시에만 body 추출

```java
if (LOGGER.isDebugEnabled()) {
    // header/body 생성
}
```

##### 2-1-7-4. 최대 로그 크기 제한

예:

```text
request body 최대 4KB
response body 최대 8KB
초과 시 "[TRUNCATED]" 표시
```

대용량 JSON, 상품 목록, 카테고리 전체 목록, 파일 응답을 모두 로그로 남기면 로그 시스템과 WAS 메모리에 부담이 큽니다.

##### 2-1-7-5. Content-Type 허용 목록 방식

권장:

```text
application/json
application/xml
text/plain
```

비권장:

```text
image/*
application/pdf
application/octet-stream
application/vnd.ms-excel
multipart/form-data
```

##### 2-1-7-6. Header 마스킹

현재 `getHeaders()`는 모든 header를 그대로 남깁니다. 운영에서는 아래 header는 마스킹해야 합니다.

```text
Authorization
Cookie
Set-Cookie
X-Auth-Token
Proxy-Authorization
```

##### 2-1-7-7. URI 하드코딩 제거

현재:

```java
!uri.contains("/comm/ajax/comm/getDispCategoryAllFront.do")
```

개선:

```text
log.exclude-uris=/comm/ajax/comm/getDispCategoryAllFront.do,/file/download.do,/image/*
```

XML property, properties 파일, enum, external config로 분리하는 편이 좋습니다.

#### 2-1-8. 개선 예시 코드

Spring 5.3 / `javax.servlet` 기준 예시입니다.

```java
import java.io.IOException;
import java.io.UnsupportedEncodingException;
import java.nio.charset.Charset;
import java.nio.charset.StandardCharsets;
import java.util.Collections;
import java.util.Enumeration;
import java.util.LinkedHashMap;
import java.util.Map;
import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.filter.OncePerRequestFilter;
import org.springframework.web.util.ContentCachingRequestWrapper;
import org.springframework.web.util.ContentCachingResponseWrapper;
public class RequestLoggingFilter extends OncePerRequestFilter {
    private static final Logger LOGGER = LoggerFactory.getLogger(RequestLoggingFilter.class);
    private static final int MAX_REQUEST_BODY_LENGTH = 4096;
    private static final int MAX_RESPONSE_BODY_LENGTH = 8192;
    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain chain
    ) throws ServletException, IOException {
        ContentCachingRequestWrapper requestWrapper = new ContentCachingRequestWrapper(request);
        ContentCachingResponseWrapper responseWrapper = new ContentCachingResponseWrapper(response);
        long start = System.currentTimeMillis();
        try {
            chain.doFilter(requestWrapper, responseWrapper);
        } finally {
            long elapsedMs = System.currentTimeMillis() - start;
            try {
                if (LOGGER.isDebugEnabled()) {
                    logRequestAndResponse(requestWrapper, responseWrapper, elapsedMs);
                }
            } finally {
                responseWrapper.copyBodyToResponse();
            }
        }
    }
    private void logRequestAndResponse(
            ContentCachingRequestWrapper request,
            ContentCachingResponseWrapper response,
            long elapsedMs
    ) throws UnsupportedEncodingException {
        String uri = request.getRequestURI();
        String contentType = response.getContentType();
        String requestBody = getRequestBody(request);
        String responseBody = shouldLogResponseBody(uri, contentType)
                ? getResponseBody(response)
                : " - ";
        LOGGER.debug(
                "\n[REQUEST] {} {} - status={} - elapsed={}s\nHeaders : {}\nRequest : {}\nResponse : {}\n",
                request.getMethod(),
                uri,
                response.getStatus(),
                elapsedMs / 1000.0D,
                getMaskedHeaders(request),
                requestBody,
                responseBody
        );
    }
    private Map<String, String> getMaskedHeaders(HttpServletRequest request) {
        Map<String, String> headers = new LinkedHashMap<>();
        Enumeration<String> names = request.getHeaderNames();
        if (names == null) {
            return Collections.emptyMap();
        }
        while (names.hasMoreElements()) {
            String name = names.nextElement();
            String value = request.getHeader(name);
            if (isSensitiveHeader(name)) {
                value = "**********";
            }
            headers.put(name, value);
        }
        return headers;
    }
    private boolean isSensitiveHeader(String name) {
        return "authorization".equalsIgnoreCase(name)
                || "cookie".equalsIgnoreCase(name)
                || "set-cookie".equalsIgnoreCase(name)
                || "proxy-authorization".equalsIgnoreCase(name)
                || "x-auth-token".equalsIgnoreCase(name);
    }
    private String getRequestBody(ContentCachingRequestWrapper request) throws UnsupportedEncodingException {
        byte[] buf = request.getContentAsByteArray();
        if (buf.length == 0) {
            return " - ";
        }
        Charset charset = getCharset(request.getCharacterEncoding());
        String body = new String(buf, charset);
        body = truncate(body, MAX_REQUEST_BODY_LENGTH);
        return maskSensitiveBody(body);
    }
    private String getResponseBody(ContentCachingResponseWrapper response) throws UnsupportedEncodingException {
        byte[] buf = response.getContentAsByteArray();
        if (buf.length == 0) {
            return " - ";
        }
        Charset charset = getCharset(response.getCharacterEncoding());
        String body = new String(buf, charset);
        body = truncate(body, MAX_RESPONSE_BODY_LENGTH);
        return maskSensitiveBody(body);
    }
    private Charset getCharset(String encoding) {
        if (encoding == null || encoding.trim().isEmpty()) {
            return StandardCharsets.UTF_8;
        }
        try {
            return Charset.forName(encoding);
        } catch (Exception e) {
            return StandardCharsets.UTF_8;
        }
    }
    private boolean shouldLogResponseBody(String uri, String contentType) {
        if (uri == null) {
            return false;
        }
        if (uri.contains("/comm/ajax/comm/getDispCategoryAllFront.do")) {
            return false;
        }
        if (uri.contains("/file/") || uri.contains("/download")) {
            return false;
        }
        if (contentType == null) {
            return false;
        }
        String ct = contentType.toLowerCase();
        return ct.contains("application/json")
                || ct.contains("application/xml")
                || ct.contains("text/plain");
    }
    private String truncate(String value, int maxLength) {
        if (value == null) {
            return " - ";
        }
        if (value.length() <= maxLength) {
            return value;
        }
        return value.substring(0, maxLength) + "...[TRUNCATED]";
    }
    private String maskSensitiveBody(String body) {
        if (body == null) {
            return " - ";
        }
        String masked = body;
        masked = masked.replaceAll("(?i)(\"(?:mberPwd|password|pwd|newPassword|confirmPassword)\"\\s*:\\s*\")([^\"]+)(\")", "$1**********$3");
        masked = masked.replaceAll("(?i)((?:mberPwd|password|pwd|newPassword|confirmPassword)=)([^&\\s]+)", "$1**********");
        masked = masked.replaceAll("(?i)(\"(?:accessToken|refreshToken|token)\"\\s*:\\s*\")([^\"]+)(\")", "$1**********$3");
        return masked;
    }
}
```

#### 2-1-9. 개선 코드 검증 포인트

|검증 항목|반영 여부|
|---|---|
|응답 복사 보장|`finally` 안에서 `copyBodyToResponse()` 호출|
|DEBUG OFF 성능|`LOGGER.isDebugEnabled()`일 때만 body/header 생성|
|대용량 방지|request/response body 길이 제한|
|바이너리 응답 제외|JSON/XML/text만 응답 body 로깅|
|Header 보안|`Authorization`, `Cookie`, `Set-Cookie` 마스킹|
|Body 보안|password/token 계열 마스킹|
|인코딩|request/response characterEncoding 우선 사용|
|Spring 5.3 호환|`javax.servlet` 기반|
|중복 필터 실행 완화|`OncePerRequestFilter` 사용|

#### 2-1-10. Spring 설정 시 주의

`web.xml`에 직접 등록되어 있다면 필터 순서가 중요합니다.

```xml
<filter>
    <filter-name>requestLoggingFilter</filter-name>
    <filter-class>com.example.RequestLoggingFilter</filter-class>
</filter>
<filter-mapping>
    <filter-name>requestLoggingFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

주의할 점:

|항목|설명|
|---|---|
|인증 필터 앞|인증 전 요청까지 로깅되어 민감정보 노출 가능|
|인증 필터 뒤|로그인 성공 후 사용자 요청 분석에 유리|
|EncodingFilter 뒤|문자셋 적용 후 로깅 가능|
|파일 다운로드 URL|필터 제외 권장|
|정적 리소스|`/css/*`, `/js/*`, `/images/*` 제외 권장|
|Health Check|`/health`, `/actuator`, `/internal/metrics` 제외 권장|

#### 2-1-11. 커머스 운영 환경에서 특히 주의할 API

아래 URI는 body 로깅을 끄거나 강한 마스킹이 필요합니다.

```text
/login
/logout
/join
/member
/order
/payment
/cart
/coupon
/point
/mypage
/address
/file/download
/excel
```

커머스 시스템에서는 request/response에 회원 정보, 배송지, 주문 상품, 쿠폰, 포인트, 결제 결과, 세션 정보가 섞일 수 있습니다. 따라서 “장애 분석용 전체 body 로그”는 운영에서 상시 켜기보다 **특정 URI·특정 시간·특정 traceId 기준으로 제한**하는 편이 안전합니다.

#### 2-1-12. AA 관점 최종 권고

|구분|권고|
|---|---|
|현재 코드 사용|개발/QA 분석용으로만 제한 권장|
|운영 상시 사용|현재 형태로는 비권장|
|운영 적용 조건|DEBUG guard, 마스킹, 크기 제한, Content-Type 제한, URI exclude 필수|
|응답 wrapper|`copyBodyToResponse()`는 반드시 `finally`에서 보장|
|보안|Header/Body/Response 모두 마스킹 대상 확대|
|성능|대용량 response body 캐싱 금지|
|구조|`Filter`보다 `OncePerRequestFilter` 권장|
|관측성|body 로그보다 `traceId`, URI, status, elapsed time, user key hash 중심 권장|

최종적으로 이 필터는 **장애 분석용 HTTP 전문 로깅 필터**로는 유용하지만, 현재 형태는 운영 커머스 서비스에 그대로 적용하기에는 위험합니다. 운영에서는 body 전체 로깅보다 `요청 ID`, `URI`, `method`, `status`, `elapsed time`, `client IP`, `user/session 식별자의 마스킹 값` 중심으로 남기고, request/response body는 필요한 URI에 한해 제한적으로 남기는 구조가 더 안전합니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
