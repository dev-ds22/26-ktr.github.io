---
layout: single
title: "OkHttpClient_설정"
excerpt: "OkHttpClient_설정"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-07-06"
last_modified_at: "2026-07-06 18:03:55 +0900"
mermaid: false
---
### 1. 결론

현재 수준에서는 **Config 파일을 새로 만들기보다 기존 `OkHttpConfig` 파일에 3개 Bean을 함께 관리하는 방식**이 더 좋습니다. 단, 기존 파일의 설명이 “상품추천 API 전용”으로 되어 있으므로, **외부 HTTP 연동용 OkHttpClient 공통 설정 파일**로 의미를 바꾸는 것이 좋습니다. 첨부 파일은 현재 `productRecommenderOkHttpClient`와 해당 Client를 종료하는 `DisposableBean`만 정의된 구조입니다.  
Spring Java Config는 `@Configuration` 클래스 안에 `@Bean` 메서드로 여러 Bean을 정의하는 방식이 표준이며, Bean 식별자는 컨테이너 내에서 유일해야 합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/core.html "Core Technologies")) 같은 `OkHttpClient` 타입 Bean이 여러 개 생기면 타입만으로 주입할 수 없으므로 `@Qualifier`를 명확히 사용해야 합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/core.html "Core Technologies"))

### 2. Config 파일 분리 판단 기준

|구분|권장 방식|이유|
|---|--:|---|
|`product/search/mail` 3개 수준|기존 `OkHttpConfig`에 추가|모두 외부 HTTP Client 설정이라는 같은 관심사|
|Client 수가 5~10개 이상 증가|도메인별 Config 분리|파일 비대화 방지|
|상품추천/검색추천이 같은 외부 시스템, 같은 timeout 정책|하나의 `recommenderOkHttpClient` 공유|불필요한 Pool 분리 방지|
|상품추천/검색추천의 timeout, 장애격리, 호출량이 다름|Bean 분리|ConnectionPool, timeout, 장애 영향 분리|
|Mail 발송 API|별도 Bean 권장|추천 API와 응답시간, 장애 영향, 재시도 정책이 다름|

### 3. 추천 구조

|Bean 이름|용도|분리 이유|
|---|---|---|
|`productRecommenderOkHttpClient`|상품 추천 API|빠른 응답 필요, 짧은 timeout|
|`searchRecommenderOkHttpClient`|검색 추천/자동완성 API|사용자 입력 흐름에 직접 영향, 더 짧은 timeout 가능|
|`mailSendOkHttpClient`|메일 발송 API|추천 API보다 응답 시간이 길 수 있고 장애격리 필요|

### 4. 실무 적용 예제: `OkHttpConfig.java`

