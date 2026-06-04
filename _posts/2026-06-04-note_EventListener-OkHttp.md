---
layout: single
title: "EventListener-OkHttp"
excerpt: "EventListener-OkHttp"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-06-04"
last_modified_at: "2026-06-04 13:31:54 +0900"
mermaid: false
---
### 1. OkHttp `EventListener` 기반 Socket Connection 추적 + Proxy 설정 예제

#### 1-1. 핵심 결론

`EventListener`를 사용하면 OkHttp 호출 과정에서 다음을 추적할 수 있습니다.
- DNS 조회 발생 여부
- TCP connect 발생 여부
- Proxy 경유 여부
- 기존 connection 재사용 여부
- connection 획득/반환 여부
- connection 실패/종료 여부
- 요청별 소요 시간  
    공식 문서 기준 `EventListener`는 HTTP Call의 수량·크기·시간 같은 metrics event를 모니터링하기 위한 listener이고, `connectionAcquired`, `connectionReleased`는 Call이 connection을 획득/반환할 때 호출됩니다. `ConnectionPool`은 같은 `Address`를 공유하는 요청이 connection을 공유할 수 있도록 HTTP/HTTP2 connection을 재사용합니다. ([Square Open Source](https://square.github.io/okhttp/5.x/okhttp/okhttp3/-event-listener/index.html?utm_source=chatgpt.com "EventListener"))

---

### 2. Connection Pool 재사용 여부 판단 기준

|이벤트|의미|해석|
|---|---|---|
|`dnsStart/dnsEnd`|DNS 조회|매번 발생하면 신규 연결 가능성 증가|
|`connectStart/connectEnd`|TCP 연결 생성|발생하면 새 socket 생성|
|`secureConnectStart/End`|HTTPS TLS handshake|발생하면 새 TLS 연결|
|`connectionAcquired`|Call이 connection 획득|신규/재사용 모두 발생|
|`connectionReleased`|Call이 connection 반환|Response close 후 정상 반환 신호|
|`connectionClosed`|socket connection 종료|pool 유지 실패 또는 idle 정리/서버 종료|
|`callEnd`|Call 정상 종료|요청 완료|
|`callFailed`|Call 실패|timeout/reset 등|
|중요한 판단:|||

```text
connectStart 없이 connectionAcquired만 발생
= 기존 Pool connection 재사용 가능성이 높음
```

```text
매 요청마다 dnsStart + connectStart + secureConnectStart 발생
= Pool 재사용이 거의 안 되고 신규 연결을 계속 만드는 패턴
```

---

### 3. 실무용 `EventListener` 구현

#### 3-1. 요청별 식별값 생성용 Factory

`EventListener`는 Call 단위로 생성해서 요청별 trace id를 부여하는 방식이 좋습니다.

```java
package com.example.http;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.Proxy;
import java.util.List;
import java.util.concurrent.atomic.AtomicLong;

import okhttp3.Call;
import okhttp3.Connection;
import okhttp3.EventListener;
import okhttp3.Handshake;
import okhttp3.Protocol;
import okhttp3.Request;
import okhttp3.Response;
import okhttp3.Route;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class OkHttpConnectionEventListener extends EventListener {
    private static final Logger log =
        LoggerFactory.getLogger(OkHttpConnectionEventListener.class);
    private static final AtomicLong SEQ = new AtomicLong(0);
    private final long callId;
    private final long startNanos;
    private OkHttpConnectionEventListener(long callId) {
        this.callId = callId;
        this.startNanos = System.nanoTime();
    }
    public static Factory factory() {
        return call -> new OkHttpConnectionEventListener(SEQ.incrementAndGet());
    }
    @Override
    public void callStart(Call call) {
        Request request = call.request();
        log.info("[OKHTTP][{}] callStart method={} url={}",
            callId, request.method(), maskUrl(request.url().toString()));
    }
    @Override
    public void dnsStart(Call call, String domainName) {
        log.info("[OKHTTP][{}] dnsStart domain={}", callId, domainName);
    }
    @Override
    public void dnsEnd(Call call, String domainName, List<java.net.InetAddress> inetAddressList) {
        log.info("[OKHTTP][{}] dnsEnd domain={} addresses={}",
            callId, domainName, inetAddressList);
    }
    @Override
    public void connectStart(
            Call call,
            InetSocketAddress inetSocketAddress,
            Proxy proxy) {
        log.info("[OKHTTP][{}] connectStart remote={} proxy={}",
            callId,
            inetSocketAddress,
            proxyInfo(proxy));
    }
    @Override
    public void secureConnectStart(Call call) {
        log.info("[OKHTTP][{}] secureConnectStart", callId);
    }
    @Override
    public void secureConnectEnd(Call call, Handshake handshake) {
        log.info("[OKHTTP][{}] secureConnectEnd tlsVersion={} cipherSuite={}",
            callId,
            handshake != null ? handshake.tlsVersion() : null,
            handshake != null ? handshake.cipherSuite() : null);
    }
    @Override
    public void connectEnd(
            Call call,
            InetSocketAddress inetSocketAddress,
            Proxy proxy,
            Protocol protocol) {
        log.info("[OKHTTP][{}] connectEnd remote={} proxy={} protocol={}",
            callId,
            inetSocketAddress,
            proxyInfo(proxy),
            protocol);
    }
    @Override
    public void connectFailed(
            Call call,
            InetSocketAddress inetSocketAddress,
            Proxy proxy,
            Protocol protocol,
            IOException ioe) {
        log.warn("[OKHTTP][{}] connectFailed remote={} proxy={} protocol={} error={}:{}",
            callId,
            inetSocketAddress,
            proxyInfo(proxy),
            protocol,
            ioe.getClass().getSimpleName(),
            ioe.getMessage());
    }
    @Override
    public void connectionAcquired(Call call, Connection connection) {
        Route route = connection.route();
        log.info("[OKHTTP][{}] connectionAcquired routeAddress={} proxy={} socketAddress={} protocol={}",
            callId,
            route.address().url(),
            proxyInfo(route.proxy()),
            route.socketAddress(),
            connection.protocol());
    }
    @Override
    public void connectionReleased(Call call, Connection connection) {
        Route route = connection.route();
        log.info("[OKHTTP][{}] connectionReleased routeAddress={} proxy={} socketAddress={} protocol={}",
            callId,
            route.address().url(),
            proxyInfo(route.proxy()),
            route.socketAddress(),
            connection.protocol());
    }
    @Override
    public void connectionClosed(Call call, Connection connection) {
        Route route = connection.route();
        log.info("[OKHTTP][{}] connectionClosed routeAddress={} proxy={} socketAddress={} protocol={}",
            callId,
            route.address().url(),
            proxyInfo(route.proxy()),
            route.socketAddress(),
            connection.protocol());
    }
    @Override
    public void requestHeadersStart(Call call) {
        log.info("[OKHTTP][{}] requestHeadersStart", callId);
    }
    @Override
    public void requestHeadersEnd(Call call, Request request) {
        log.info("[OKHTTP][{}] requestHeadersEnd", callId);
    }
    @Override
    public void responseHeadersStart(Call call) {
        log.info("[OKHTTP][{}] responseHeadersStart", callId);
    }
    @Override
    public void responseHeadersEnd(Call call, Response response) {
        log.info("[OKHTTP][{}] responseHeadersEnd status={}", callId, response.code());
    }
    @Override
    public void callEnd(Call call) {
        log.info("[OKHTTP][{}] callEnd elapsedMs={}", callId, elapsedMs());
    }
    @Override
    public void callFailed(Call call, IOException ioe) {
        log.warn("[OKHTTP][{}] callFailed elapsedMs={} error={}:{}",
            callId,
            elapsedMs(),
            ioe.getClass().getSimpleName(),
            ioe.getMessage());
    }
    private long elapsedMs() {
        return (System.nanoTime() - startNanos) / 1_000_000L;
    }
    private static String proxyInfo(Proxy proxy) {
        if (proxy == null) {
            return "null";
        }
        return proxy.type() + "@" + proxy.address();
    }
    private static String maskUrl(String url) {
        int queryIndex = url.indexOf('?');
        if (queryIndex < 0) {
            return url;
        }
        return url.substring(0, queryIndex) + "?...";
    }
}
```

주의:

- 운영 로그에 검색어, 토큰, 개인정보가 포함될 수 있으므로 URL query/body 전체를 남기지 않는 것이 안전합니다.

- `connectionAcquired`는 신규 connection과 재사용 connection 모두에서 발생합니다.

- “재사용” 판단은 `connectStart` 없이 `connectionAcquired`가 발생하는지로 봅니다.

---

### 4. Proxy 미사용 + EventListener 적용 `OkHttpConfig`

```java
package com.example.config;

import java.io.IOException;
import java.util.concurrent.TimeUnit;

import javax.annotation.PreDestroy;

import com.example.http.OkHttpConnectionEventListener;

import okhttp3.ConnectionPool;
import okhttp3.OkHttpClient;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OkHttpConfig {
    private OkHttpClient searchOkHttpClient;
    @Bean(name = "searchOkHttpClient")
    public OkHttpClient searchOkHttpClient() {
        this.searchOkHttpClient = new OkHttpClient.Builder()
            .cache(null)
            .connectionPool(new ConnectionPool(20, 120, TimeUnit.SECONDS))
            .connectTimeout(3, TimeUnit.SECONDS)
            .readTimeout(5, TimeUnit.SECONDS)
            .writeTimeout(5, TimeUnit.SECONDS)
            .callTimeout(10, TimeUnit.SECONDS)
            .retryOnConnectionFailure(true)
            .eventListenerFactory(OkHttpConnectionEventListener.factory())
            .build();
        return this.searchOkHttpClient;
    }
    @PreDestroy
    public void destroy() throws IOException {
        if (this.searchOkHttpClient == null) {
            return;
        }
        this.searchOkHttpClient.dispatcher().executorService().shutdown();
        this.searchOkHttpClient.connectionPool().evictAll();
        if (this.searchOkHttpClient.cache() != null) {
            this.searchOkHttpClient.cache().close();
        }
    }
}
```

---

### 5. Proxy 사용 방식 1: 고정 HTTP Proxy

OkHttp `Builder.proxy(Proxy)`는 이 client가 생성하는 connection에 사용할 HTTP proxy를 설정합니다. 공식 Builder 문서 기준 `proxy(...)`는 `proxySelector`보다 우선하고, proxy를 완전히 끄려면 `proxy(Proxy.NO_PROXY)`를 사용합니다. ([Square Open Source](https://square.github.io/okhttp/3.x/okhttp/okhttp3/OkHttpClient.Builder.html?utm_source=chatgpt.com "OkHttpClient.Builder (OkHttp 3.14.0 API)"))

```java
package com.example.config;

import java.net.InetSocketAddress;
import java.net.Proxy;
import java.util.concurrent.TimeUnit;

import com.example.http.OkHttpConnectionEventListener;

import okhttp3.ConnectionPool;
import okhttp3.OkHttpClient;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OkHttpProxyConfig {
    @Bean(name = "searchOkHttpClient")
    public OkHttpClient searchOkHttpClient() {
        Proxy proxy = new Proxy(
            Proxy.Type.HTTP,
            new InetSocketAddress("10.10.10.50", 3128)
        );
        return new OkHttpClient.Builder()
            .cache(null)
            .proxy(proxy)
            .connectionPool(new ConnectionPool(20, 120, TimeUnit.SECONDS))
            .connectTimeout(3, TimeUnit.SECONDS)
            .readTimeout(5, TimeUnit.SECONDS)
            .writeTimeout(5, TimeUnit.SECONDS)
            .callTimeout(10, TimeUnit.SECONDS)
            .retryOnConnectionFailure(true)
            .eventListenerFactory(OkHttpConnectionEventListener.factory())
            .build();
    }
}
```

## 6. 이 경우 EventListener 로그의 `connectStart remote`는 목적지 서버가 아니라 **Proxy 서버 주소**로 보일 수 있습니다. HTTPS 요청이면 OkHttp는 proxy에 `CONNECT` 터널을 만들고 이후 TLS handshake를 수행합니다.

### 6-1. Proxy 사용 방식 2: 인증 Proxy

Proxy가 Basic 인증을 요구하면 `proxyAuthenticator`를 사용합니다.

```java
package com.example.config;

import java.net.InetSocketAddress;
import java.net.Proxy;
import java.util.concurrent.TimeUnit;

import com.example.http.OkHttpConnectionEventListener;

import okhttp3.Authenticator;
import okhttp3.ConnectionPool;
import okhttp3.Credentials;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.Response;
import okhttp3.Route;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OkHttpAuthenticatedProxyConfig {
    @Bean(name = "searchOkHttpClient")
    public OkHttpClient searchOkHttpClient() {
        Proxy proxy = new Proxy(
            Proxy.Type.HTTP,
            new InetSocketAddress("10.10.10.50", 3128)
        );
        Authenticator proxyAuthenticator = new Authenticator() {
            @Override
            public Request authenticate(Route route, Response response) {
                if (response.request().header("Proxy-Authorization") != null) {
                    return null;
                }
                String credential = Credentials.basic("proxyUser", "proxyPassword");
                return response.request().newBuilder()
                    .header("Proxy-Authorization", credential)
                    .build();
            }
        };
        return new OkHttpClient.Builder()
            .cache(null)
            .proxy(proxy)
            .proxyAuthenticator(proxyAuthenticator)
            .connectionPool(new ConnectionPool(20, 120, TimeUnit.SECONDS))
            .connectTimeout(3, TimeUnit.SECONDS)
            .readTimeout(5, TimeUnit.SECONDS)
            .writeTimeout(5, TimeUnit.SECONDS)
            .callTimeout(10, TimeUnit.SECONDS)
            .retryOnConnectionFailure(true)
            .eventListenerFactory(OkHttpConnectionEventListener.factory())
            .build();
    }
}
```

주의:

- Proxy 계정/비밀번호는 코드 하드코딩 금지

- Spring property, 환경변수, Vault 등으로 분리 권장

- 로그에 `Proxy-Authorization` 헤더가 찍히지 않게 주의

---

### 6-2. Proxy 사용 방식 3: Proxy 완전 미사용 강제

OS/JVM proxy 설정이 있을 수 있는 환경에서 특정 client는 proxy를 절대 타지 않게 하려면 다음처럼 설정합니다.

```java
OkHttpClient client = new OkHttpClient.Builder()
    .proxy(Proxy.NO_PROXY)
    .connectionPool(new ConnectionPool(20, 120, TimeUnit.SECONDS))
    .eventListenerFactory(OkHttpConnectionEventListener.factory())
    .build();
```

## 7. 공식 문서 기준 `proxy(Proxy.NO_PROXY)`는 proxy 사용을 완전히 비활성화하는 방식입니다. ([Square Open Source](https://square.github.io/okhttp/3.x/okhttp/okhttp3/OkHttpClient.Builder.html?utm_source=chatgpt.com "OkHttpClient.Builder (OkHttp 3.14.0 API)"))

### 7-1. Proxy 사용 방식 4: 목적지별 ProxySelector

도메인별로 proxy 사용 여부를 다르게 하고 싶으면 `ProxySelector`를 사용할 수 있습니다.

```java
package com.example.http;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.Proxy;
import java.net.ProxySelector;
import java.net.SocketAddress;
import java.net.URI;
import java.util.Collections;
import java.util.List;

public class DomainProxySelector extends ProxySelector {
    private final Proxy searchProxy =
        new Proxy(Proxy.Type.HTTP, new InetSocketAddress("10.10.10.50", 3128));
    @Override
    public List<Proxy> select(URI uri) {
        String host = uri.getHost();
        if (host == null) {
            return Collections.singletonList(Proxy.NO_PROXY);
        }
        if (host.endsWith(".external-api.com")) {
            return Collections.singletonList(searchProxy);
        }
        return Collections.singletonList(Proxy.NO_PROXY);
    }
    @Override
    public void connectFailed(URI uri, SocketAddress sa, IOException ioe) {
        // 필요 시 로그 처리
    }
}
```

```java
OkHttpClient client = new OkHttpClient.Builder()
    .proxySelector(new DomainProxySelector())
    .connectionPool(new ConnectionPool(20, 120, TimeUnit.SECONDS))
    .eventListenerFactory(OkHttpConnectionEventListener.factory())
    .build();
```

주의:

- `.proxy(proxy)`를 설정하면 `.proxySelector(...)`보다 우선합니다.

- 목적지별 분기 정책이 필요하면 `.proxy(...)`를 쓰지 말고 `.proxySelector(...)`만 사용합니다. ([Square Open Source](https://square.github.io/okhttp/3.x/okhttp/okhttp3/OkHttpClient.Builder.html?utm_source=chatgpt.com "OkHttpClient.Builder (OkHttp 3.14.0 API)"))

---

### 7-2. `sendAutoSearch` 사용 예제

```java
package com.example.autosearch;

import java.io.IOException;

import com.google.gson.Gson;

import okhttp3.MediaType;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.RequestBody;
import okhttp3.Response;
import okhttp3.ResponseBody;

import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;

@Component
public class AutoSearchApiClient {
    private static final MediaType JSON_MEDIA_TYPE =
        MediaType.parse("application/json; charset=utf-8");
    private final OkHttpClient okHttpClient;
    private final Gson gson;
    private final String autoSearchUrl;
    public AutoSearchApiClient(
            @Qualifier("searchOkHttpClient") OkHttpClient okHttpClient,
            Gson gson,
            @Value("${api.autosearch.url}") String autoSearchUrl) {
        this.okHttpClient = okHttpClient;
        this.gson = gson;
        this.autoSearchUrl = autoSearchUrl;
    }
    public AutoSearchResponse sendAutoSearch(String keyword) throws IOException {
        if (!StringUtils.hasText(keyword)) {
            return AutoSearchResponse.empty();
        }
        AutoSearchRequest requestDto =
            new AutoSearchRequest(keyword.trim(), 10);
        RequestBody requestBody = RequestBody.create(
            JSON_MEDIA_TYPE,
            gson.toJson(requestDto)
        );
        Request request = new Request.Builder()
            .url(autoSearchUrl)
            .post(requestBody)
            .header("Accept", "application/json")
            .build();
        try (Response response = okHttpClient.newCall(request).execute()) {
            ResponseBody responseBody = response.body();
            String responseText = responseBody != null ? responseBody.string() : "";
            if (!response.isSuccessful()) {
                throw new IOException(
                    "AutoSearch API failed. status=" + response.code()
                        + ", body=" + abbreviate(responseText, 1000)
                );
            }
            return gson.fromJson(responseText, AutoSearchResponse.class);
        }
    }
    private static String abbreviate(String value, int maxLength) {
        if (value == null) {
            return "";
        }
        return value.length() <= maxLength
            ? value
            : value.substring(0, maxLength) + "...";
    }
}
```

## 8. 핵심은 `try (Response response = ...)`입니다. 이것이 없으면 `connectionReleased`가 늦거나 발생하지 않고, `CLOSE-WAIT`/FD 누수로 이어질 수 있습니다.

### 8-1. 로그 해석 예시

#### 8-1-1. 신규 연결 생성 패턴

```text
[OKHTTP][101] callStart method=POST url=https://search.example.com/api?...
[OKHTTP][101] dnsStart domain=search.example.com
[OKHTTP][101] dnsEnd domain=search.example.com addresses=[/10.10.10.21]
[OKHTTP][101] connectStart remote=search.example.com/10.10.10.21:443 proxy=DIRECT@null
[OKHTTP][101] secureConnectStart
[OKHTTP][101] secureConnectEnd tlsVersion=TLS_1_2 cipherSuite=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
[OKHTTP][101] connectEnd remote=search.example.com/10.10.10.21:443 proxy=DIRECT@null protocol=http/1.1
[OKHTTP][101] connectionAcquired routeAddress=https://search.example.com:443 proxy=DIRECT@null socketAddress=/10.10.10.21:443 protocol=http/1.1
[OKHTTP][101] connectionReleased routeAddress=https://search.example.com:443 proxy=DIRECT@null socketAddress=/10.10.10.21:443 protocol=http/1.1
[OKHTTP][101] callEnd elapsedMs=87
```

해석:

- DNS/TCP/TLS가 모두 발생

- 신규 connection 생성

- 요청 완료 후 connection 반환

#### 8-1-2. Pool 재사용 패턴

```text
[OKHTTP][102] callStart method=POST url=https://search.example.com/api?...
[OKHTTP][102] connectionAcquired routeAddress=https://search.example.com:443 proxy=DIRECT@null socketAddress=/10.10.10.21:443 protocol=http/1.1
[OKHTTP][102] requestHeadersStart
[OKHTTP][102] responseHeadersEnd status=200
[OKHTTP][102] connectionReleased routeAddress=https://search.example.com:443 proxy=DIRECT@null socketAddress=/10.10.10.21:443 protocol=http/1.1
[OKHTTP][102] callEnd elapsedMs=12
```

해석:

- `dnsStart`, `connectStart`, `secureConnectStart` 없음

- 기존 Pool connection 재사용 가능성이 높음

#### 8-1-3. Proxy 경유 패턴

```text
[OKHTTP][201] connectStart remote=/10.10.10.50:3128 proxy=HTTP@/10.10.10.50:3128
[OKHTTP][201] connectEnd remote=/10.10.10.50:3128 proxy=HTTP@/10.10.10.50:3128 protocol=http/1.1
[OKHTTP][201] connectionAcquired routeAddress=https://external-api.com:443 proxy=HTTP@/10.10.10.50:3128 socketAddress=/10.10.10.50:3128 protocol=http/1.1
```

해석:

- 실제 TCP 연결 대상은 proxy 서버

- `routeAddress`는 원 목적지

- Proxy 서버까지의 connection이 OkHttp Pool 재사용 대상

---

### 8-2. 운영 적용 시 주의점

|항목|주의|
|---|---|
|로그량|부하 테스트 시 이벤트 로그가 대량 발생하므로 sampling 또는 DEBUG 레벨 권장|
|개인정보|URL query/body/header 전체 로그 금지|
|Proxy 인증|`Proxy-Authorization` 로그 노출 금지|
|HTTP/2|하나의 connection에서 여러 stream이 동시 처리될 수 있어 `ESTAB` 수가 요청 수보다 훨씬 적을 수 있음|
|Redirect|하나의 Call에서 `connectionAcquired`가 여러 번 발생할 수 있음|
|Retry|`retryOnConnectionFailure(true)`면 실패 후 재연결 이벤트가 추가 발생 가능|
|비동기 호출|`Dispatcher` 설정과 함께 봐야 함|
|동기 호출|WAS Thread 점유와 `callTimeout`을 같이 봐야 함|

---

### 8-3. 권장 로그 레벨 설정

운영 상시 적용보다는 장애 분석 기간에만 켜는 것을 권장합니다.

```xml
<logger name="com.example.http.OkHttpConnectionEventListener" level="DEBUG"/>
```

코드 로그도 `info` 대신 `debug`를 기본으로 두는 편이 안전합니다.

```java
if (log.isDebugEnabled()) {
    log.debug("[OKHTTP][{}] connectionAcquired routeAddress={} proxy={} socketAddress={} protocol={}",
        callId,
        route.address().url(),
        proxyInfo(route.proxy()),
        route.socketAddress(),
        connection.protocol());
}
```

## 9. 부하 테스트 중에는 `INFO`로 잠깐 올려도 되지만, 운영 상시는 `DEBUG` 또는 sampling이 안전합니다.

### 9-1. 최종 적용 조합

```java
@Bean(name = "searchOkHttpClient")
public OkHttpClient searchOkHttpClient() {
    return new OkHttpClient.Builder()
        .cache(null)
        .proxy(Proxy.NO_PROXY) // proxy 미사용 강제. 필요 시 HTTP Proxy로 교체
        .connectionPool(new ConnectionPool(20, 120, TimeUnit.SECONDS))
        .connectTimeout(3, TimeUnit.SECONDS)
        .readTimeout(5, TimeUnit.SECONDS)
        .writeTimeout(5, TimeUnit.SECONDS)
        .callTimeout(10, TimeUnit.SECONDS)
        .retryOnConnectionFailure(true)
        .eventListenerFactory(OkHttpConnectionEventListener.factory())
        .build();
}
```

Proxy 사용 시:

```java
Proxy proxy = new Proxy(
    Proxy.Type.HTTP,
    new InetSocketAddress("10.10.10.50", 3128)
);
@Bean(name = "searchOkHttpClient")
public OkHttpClient searchOkHttpClient() {
    return new OkHttpClient.Builder()
        .cache(null)
        .proxy(proxy)
        .connectionPool(new ConnectionPool(20, 120, TimeUnit.SECONDS))
        .connectTimeout(3, TimeUnit.SECONDS)
        .readTimeout(5, TimeUnit.SECONDS)
        .writeTimeout(5, TimeUnit.SECONDS)
        .callTimeout(10, TimeUnit.SECONDS)
        .retryOnConnectionFailure(true)
        .eventListenerFactory(OkHttpConnectionEventListener.factory())
        .build();
}
```

---

### 9-2. 최종 판단 기준

```text
Pool 재사용 확인:
connectStart 없이 connectionAcquired 발생 여부 확인

Response close 확인:
connectionReleased가 callEnd 전에 정상적으로 발생하는지 확인

Proxy 사용 확인:
connectStart remote가 proxy 서버 주소인지 확인

비정상 종료 확인:
connectFailed, callFailed, connectionClosed 발생 시점과 ss 로그의 TIME-WAIT/CLOSE-WAIT를 대조
```

이 구성을 적용하면 Postman 부하 테스트 중에도 **신규 socket 생성 수**, **Pool 재사용 여부**, **Proxy 경유 여부**, **Response close 후 connection 반환 여부**를 로그로 직접 확인할 수 있습니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
