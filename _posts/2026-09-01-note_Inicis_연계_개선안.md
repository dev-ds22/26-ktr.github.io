---
layout: single
title: "Inicis_연계_개선안"
excerpt: "Inicis_연계_개선안"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-09-01"
last_modified_at: "2026-09-01 11:37:18 +0900"
mermaid: false
---
## 1. 개선 결론
현재 코드는 `HttpURLConnection`을 Controller 내부에서 직접 생성하고 있어 다음 문제가 있습니다.

|구분|현재 코드|개선안|
|---|---|---|
|Connection Pool|명시적 Pool 없음|`PoolingHttpClientConnectionManager`|
|Connection 재사용|JVM 내부 Keep-Alive 의존|Apache HttpClient Pool에서 명시적 관리|
|Proxy|`@UseProxy` → JVM System Property 변경|**HttpClient 전용 Proxy 설정**|
|멀티스레드 Proxy 안전성|다른 요청에 영향 가능|해당 HttpClient에만 Proxy 적용|
|자원 반환|`close()`, `disconnect()` 수동 관리|`RestTemplate`/HttpClient가 자동 관리|
|Pool 대기 Timeout|없음|`connectionRequestTimeout` 추가|
|Connect Timeout|3초|3초 유지|
|Read Timeout|9999ms|10초|
|Idle Connection|관리 없음|30초 Idle connection 제거|
|Expired Connection|관리 없음|자동 제거|
|HTTP 오류|200만 처리, 나머지 불명확|4xx/5xx 명시적 처리|
|POST 재시도|명시 없음|자동 Retry 차단|
|SSRF 방어|`containsDomain()` 의존|HTTPS + 정확한 Host Allowlist|
|개인정보 로그|CI/전화번호/성명 출력|**출력 제거**|
|Charset|일부 JVM 기본 Charset|UTF-8 명시|
|JSON 처리|불필요한 `keySet()` 반복|직접 필드 취득|
Spring 5.3의 `HttpComponentsClientHttpRequestFactory`는 미리 구성된 Apache `HttpClient`를 사용할 수 있으며 Connection Pool도 지원하므로 현재 환경에 적합합니다.  Apache의 `PoolingHttpClientConnectionManager`는 route별 Connection을 재사용하며 기본값이 route당 2개/전체 20개에 불과하므로 운영 환경에서는 명시적인 Pool 크기 지정이 권장됩니다. 
## 2. 권장 구조
```text
Browser
   ↓
Controller
   ↓
InicisAuthClient
   ↓
RestTemplate
   ↓
HttpComponentsClientHttpRequestFactory
   ↓
CloseableHttpClient
   ↓
PoolingHttpClientConnectionManager
   ↓
HTTP Proxy
   ↓
https://fcsa.inicis.com/getResultData
```
가장 중요한 변경점은 다음입니다.
```text
기존
@UseProxy
 ↓
System.setProperty("https.proxyHost", ...)
 ↓
HttpURLConnection
```
에서
```text
개선
Inicis 전용 RestTemplate
 ↓
Inicis 전용 HttpClient
 ↓
DefaultProxyRoutePlanner
 ↓
Proxy Server
```
로 변경합니다.
따라서 **아래 구조를 적용한다면 `resultInfoFromInicis()`의 `@UseProxy`는 제거하는 것을 권장합니다.**