```java
package bk.comm.config;
import java.io.IOException;
import java.util.concurrent.TimeUnit;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.DisposableBean;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import okhttp3.ConnectionPool;
import okhttp3.OkHttpClient;
/**
 * 외부 HTTP 연동을 위한 OkHttpClient 설정
 * - 외부 시스템 / timeout 정책 / 장애격리 기준으로 Client Bean 분리
 * - 동일 정책의 URL은 같은 OkHttpClient를 재사용
 */
@Configuration
public class OkHttpConfig {
    private static final Logger LOGGER = LoggerFactory.getLogger(OkHttpConfig.class);
    public static final String PRODUCT_RECOMMENDER_OKHTTP_CLIENT = "productRecommenderOkHttpClient";
    public static final String SEARCH_RECOMMENDER_OKHTTP_CLIENT = "searchRecommenderOkHttpClient";
    public static final String MAIL_SEND_OKHTTP_CLIENT = "mailSendOkHttpClient";
    /**
     * 상품 추천 API용 OkHttpClient
     * - 화면 응답에 직접 영향
     * - 짧은 timeout 적용
     */
    @Bean(name = PRODUCT_RECOMMENDER_OKHTTP_CLIENT)
    public OkHttpClient productRecommenderOkHttpClient() {
        return buildClient(
            PRODUCT_RECOMMENDER_OKHTTP_CLIENT,
            120,    // maxIdleConnections
            60,     // keepAliveSeconds
            1,      // connectTimeoutSeconds
            2,      // readTimeoutSeconds
            2,      // writeTimeoutSeconds
            false   // retryOnConnectionFailure
        );
    }
    /**
     * 검색 추천 / 자동완성 API용 OkHttpClient
     * - 사용자 입력 중 호출될 가능성이 높음
     * - 상품 추천보다 더 짧은 응답 기준 적용 가능
     */
    @Bean(name = SEARCH_RECOMMENDER_OKHTTP_CLIENT)
    public OkHttpClient searchRecommenderOkHttpClient() {
        return buildClient(
            SEARCH_RECOMMENDER_OKHTTP_CLIENT,
            80,
            60,
            1,
            1,
            1,
            false
        );
    }
    /**
     * 메일 발송 API용 OkHttpClient
     * - 추천 API와 장애 영향 분리
     * - 외부 메일 서버 응답 지연 가능성을 고려하여 timeout을 다르게 설정
     */
    @Bean(name = MAIL_SEND_OKHTTP_CLIENT)
    public OkHttpClient mailSendOkHttpClient() {
        return buildClient(
            MAIL_SEND_OKHTTP_CLIENT,
            20,
            120,
            3,
            10,
            5,
            false
        );
    }
    private OkHttpClient buildClient(String beanName,
                                     int maxIdleConnections,
                                     long keepAliveSeconds,
                                     long connectTimeoutSeconds,
                                     long readTimeoutSeconds,
                                     long writeTimeoutSeconds,
                                     boolean retryOnConnectionFailure) {
        ConnectionPool connectionPool = new ConnectionPool(
            maxIdleConnections,
            keepAliveSeconds,
            TimeUnit.SECONDS
        );
        OkHttpClient client = new OkHttpClient.Builder()
            .connectionPool(connectionPool)
            .cache(null)
            .connectTimeout(connectTimeoutSeconds, TimeUnit.SECONDS)
            .readTimeout(readTimeoutSeconds, TimeUnit.SECONDS)
            .writeTimeout(writeTimeoutSeconds, TimeUnit.SECONDS)
            .retryOnConnectionFailure(retryOnConnectionFailure)
            .build();
        LOGGER.info("[{}] OkHttpClient created. maxIdle={}, keepAlive={}s, connect={}s, read={}s, write={}s",
            beanName,
            maxIdleConnections,
            keepAliveSeconds,
            connectTimeoutSeconds,
            readTimeoutSeconds,
            writeTimeoutSeconds);
        return client;
    }
    /**
     * ApplicationContext 종료 시 OkHttpClient 리소스 정리
     * - Dispatcher Thread Pool 종료
     * - ConnectionPool 비움
     * - Cache 사용 시 Cache close
     */
    @Bean
    public DisposableBean okHttpClientDestroyer(
        @Qualifier(PRODUCT_RECOMMENDER_OKHTTP_CLIENT) OkHttpClient productClient,
        @Qualifier(SEARCH_RECOMMENDER_OKHTTP_CLIENT) OkHttpClient searchClient,
        @Qualifier(MAIL_SEND_OKHTTP_CLIENT) OkHttpClient mailClient) {
        return () -> {
            shutdownClient(PRODUCT_RECOMMENDER_OKHTTP_CLIENT, productClient);
            shutdownClient(SEARCH_RECOMMENDER_OKHTTP_CLIENT, searchClient);
            shutdownClient(MAIL_SEND_OKHTTP_CLIENT, mailClient);
        };
    }
    private void shutdownClient(String beanName, OkHttpClient client) {
        LOGGER.debug("[{}] closing. idle={}, connection={}",
            beanName,
            client.connectionPool().idleConnectionCount(),
            client.connectionPool().connectionCount());
        client.dispatcher().executorService().shutdown();
        client.connectionPool().evictAll();
        if (client.cache() != null) {
            try {
                client.cache().close();
            } catch (IOException e) {
                LOGGER.warn("[{}] cache close failed.", beanName, e);
            }
        }
        LOGGER.debug("[{}] closed. idle={}, connection={}",
            beanName,
            client.connectionPool().idleConnectionCount(),
            client.connectionPool().connectionCount());
    }
}
```