- 현재 프로젝트의 `@UseProxy` 방식처럼 요청 전후로 JVM의 `http.proxyHost`, `https.proxyHost` 등을 변경하는 구조는 JBoss의 여러 request thread가 동일한 JVM System Property를 공유하기 때문에 동시 요청 환경에서 안전하지 않습니다.
---
## 3. Maven Dependency
Spring Framework 5.3.x에서는 Apache HttpClient **4.x 계열**을 사용하는 것이 맞습니다. Spring 5.3.39 공식 API도 `HttpComponentsClientHttpRequestFactory`가 Apache HttpComponents 4.3 이상을 요구한다고 명시합니다. 
```xml
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.14</version>
</dependency>
```
이미 프로젝트에 HttpClient가 존재한다면 중복 선언 전 다음을 먼저 확인하는 것이 좋습니다.
```bash
mvn dependency:tree | findstr httpclient
mvn dependency:tree | findstr httpcore
```
권장 조합은 대체로:
```text
httpclient 4.5.14
httpcore   4.4.x
```
- 입니다.
---
## 4. Inicis 전용 Connection Pool + Proxy 설정
현재 사용 중인 Proxy property 이름을 그대로 사용할 수 있도록 작성한 예입니다.
```java
import java.util.concurrent.TimeUnit;

import org.apache.http.HttpHost;
import org.apache.http.client.config.RequestConfig;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.impl.conn.DefaultProxyRoutePlanner;
import org.apache.http.impl.conn.PoolingHttpClientConnectionManager;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.util.StringUtils;
import org.springframework.web.client.RestTemplate;

import egovframework.com.cmm.util.EgovProperties;

/**
 * KG이니시스 외부 HTTP 통신 설정 클래스.
 *
 * <p>
 * Connection Pool, Timeout, Proxy 및 Connection 자원 반환 정책을
 * KG이니시스 통신 전용 HttpClient에 적용한다.
 * </p>
 */
@Configuration
public class InicisHttpClientConfig {

    /** 전체 Connection Pool 최대 개수 */
    private static final int MAX_TOTAL_CONNECTIONS = 50;

    /** 동일 Route 당 최대 Connection 개수 */
    private static final int MAX_CONNECTIONS_PER_ROUTE = 20;

    /** Connection Pool 대기 시간 */
    private static final int CONNECTION_REQUEST_TIMEOUT = 2000;

    /** Connection 연결 제한 시간 */
    private static final int CONNECT_TIMEOUT = 3000;

    /** Response 데이터 수신 제한 시간 */
    private static final int READ_TIMEOUT = 10000;

    /** Connection 최대 생존 시간 */
    private static final int CONNECTION_TIME_TO_LIVE = 300000;

    /** Idle Connection 제거 시간 */
    private static final int IDLE_CONNECTION_TIMEOUT = 30000;

    /**
     * KG이니시스 전용 Connection Pool을 생성한다.
     *
     * @return PoolingHttpClientConnectionManager
     */
    @Bean(name = "inicisConnectionManager", destroyMethod = "close")
    public PoolingHttpClientConnectionManager inicisConnectionManager() {

        PoolingHttpClientConnectionManager connectionManager =
                new PoolingHttpClientConnectionManager(
                        CONNECTION_TIME_TO_LIVE,
                        TimeUnit.MILLISECONDS);

        // 전체 Connection 최대 개수
        connectionManager.setMaxTotal(MAX_TOTAL_CONNECTIONS);

        // 하나의 Route(Proxy -> 이니시스)에 대한 최대 Connection 개수
        connectionManager.setDefaultMaxPerRoute(MAX_CONNECTIONS_PER_ROUTE);

        // 일정시간 사용되지 않은 Connection은 재사용 전에 검증
        connectionManager.setValidateAfterInactivity(5000);

        return connectionManager;
    }

    /**
     * KG이니시스 전용 HttpClient를 생성한다.
     *
     * @param connectionManager Connection Pool Manager
     * @return CloseableHttpClient
     */
    @Bean(name = "inicisHttpClient", destroyMethod = "close")
    public CloseableHttpClient inicisHttpClient(
            @Qualifier("inicisConnectionManager")
            PoolingHttpClientConnectionManager connectionManager) {

        String proxyHost =
                EgovProperties.getProperty("Globals.proxy.host.url");

        String proxyPortValue =
                EgovProperties.getProperty("Globals.proxy.host.port");

        if (!StringUtils.hasText(proxyHost)
                || !StringUtils.hasText(proxyPortValue)) {

            throw new IllegalStateException(
                    "KG이니시스 Proxy 설정이 존재하지 않습니다.");
        }

        final int proxyPort;

        try {
            proxyPort = Integer.parseInt(proxyPortValue);
        } catch (NumberFormatException e) {
            throw new IllegalStateException(
                    "Proxy Port 설정이 올바르지 않습니다.", e);
        }

        /*
         * HTTPS 목적지라도 일반적인 사내 Forward Proxy는
         * HTTP CONNECT 방식이므로 Proxy 자체 scheme은 http로 지정한다.
         */
        HttpHost proxy =
                new HttpHost(proxyHost, proxyPort, "http");

        DefaultProxyRoutePlanner routePlanner =
                new DefaultProxyRoutePlanner(proxy);

        RequestConfig requestConfig =
                RequestConfig.custom()
                        /*
                         * Pool에서 사용할 Connection을 얻기 위해
                         * 대기할 수 있는 최대 시간
                         */
                        .setConnectionRequestTimeout(
                                CONNECTION_REQUEST_TIMEOUT)

                        /*
                         * Proxy 또는 목적지 서버와 TCP Connection을
                         * 생성하기 위한 최대 시간
                         */
                        .setConnectTimeout(CONNECT_TIMEOUT)

                        /*
                         * Connection 생성 이후 Response 데이터를
                         * 기다리는 최대 시간
                         */
                        .setSocketTimeout(READ_TIMEOUT)
                        .build();

        return HttpClients.custom()
                .setConnectionManager(connectionManager)

                /*
                 * ConnectionManager는 Spring Bean에서 별도로
                 * lifecycle 관리한다.
                 */
                .setConnectionManagerShared(true)

                .setDefaultRequestConfig(requestConfig)

                /*
                 * JVM System Property를 사용하지 않고
                 * 이 HttpClient에서만 Proxy를 사용한다.
                 */
                .setRoutePlanner(routePlanner)

                // 만료된 Connection 제거
                .evictExpiredConnections()

                // 30초 이상 미사용 Connection 제거
                .evictIdleConnections(
                        IDLE_CONNECTION_TIMEOUT,
                        TimeUnit.MILLISECONDS)

                /*
                 * 인증 관련 POST 호출이므로 HttpClient 레벨의
                 * 자동 Retry는 수행하지 않는다.
                 */
                .disableAutomaticRetries()

                .build();
    }

    /**
     * KG이니시스 외부 연동용 RestTemplate을 생성한다.
     *
     * @param httpClient HttpClient
     * @return RestTemplate
     */
    @Bean(name = "inicisRestTemplate")
    public RestTemplate inicisRestTemplate(
            @Qualifier("inicisHttpClient")
            CloseableHttpClient httpClient) {

        HttpComponentsClientHttpRequestFactory requestFactory =
                new HttpComponentsClientHttpRequestFactory(httpClient);

        return new RestTemplate(requestFactory);
    }
}
```
Apache HttpClient는 Idle/Expired Connection을 자동 제거하는 API도 공식적으로 제공합니다. 
## 5. KG이니시스 HTTP 호출을 별도 Client로 분리
Controller가 Connection Pool이나 Proxy를 알 필요가 없도록 분리하는 것을 권장합니다.
```java
import java.net.URI;
import java.net.URISyntaxException;
import java.util.Arrays;

import org.json.simple.JSONObject;
import org.json.simple.parser.JSONParser;
import org.json.simple.parser.ParseException;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.client.ResourceAccessException;
import org.springframework.web.client.RestClientResponseException;
import org.springframework.web.client.RestTemplate;

import egovframework.com.cmm.util.EgovProperties;

/**
 * KG이니시스 인증결과 조회 Client.
 *
 */
@Component
public class InicisAuthClient {

    private final RestTemplate inicisRestTemplate;

    /**
     * 생성자.
     *
     * @param inicisRestTemplate KG이니시스 전용 RestTemplate
     */
    public InicisAuthClient(
            @Qualifier("inicisRestTemplate")
            RestTemplate inicisRestTemplate) {

        this.inicisRestTemplate = inicisRestTemplate;
    }

    /**
     * KG이니시스 인증 결과를 조회한다.
     *
     * @param authRequestUrl 인증결과 조회 URL
     * @param mid MID
     * @param txId 거래 ID
     * @return 인증결과 JSON
     * @exception Exception 인증결과 조회 실패
     */
    public JSONObject requestAuthResult(
            String authRequestUrl,
            String mid,
            String txId) throws Exception {

        URI uri = validateAuthRequestUri(authRequestUrl);

        JSONObject requestJson = new JSONObject();
        requestJson.put("mid", mid);
        requestJson.put("txId", txId);

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.setAccept(
                Arrays.asList(MediaType.APPLICATION_JSON));

        HttpEntity<String> requestEntity =
                new HttpEntity<>(
                        requestJson.toJSONString(),
                        headers);

        try {

            /*
             * Response body를 String으로 완전히 읽은 후 반환하므로
             * RestTemplate/HttpClient가 Connection을 Pool에 반환한다.
             */
            ResponseEntity<String> response =
                    inicisRestTemplate.exchange(
                            uri,
                            HttpMethod.POST,
                            requestEntity,
                            String.class);

            if (!response.getStatusCode().is2xxSuccessful()) {
                throw new IllegalStateException(
                        "KG이니시스 인증결과 HTTP 오류. status="
                                + response.getStatusCodeValue());
            }

            String responseBody = response.getBody();

            if (!StringUtils.hasText(responseBody)) {
                throw new IllegalStateException(
                        "KG이니시스 인증결과 Response Body가 없습니다.");
            }

            JSONParser parser = new JSONParser();

            Object parsedResult = parser.parse(responseBody);

            if (!(parsedResult instanceof JSONObject)) {
                throw new IllegalStateException(
                        "KG이니시스 인증결과 JSON 형식이 올바르지 않습니다.");
            }

            return (JSONObject) parsedResult;

        } catch (RestClientResponseException e) {

            /*
             * 4xx / 5xx 응답
             */
            throw new IllegalStateException(
                    "KG이니시스 인증결과 HTTP 통신 오류. status="
                            + e.getRawStatusCode(),
                    e);

        } catch (ResourceAccessException e) {

            /*
             * Connect timeout
             * Read timeout
             * Connection Pool timeout
             */
            throw new IllegalStateException(
                    "KG이니시스 인증결과 Connection 오류.",
                    e);

        } catch (ParseException e) {

            throw new IllegalStateException(
                    "KG이니시스 인증결과 JSON Parsing 오류.",
                    e);
        }
    }

    /**
     * 외부에서 전달된 인증결과 URL의 유효성을 검사한다.
     *
     * @param authRequestUrl 인증결과 URL
     * @return 검증된 URI
     * @exception URISyntaxException URI 형식 오류
     */
    private URI validateAuthRequestUri(
            String authRequestUrl) throws URISyntaxException {

        if (!StringUtils.hasText(authRequestUrl)) {
            throw new IllegalArgumentException(
                    "authRequestUrl이 없습니다.");
        }

        URI uri = new URI(authRequestUrl);

        // 반드시 HTTPS 사용
        if (!"https".equalsIgnoreCase(uri.getScheme())) {
            throw new IllegalArgumentException(
                    "HTTPS URL만 허용됩니다.");
        }

        String host = uri.getHost();

        if (!StringUtils.hasText(host)) {
            throw new IllegalArgumentException(
                    "인증결과 Host가 없습니다.");
        }

        /*
         * 서버에서 관리하는 Allowlist.
         *
         * 예:
         * Globals.seller.auth.allowedHosts=
         *     fcsa.inicis.com,kssa.inicis.com
         */
        String allowedHosts =
                EgovProperties.getProperty(
                        "Globals.seller.auth.allowedHosts");

        if (!StringUtils.hasText(allowedHosts)) {
            throw new IllegalStateException(
                    "KG이니시스 허용 Host가 설정되지 않았습니다.");
        }

        boolean allowed =
                Arrays.stream(allowedHosts.split(","))
                        .map(String::trim)
                        .filter(StringUtils::hasText)
                        .anyMatch(
                                allowedHost ->
                                        host.equalsIgnoreCase(
                                                allowedHost));

        if (!allowed) {
            throw new IllegalArgumentException(
                    "허용되지 않은 KG이니시스 Host입니다.");
        }

        /*
         * user:password@host 형태 차단
         */
        if (uri.getUserInfo() != null) {
            throw new IllegalArgumentException(
                    "UserInfo가 포함된 URL은 허용되지 않습니다.");
        }

        /*
         * HTTPS 표준 Port 외의 임의 Port 차단
         */
        if (uri.getPort() != -1
                && uri.getPort() != 443) {

            throw new IllegalArgumentException(
                    "허용되지 않은 Port입니다.");
        }

        return uri;
    }
}
```
여기서 기존
```java
if(!CommonUtil.containsDomain(
        url.getHost(),
        SELLER_JOIN_KSSA_URL)) {
```
보다 **정확한 hostname 일치 방식**을 권장합니다.
예를 들어 단순 `contains()` 계열 구현이면 다음과 같은 우회 가능성이 생길 수 있습니다.
```text
정상
fcsa.inicis.com
악성 예
fcsa.inicis.com.attacker.com
evil-inicis.com
```
외부에서 전달받는 `authRequestUrl`은 SSRF 관점에서 특히 엄격하게 검증해야 합니다.
더 안전한 방법은 가능하다면 아예 request로 받은 URL을 사용하지 않고:
```properties
Globals.seller.auth.resultUrl=https://fcsa.inicis.com/getResultData
```
- 처럼 **서버 설정값만 사용**하는 것입니다.
---
## 6. Controller 최종 개선 예
기존 로직을 최대한 유지하면서 HTTP 부분과 자원 관리를 제거하면 다음 형태가 됩니다.
```java
@RequestMapping(
        value = "/resultSuccessFromInicis.do",
        method = RequestMethod.POST)
public String resultInfoFromInicis(
        @RequestParam Map<String, String> map,
        ModelMap model,
        HttpServletRequest request,
        ViewBaseVO view) throws Exception {

    String resultCode = map.get("resultCode");
    String resultMsg = map.get("resultMsg");

    String authRequestUrl = map.get("authRequestUrl");
    String txId = map.get("txId");
    String token = map.get("token");

    String userName = "";
    String userCi = "";
    String userPhone = "";

    String encUserName = "";
    String encUserCi = "";

    String mid =
            EgovProperties.getProperty(
                    "Globlas.seller.auth.mid");

    long histSn = 0L;

    /*
     * KG이니시스 1차 인증 실패 시
     * 인증결과 서버를 호출하지 않는다.
     */
    if (!"0000".equals(resultCode)) {

        model.addAttribute("histSn", histSn);
        model.addAttribute("resultCode", resultCode);
        model.addAttribute("resultMsg", resultMsg);
        model.addAttribute("userName", userName);

        return view.tiles(
                "none/mm/mim/authInfoPopup.tiles");
    }

    /*
     * 필수 Parameter 검사
     */
    if (!StringUtils.hasText(authRequestUrl)
            || !StringUtils.hasText(txId)
            || !StringUtils.hasText(token)) {

        resultCode = "8888";
        resultMsg = "인증 필수정보가 없습니다.";

        model.addAttribute("histSn", histSn);
        model.addAttribute("resultCode", resultCode);
        model.addAttribute("resultMsg", resultMsg);
        model.addAttribute("userName", userName);

        return view.tiles(
                "none/mm/mim/authInfoPopup.tiles");
    }

    JSONObject resJson;

    try {

        /*
         * Connection Pool + Proxy가 적용된
         * KG이니시스 전용 Client 사용
         */
        resJson =
                inicisAuthClient.requestAuthResult(
                        authRequestUrl,
                        mid,
                        txId);

    } catch (IllegalArgumentException e) {

        log.error(
                "KG이니시스 인증결과 요청 URL 검증 실패. txId={}",
                txId,
                e);

        resultCode =
                CommonConstants.COMMON_RESULT_FAIL_CODE;
        resultMsg =
                "인증결과 요청정보가 올바르지 않습니다.";

        model.addAttribute("histSn", histSn);
        model.addAttribute("resultCode", resultCode);
        model.addAttribute("resultMsg", resultMsg);
        model.addAttribute("userName", userName);

        return view.tiles(
                "none/mm/mim/authInfoPopup.tiles");

    } catch (Exception e) {

        log.error(
                "KG이니시스 인증결과 조회 실패. txId={}",
                txId,
                e);

        resultCode =
                CommonConstants.COMMON_RESULT_FAIL_CODE;
        resultMsg =
                "인증결과 조회 중 오류가 발생했습니다.";

        model.addAttribute("histSn", histSn);
        model.addAttribute("resultCode", resultCode);
        model.addAttribute("resultMsg", resultMsg);
        model.addAttribute("userName", userName);

        return view.tiles(
                "none/mm/mim/authInfoPopup.tiles");
    }

    /*
     * JSON 결과 취득
     *
     * 기존 코드의 keySet() loop는 필요하지 않다.
     */
    userName =
            resJson.get("userName") == null
                    ? ""
                    : String.valueOf(
                            resJson.get("userName"));

    userCi =
            resJson.get("userCi") == null
                    ? ""
                    : String.valueOf(
                            resJson.get("userCi"));

    userPhone =
            resJson.get("userPhone") == null
                    ? ""
                    : String.valueOf(
                            resJson.get("userPhone"));

    /*
     * 사용자 개인정보는 로그에 출력하지 않는다.
     *
     * 기존:
     * log.debug("userName=" + userName);
     * log.debug("userCi=" + userCi);
     * log.debug("userPhone=" + userPhone);
     *
     * 위 로그는 제거 권장.
     */
    log.debug(
            "KG이니시스 인증결과 수신 완료. txId={}",
            txId);

    try {

        Base64.Decoder decoder =
                Base64.getDecoder();

        byte[] pbUserKey =
                decoder.decode(token);

        byte[] bszIV =
                EgovProperties
                        .getProperty(
                                "Globlas.seller.auth.bszIV")
                        .getBytes(
                                StandardCharsets.UTF_8);

        /*
         * JVM Default Charset을 사용하지 않고
         * UTF-8을 명시한다.
         */
        userName =
                decrypt(
                        userName.getBytes(
                                StandardCharsets.UTF_8),
                        pbUserKey,
                        bszIV,
                        decoder);

        if (StringUtils.hasText(userCi)) {

            userCi =
                    decrypt(
                            userCi.getBytes(
                                    StandardCharsets.UTF_8),
                            pbUserKey,
                            bszIV,
                            decoder);

        } else if ("INIiasTest".equals(mid)) {

            /*
             * 테스트 MID는 CI가 전달되지 않을 수 있다.
             */
            log.debug(
                    "KG이니시스 TEST MID CI 미수신.");
        }

        userPhone =
                decrypt(
                        userPhone.getBytes(
                                StandardCharsets.UTF_8),
                        pbUserKey,
                        bszIV,
                        decoder);

    } catch (IllegalArgumentException
            | IllegalStateException e) {

        log.error(
                "KG이니시스 인증결과 복호화 실패. txId={}",
                txId,
                e);

        resultCode = "8888";
        resultMsg = "Decrypt Error";

        model.addAttribute("histSn", histSn);
        model.addAttribute("resultCode", resultCode);
        model.addAttribute("resultMsg", resultMsg);
        model.addAttribute("userName", "");

        return view.tiles(
                "none/mm/mim/authInfoPopup.tiles");
    }

    /*
     * 인증이력 저장
     */
    try {

        CryptoUtil cryptoUtil =
                new CryptoUtil();

        encUserName =
                cryptoUtil.encrypt(userName);

        encUserCi =
                cryptoUtil.encrypt(userCi);

        SellerAuthHistVO sellerAuthHistVO =
                new SellerAuthHistVO();

        sellerAuthHistVO.setCertcoCd(
                CommonConstants.MM_AUTH_CMPN_CD);

        sellerAuthHistVO.setTxidVal(txId);

        sellerAuthHistVO.setCstmrName(
                encUserName);

        sellerAuthHistVO.setCiVal(
                encUserCi);

        sellerAuthHistVO.setRegtrId(
                CommonConstants.FO_AUTH_USER);

        sellerAuthHistVO.setUpdusrId(
                CommonConstants.FO_AUTH_USER);

        histSn =
                joinSellerService
                        .insertSellerAuthHist(
                                sellerAuthHistVO);

        log.debug(
                "KG이니시스 인증이력 저장 완료. histSn={}",
                histSn);

    } catch (IllegalArgumentException
            | IllegalStateException e) {

        log.error(
                "userName, userCi encrypt FAIL.",
                e);

        resultCode = "8888";
        resultMsg = "Encrypt Error";

    } catch (DataAccessException e) {

        log.error(
                "insertSellerAuthHist Data FAIL.",
                e);

        resultCode =
                CommonConstants.COMMON_RESULT_FAIL_CODE;

        resultMsg =
                "insertSellerAuthHist Data FAIL.";

    } catch (Exception e) {

        log.error(
                "insertSellerAuthHist Exception.",
                e);

        resultCode =
                CommonConstants.COMMON_RESULT_FAIL_CODE;

        resultMsg =
                "insertSellerAuthHist FAIL.";
    }

    if (histSn > 0) {

        resultCode = "0000";

    } else {

        /*
         * 앞에서 구체적인 오류코드가 설정되어 있는 경우
         * 불필요하게 7777로 덮어쓰지 않는 것도 권장한다.
         */
        if ("0000".equals(resultCode)) {

            resultCode = "7777";
            resultMsg =
                    "인증결과 값이 조회되지 않았습니다. "
                    + "잠시 후 다시 시도해 주세요.";
        }
    }

    model.addAttribute(
            "histSn",
            histSn);

    model.addAttribute(
            "resultCode",
            resultCode);

    model.addAttribute(
            "resultMsg",
            resultMsg);

    /*
     * userName을 View에 전달해야 하는지
     * 실제 사용 여부를 확인하는 것이 좋다.
     */
    model.addAttribute(
            "userName",
            userName);

    return view.tiles(
            "none/mm/mim/authInfoPopup.tiles");
}
```
그리고 Controller에 다음 필드를 생성자 주입하면 됩니다.
```java
private final InicisAuthClient inicisAuthClient;

public SellerController(
        InicisAuthClient inicisAuthClient) {

    this.inicisAuthClient = inicisAuthClient;
}
```
- 기존 `joinSellerService`까지 생성자 주입을 사용하고 있다면 같이 묶는 것이 좋습니다.
---
## 7. Properties
기존 Proxy 설정은 그대로 재사용할 수 있습니다.
```properties
## Proxy
Globals.proxy.host.url=10.10.10.10
Globals.proxy.host.port=8080

## KG이니시스 인증결과 호출 허용 Host
Globals.seller.auth.allowedHosts=fcsa.inicis.com,kssa.inicis.com
```
실제 Proxy 주소/Port는 현재 운영값을 사용하면 됩니다.
중요한 차이는 기존에는:
```java
System.setProperty("https.proxyHost", ...);
System.setProperty("https.proxyPort", ...);
```
- 였지만 개선 후에는 **절대로 System Property를 건드리지 않는다는 것**입니다.
---
## 8. `@UseProxy`는 어떻게 해야 하는가
이 메소드에서는 다음과 같이 바꾸는 것을 권장합니다.
### 8-1. 기존
```java
@UseProxy
@RequestMapping(...)
public String resultInfoFromInicis(...) {
```
### 8-2. 변경
```java
@RequestMapping(...)
public String resultInfoFromInicis(...) {
```
이유는 명확합니다.