OkHttp는 Client를 재사용할 때 connection pool과 thread pool 재사용 효과가 있고, `newBuilder()`로 파생 Client를 만들면 기존 Client의 connection pool과 설정을 공유할 수 있습니다. ([square.github.io](https://square.github.io/okhttp/5.x/okhttp/okhttp3/-ok-http-client/ "OkHttpClient")) 또한 애플리케이션 종료 시 리소스를 적극적으로 해제하려면 `dispatcher().executorService().shutdown()`, `connectionPool().evictAll()`, cache close를 사용할 수 있습니다. ([square.github.io](https://square.github.io/okhttp/5.x/okhttp/okhttp3/-ok-http-client/ "OkHttpClient"))

### 5. 사용 예제

```java
package bk.comm.client;
import java.io.IOException;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;
import bk.comm.config.OkHttpConfig;
import okhttp3.MediaType;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.RequestBody;
import okhttp3.Response;
import okhttp3.ResponseBody;
@Service
public class ProductRecommenderClient {
    private static final MediaType JSON = MediaType.parse("application/json; charset=utf-8");
    private final OkHttpClient okHttpClient;
    public ProductRecommenderClient(
        @Qualifier(OkHttpConfig.PRODUCT_RECOMMENDER_OKHTTP_CLIENT) OkHttpClient okHttpClient) {
        this.okHttpClient = okHttpClient;
    }
    public String postJson(String url, String jsonBody) throws IOException {
        RequestBody body = RequestBody.create(JSON, jsonBody);
        Request request = new Request.Builder()
            .url(url)
            .post(body)
            .build();
        try (Response response = okHttpClient.newCall(request).execute()) {
            if (!response.isSuccessful()) {
                throw new IOException("Product recommender API failed. status=" + response.code());
            }
            ResponseBody responseBody = response.body();
            return responseBody == null ? "" : responseBody.string();
        }
    }
}
```

### 6. 중요한 실무 주의점

|항목|설명|
|---|---|
|`ConnectionPool`은 최대 호출 수 제한이 아님|`maxIdleConnections`는 idle 상태로 보관할 커넥션 수이지, 동시 호출 상한이 아님|
|같은 URL별로 Client를 만들면 안 됨|URL 단위가 아니라 외부 시스템, timeout, 장애격리 기준으로 분리|
|`readTimeout/writeTimeout`은 명시 권장|현재 첨부 소스는 `connectTimeout`만 설정되어 있어 외부 API 응답 지연 시 WAS Thread 점유 시간이 길어질 수 있음|
|메일 API는 추천 API와 분리 권장|메일 API 장애가 상품추천/검색추천 호출 Pool에 영향을 주지 않도록 분리|
|여러 `OkHttpClient` Bean 주입 시 `@Qualifier` 필수|동일 타입 Bean이 2개 이상이면 Spring이 어떤 Bean을 주입할지 판단하지 못할 수 있음|

### 7. 최종 권장안

현재는 **기존 `OkHttpConfig`에 3개 Bean을 추가하고, 공통 생성 메서드와 공통 Destroyer로 정리하는 방식**이 가장 적절합니다. 다만 `productRecommenderOkHttpClient`와 `searchRecommenderOkHttpClient`가 같은 외부 추천 시스템이고 timeout·장애격리 정책도 같다면, 2개로 나누지 말고 `recommenderOkHttpClient` 하나로 합치는 편이 더 좋습니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