|방법|Proxy 범위|멀티스레드 안전성|권장|
|---|---|---:|---:|
|`System.setProperty()`|JVM 전체|낮음|❌|
|`@UseProxy` + System Property|JVM 전체|낮음|❌|
|`HttpURLConnection.openConnection(proxy)`|해당 요청|높음|△|
|Apache HttpClient `RoutePlanner`|해당 Client|높음|**◎**|
`@UseProxy`가 JVM 전역 프록시 Property를 설정/해제하는 현재 방식이라면 특히 이런 race condition이 가능합니다.
```text
Thread-A Inicis 호출
   System.setProperty(proxy)
              ↓
Thread-B 다른 외부 API 호출
   → 의도하지 않았는데 Proxy 사용
              ↓
Thread-A finally
   System.clearProperty(proxy)
              ↓
Thread-C Inicis 호출
   → Proxy 설정이 사라진 순간 통신
```
- WAS에서는 Controller 요청들이 동일 JVM 안에서 동시에 동작하므로 이 패턴을 제거하는 편이 맞습니다.
---
## 9. Connection Pool 동작
기존에는:
```text
요청
 → HttpURLConnection 생성
 → Proxy 연결
 → TLS
 → POST
 → 응답
 → disconnect()
```
개선 후:
```text
요청 #1
 → Pool
 → Connection 없음
 → Proxy CONNECT
 → TLS 연결
 → POST
 → Response Body 소비
 → Connection Pool 반환

요청 #2
 → Pool
 → 기존 Connection lease
 → POST
 → Response Body 소비
 → Connection Pool 반환
```
즉 매 요청마다 Connection 객체를 직접 닫는 것이 아니라:
```text
borrow → use → return
```
구조가 됩니다. Apache 문서에서도 Pooling Connection Manager는 기존 persistent connection을 route별로 lease하여 재사용하는 구조임을 명시하고 있습니다. 
## 10. 권장 초기값
|설정|권장 초기값|의미|
|---|---:|---|
|Max Total|50|전체 Connection|
|Max Per Route|20|Inicis 방향 동시 Connection|
|Connection Request Timeout|2초|Pool 고갈 대기시간|
|Connect Timeout|3초|Proxy/TCP 연결|
|Read Timeout|10초|응답 대기|
|Idle Eviction|30초|미사용 Connection 제거|
|Connection TTL|5분|장기 Connection 강제 교체|
특히 `connectionRequestTimeout`이 중요합니다.
예를 들어:
```text
MaxPerRoute = 20
현재 사용 중 = 20
21번째 요청 발생
```
이면 무한정 Pool을 기다리는 것이 아니라:
```text
2초 대기
 ↓
ConnectionPoolTimeoutException 계열
 ↓
ResourceAccessException
```
- 으로 빠져나오게 됩니다.
---
## 11. 현재 소스에서 반드시 같이 수정해야 할 부분
### 11-1. ① `getOutputStream()` 반복 호출
현재:
```java
conn.getOutputStream().write(...);
conn.getOutputStream().flush();
conn.getOutputStream().close();
```
동일 Stream을 세 번 요청하고 있습니다.
최소한 `HttpURLConnection`을 유지한다면:
```java
try (OutputStream os = conn.getOutputStream()) {
    os.write(
        reqJson.toString()
               .getBytes(StandardCharsets.UTF_8));
    os.flush();
}
```
여야 합니다.
하지만 이번 구조에서는 이 코드 자체가 제거됩니다.
### 11-2. ② 불필요한 JSON loop
현재:
```java
for (Object key : resJson.keySet()) {
    userName = ...
    userCi = ...
    userPhone = ...
}
```
`key`를 한 번도 사용하지 않으므로 JSON field 수만큼 같은 코드를 반복합니다.
반드시 제거하는 것이 맞습니다.
### 11-3. ③ 개인정보 로그 제거
현재 가장 위험한 부분 중 하나입니다.
```java
log.debug("==userName==" + userName);
log.debug("==userCi==" + userCi);
log.debug("==userPhone==" + userPhone);
```
그리고 복호화 후:
```java
log.debug("==decrypt userName ==" + userName);
log.debug("==decrypt userCi   ==" + userCi);
log.debug("==decrypt userPhone==" + userPhone);
```
**운영에서는 전부 제거를 권장합니다.**
CI는 특히 개인 식별성이 매우 높은 값입니다.
다음 정도면 충분합니다.
```java
log.debug(
    "KG이니시스 인증결과 정상 수신. txId={}",
    txId);
```
### 11-4. ④ Charset 명시
현재:
```java
token.getBytes();
userName.getBytes();
userCi.getBytes();
userPhone.getBytes();
new String(...).getBytes();
```
는 모두 JVM Default Charset 영향을 받을 수 있습니다.
변경:
```java
userName.getBytes(StandardCharsets.UTF_8)
userCi.getBytes(StandardCharsets.UTF_8)
userPhone.getBytes(StandardCharsets.UTF_8)
```
Base64 token은 더 간단하게:
```java
decoder.decode(token);
```
를 사용하면 됩니다.
### 11-5. ⑤ `mid.equals()` 변경
현재:
```java
if (mid.equals("INIiasTest"))
```
보다:
```java
if ("INIiasTest".equals(mid))
```
가 안전합니다.
### 11-6. ⑥ Parameter 접근 방식 통일
현재 이미:
```java
@RequestParam Map<Object, String> map
```
을 받고 있는데 다시:
```java
request.getParameter("authRequestUrl");
request.getParameter("txId");
request.getParameter("token");
```
을 사용하고 있습니다.
다음처럼 통일하는 것이 낫습니다.
```java
@RequestParam Map<String, String> map
```
```java
String authRequestUrl = map.get("authRequestUrl");
String txId = map.get("txId");
String token = map.get("token");
```
---
## 12. 최종적으로 권장하는 적용 범위
이번 소스에서는 **아래 4개 변경을 한 세트로 적용하는 것이 가장 적절합니다.**
```text
① @UseProxy 제거
        ↓
② InicisHttpClientConfig 추가
   - PoolingHttpClientConnectionManager
   - Proxy
   - Timeout
   - Idle/Expired eviction
        ↓
③ InicisAuthClient 추가
   - HTTP 통신 책임 이동
   - URL Allowlist
   - HTTP/JSON 예외 처리
        ↓
④ Controller 수정
   - HttpURLConnection 전체 제거
   - 인증/복호화/DB 저장 업무만 처리
```
특히 **`RestTemplate`을 요청할 때마다 `new RestTemplate()`하거나 HttpClient를 매번 생성하면 Connection Pool을 적용한 의미가 없어집니다.** `RestTemplate → HttpClient → PoolingHttpClientConnectionManager` 전체를 Spring Singleton Bean으로 유지해야 합니다.
Spring 5.3은 미리 구성된 Apache HttpClient를 `HttpComponentsClientHttpRequestFactory`에 주입하는 방식을 공식 지원합니다.  또한 Apache 측에서도 Connection Manager 종료를 통해 Pool의 시스템 자원을 최종 해제해야 한다고 명시하므로 위 코드처럼 Spring Bean의 `destroyMethod="close"`로 WAS 종료/재배포 시 정리하는 것이 적절합니다. 
**현재 26_KTR 환경에서는 이 구조를 1순위로 권장합니다.** 특히 기존 `@UseProxy`의 JVM 전역 Proxy 변경을 제거하는 것과 `HttpURLConnection`을 Pooling HttpClient로 교체하는 것이 이번 개선의 핵심입니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
