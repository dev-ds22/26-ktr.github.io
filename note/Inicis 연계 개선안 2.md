## 1. 적용 구조

```text
SellerController
        │
        │ HTTP Request / Model 처리만 담당
        ▼
SellerAuthService
        │
        ├─ 인증 업무 Flow 제어
        ├─ 응답 검증
        ├─ 복호화
        └─ 인증이력 저장 요청
        │
        ▼
InicisAuthClient
        │
        │ 외부 연계 전담
        ▼
RestTemplate
        ▼
Apache HttpClient
        ▼
PoolingHttpClientConnectionManager
        ▼
Proxy
        ▼
KG이니시스
```
KG이니시스 현재 통합인증 가이드에서도 결과조회는 `authRequestUrl`을 검증한 뒤 HTTPS Server-to-Server POST로 호출하고, 요청값은 `mid`, `txId`, 권고 Timeout은 5초입니다. 운영 목적지는 `fcsa.inicis.com`, `kssa.inicis.com`, 443/TLS 1.2 이상으로 안내됩니다. 
또한 현재 `isUseToken=Y`인 경우 `userName`, `userPhone`, `userCi`는 SEED 암호화 응답이므로 기존 프로젝트의 SEED 복호화 로직은 임의로 다른 암호 알고리즘으로 변경하면 안 됩니다. 

|파일|역할|
|---|---|
|`InicisHttpClientConfig.java`|Connection Pool/Proxy/Timeout|
|`InicisAuthClient.java`|KG이니시스 HTTP 통신|
|`InicisAuthException.java`|연계 예외 추상화|
|`InicisAuthResponseVO.java`|이니시스 응답|
|`InicisSeedCryptoService.java`|SEED 복호화|
|`SellerAuthResultVO.java`|Controller 반환 결과|
|`SellerAuthService.java`|Application Service|
|기존 Controller|요청/응답만 담당|

## 2. Maven Dependency
현재 프로젝트가 Spring 5.3 계열이므로 Apache HttpClient 4.5.x를 사용합니다. 기존 POM에 이미 존재한다면 중복 추가하지 않습니다.
```xml
<dependency>
	<groupId>org.apache.httpcomponents</groupId>
	<artifactId>httpclient</artifactId>
	<version>4.5.14</version>
</dependency>
```
먼저 확인:
```bash
mvn dependency:tree | findstr httpclient
mvn dependency:tree | findstr httpcore
```
---
## 3. `InicisHttpClientConfig.java`
```java
import java.util.concurrent.TimeUnit;

import org.apache.commons.lang3.StringUtils;
import org.apache.http.HttpHost;
import org.apache.http.client.config.RequestConfig;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.DefaultConnectionKeepAliveStrategy;
import org.apache.http.impl.client.HttpClientBuilder;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.impl.conn.DefaultProxyRoutePlanner;
import org.apache.http.impl.conn.PoolingHttpClientConnectionManager;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.web.client.RestTemplate;

import egovframework.com.cmm.util.EgovProperties;

/**
 * KG이니시스 외부 연계 HTTP Client 설정 클래스.
 *
 * Connection Pool, Proxy, Connection Timeout, Read Timeout 및
 * Connection 재사용 정책을 관리한다.
 *
 * <pre>
 * Controller 또는 Service에서 HttpClient를 직접 생성하지 않고
 * 본 설정에서 생성된 Singleton RestTemplate을 공통으로 사용한다.
 *
 * InicisAuthClient
 *       ↓
 * RestTemplate
 *       ↓
 * CloseableHttpClient
 *       ↓
 * PoolingHttpClientConnectionManager
 *       ↓
 * Proxy
 *       ↓
 * KG이니시스
 * </pre>
 *
 * @author 김원태
 * @since 2026. 9. 1.
 * @version 0.1
 * @see
 *
 * <pre>
 * << 개정이력(Modification Information) >>
 *
 * 수정일         수정자                 수정내용
 * ----------  ------------- --------------------
 * 2026. 9. 1.    김원태       최초 생성
 * </pre>
 */
@Configuration
public class InicisHttpClientConfig {

	private static final int DEFAULT_MAX_TOTAL = 50;
	private static final int DEFAULT_MAX_PER_ROUTE = 20;

	private static final int CONNECTION_REQUEST_TIMEOUT = 2000;
	private static final int CONNECT_TIMEOUT = 3000;
	private static final int READ_TIMEOUT = 5000;

	private static final int VALIDATE_AFTER_INACTIVITY = 5000;
	private static final int IDLE_CONNECTION_TIMEOUT = 30000;
	private static final int CONNECTION_TTL = 300000;
	private static final int DEFAULT_KEEP_ALIVE = 30000;

	private static final String PROXY_USE_PROPERTY = "Globals.seller.auth.proxy.use";
	private static final String PROXY_HOST_PROPERTY = "Globals.proxy.host.url";
	private static final String PROXY_PORT_PROPERTY = "Globals.proxy.host.port";

	/**
	 * KG이니시스 연계용 Connection Pool 생성.
	 *
	 * HttpClient에서 Connection을 매 요청마다 신규 생성하지 않고
	 * Pool에서 대여 후 반환하여 재사용한다.
	 *
	 * @return PoolingHttpClientConnectionManager
	 * @exception
	 *
	 * <pre>
	 * << 개정이력(Modification Information) >>
	 *
	 * 수정일         수정자                 수정내용
	 * ----------  ------------- --------------------
	 * 2026. 9. 1.    김원태       최초 생성
	 * </pre>
	 */
	@Bean(name = "inicisConnectionManager", destroyMethod = "")
	public PoolingHttpClientConnectionManager inicisConnectionManager() {

		PoolingHttpClientConnectionManager connectionManager =
				new PoolingHttpClientConnectionManager(
						CONNECTION_TTL,
						TimeUnit.MILLISECONDS);

		// 전체 Connection Pool 최대 Connection 수
		connectionManager.setMaxTotal(DEFAULT_MAX_TOTAL);

		// 동일 Route에 사용할 수 있는 최대 Connection 수
		connectionManager.setDefaultMaxPerRoute(DEFAULT_MAX_PER_ROUTE);

		// 일정시간 사용하지 않은 Connection은 재사용 전 상태 검증
		connectionManager.setValidateAfterInactivity(VALIDATE_AFTER_INACTIVITY);

		return connectionManager;
	}

	/**
	 * KG이니시스 연계용 HttpClient 생성.
	 *
	 * Proxy 사용 여부는 환경설정 값으로 결정하며,
	 * JVM System Property를 변경하지 않고 본 HttpClient에만 적용한다.
	 *
	 * @param connectionManager Connection Pool Manager
	 * @return CloseableHttpClient
	 * @exception IllegalStateException Proxy 설정 오류
	 *
	 * <pre>
	 * << 개정이력(Modification Information) >>
	 *
	 * 수정일         수정자                 수정내용
	 * ----------  ------------- --------------------
	 * 2026. 9. 1.    김원태       최초 생성
	 * </pre>
	 */
	@Bean(name = "inicisHttpClient", destroyMethod = "close")
	public CloseableHttpClient inicisHttpClient(
			@Qualifier("inicisConnectionManager")
			PoolingHttpClientConnectionManager connectionManager) {

		RequestConfig requestConfig =
				RequestConfig.custom()
						// Connection Pool에서 Connection을 얻기 위한 최대 대기시간
						.setConnectionRequestTimeout(CONNECTION_REQUEST_TIMEOUT)
						// Proxy 또는 목적지 서버와 TCP 연결을 위한 최대 대기시간
						.setConnectTimeout(CONNECT_TIMEOUT)
						// 연결 완료 후 Response를 기다리는 최대시간
						.setSocketTimeout(READ_TIMEOUT)
						.build();

		HttpClientBuilder httpClientBuilder =
				HttpClients.custom()
						.setConnectionManager(connectionManager)
						.setDefaultRequestConfig(requestConfig)

						// 인증결과 POST 요청의 자동 재전송 방지
						.disableAutomaticRetries()

						// 외부 서버 Redirect를 이용한 URL 검증 우회 방지
						.disableRedirectHandling()

						// 해당 연계에서 Cookie를 사용하지 않음
						.disableCookieManagement()

						// 만료 Connection 제거
						.evictExpiredConnections()

						// 장시간 미사용 Connection 제거
						.evictIdleConnections(
								IDLE_CONNECTION_TIMEOUT,
								TimeUnit.MILLISECONDS)

						// 서버가 Keep-Alive 시간을 주지 않을 경우 기본값 사용
						.setKeepAliveStrategy((response, context) -> {

							long keepAlive =
									DefaultConnectionKeepAliveStrategy.INSTANCE
											.getKeepAliveDuration(response, context);

							return keepAlive > 0
									? keepAlive
									: DEFAULT_KEEP_ALIVE;
						});

		String proxyUse =
				EgovProperties.getProperty(PROXY_USE_PROPERTY);

		// 운영/개발환경에서 Proxy를 사용하도록 설정된 경우에만 Proxy 적용
		if ("Y".equalsIgnoreCase(proxyUse)) {

			String proxyHost =
					EgovProperties.getProperty(PROXY_HOST_PROPERTY);

			String proxyPortString =
					EgovProperties.getProperty(PROXY_PORT_PROPERTY);

			if (StringUtils.isBlank(proxyHost)
					|| StringUtils.isBlank(proxyPortString)) {

				throw new IllegalStateException(
						"KG이니시스 Proxy 설정정보가 없습니다.");
			}

			int proxyPort;

			try {
				proxyPort = Integer.parseInt(proxyPortString);
			} catch (NumberFormatException e) {
				throw new IllegalStateException(
						"KG이니시스 Proxy Port 설정이 올바르지 않습니다.",
						e);
			}

			// HTTPS 목적지와의 통신은 HTTP Forward Proxy의 CONNECT를 이용
			HttpHost proxy =
					new HttpHost(proxyHost, proxyPort, "http");

			DefaultProxyRoutePlanner routePlanner =
					new DefaultProxyRoutePlanner(proxy);

			httpClientBuilder.setRoutePlanner(routePlanner);
		}

		return httpClientBuilder.build();
	}

	/**
	 * KG이니시스 연계 전용 RestTemplate 생성.
	 *
	 * @param httpClient KG이니시스 전용 HttpClient
	 * @return RestTemplate
	 * @exception
	 *
	 * <pre>
	 * << 개정이력(Modification Information) >>
	 *
	 * 수정일         수정자                 수정내용
	 * ----------  ------------- --------------------
	 * 2026. 9. 1.    김원태       최초 생성
	 * </pre>
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
### 중요한 변경
기존:
```java
@UseProxy
```
는 **이 이니시스 Controller에서 제거**합니다.
이제 Proxy는:
```text
JVM 전체 Proxy
        X
System.setProperty()
        X
@UseProxy
        X

Inicis 전용 HttpClient
        ↓
DefaultProxyRoutePlanner
        ↓
Proxy
```
로 제한됩니다.
---
## 4. `InicisAuthException.java`
```java
/**
 * KG이니시스 외부 인증 연계 처리 Exception.
 *
 * 외부 통신, URL 검증, Response 검증 및 복호화 과정에서
 * 발생하는 기술 Exception을 Application Service에 전달한다.
 *
 * @author 김원태
 * @since 2026. 9. 1.
 * @version 0.1
 * @see
 *
 * <pre>
 * << 개정이력(Modification Information) >>
 *
 * 수정일         수정자                 수정내용
 * ----------  ------------- --------------------
 * 2026. 9. 1.    김원태       최초 생성
 * </pre>
 */
public class InicisAuthException extends RuntimeException {

	private static final long serialVersionUID = 1L;

	public enum Type {
		INVALID_REQUEST,
		CONNECTION_ERROR,
		HTTP_ERROR,
		INVALID_RESPONSE,
		DECRYPT_ERROR
	}

	private final Type type;

	public InicisAuthException(Type type, String message) {
		super(message);
		this.type = type;
	}

	public InicisAuthException(
			Type type,
			String message,
			Throwable cause) {

		super(message, cause);
		this.type = type;
	}

	public Type getType() {
		return type;
	}
}
```
---
## 5. `InicisAuthResponseVO.java`
```java
/**
 * KG이니시스 통합인증 결과조회 Response VO.
 *
 * @author 김원태
 * @since 2026. 9. 1.
 * @version 0.1
 * @see
 *
 * <pre>
 * << 개정이력(Modification Information) >>
 *
 * 수정일         수정자                 수정내용
 * ----------  ------------- --------------------
 * 2026. 9. 1.    김원태       최초 생성
 * </pre>
 */
public class InicisAuthResponseVO {

	private String resultCode;
	private String resultMsg;
	private String txId;
	private String userName;
	private String userCi;
	private String userPhone;

	public String getResultCode() {
		return resultCode;
	}

	public void setResultCode(String resultCode) {
		this.resultCode = resultCode;
	}

	public String getResultMsg() {
		return resultMsg;
	}

	public void setResultMsg(String resultMsg) {
		this.resultMsg = resultMsg;
	}

	public String getTxId() {
		return txId;
	}

	public void setTxId(String txId) {
		this.txId = txId;
	}

	public String getUserName() {
		return userName;
	}

	public void setUserName(String userName) {
		this.userName = userName;
	}

	public String getUserCi() {
		return userCi;
	}

	public void setUserCi(String userCi) {
		this.userCi = userCi;
	}

	public String getUserPhone() {
		return userPhone;
	}

	public void setUserPhone(String userPhone) {
		this.userPhone = userPhone;
	}
}
```
---
## 6. `InicisAuthClient.java`
외부 연계 자체는 이 클래스에서만 수행합니다.
```java
import java.net.URI;
import java.net.URISyntaxException;
import java.nio.charset.StandardCharsets;
import java.util.Arrays;
import java.util.Collections;

import org.apache.commons.lang3.StringUtils;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.json.simple.JSONObject;
import org.json.simple.parser.JSONParser;
import org.json.simple.parser.ParseException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Component;
import org.springframework.web.client.HttpStatusCodeException;
import org.springframework.web.client.ResourceAccessException;
import org.springframework.web.client.RestClientException;
import org.springframework.web.client.RestTemplate;

import bk.app.mm.exception.InicisAuthException;
import bk.app.mm.exception.InicisAuthException.Type;
import bk.app.mm.vo.InicisAuthResponseVO;
import egovframework.com.cmm.util.EgovProperties;

/**
 * KG이니시스 통합인증 결과조회 외부 연계 Client.
 *
 * 외부 URL 검증, HTTP Request/Response 및 JSON 변환을 담당한다.
 * Controller 또는 Application Service에서는 HTTP Connection을
 * 직접 생성하거나 관리하지 않는다.
 *
 * @author 김원태
 * @since 2026. 9. 1.
 * @version 0.1
 * @see
 *
 * <pre>
 * << 개정이력(Modification Information) >>
 *
 * 수정일         수정자                 수정내용
 * ----------  ------------- --------------------
 * 2026. 9. 1.    김원태       최초 생성
 * </pre>
 */
@Component("inicisAuthClient")
public class InicisAuthClient {

	private static final Logger LOGGER =
			LogManager.getLogger(InicisAuthClient.class);

	private static final String ALLOWED_HOST_PROPERTY =
			"Globals.seller.auth.allowedHosts";

	private RestTemplate inicisRestTemplate;

	/**
	 * KG이니시스 RestTemplate 설정.
	 *
	 * @param inicisRestTemplate KG이니시스 전용 RestTemplate
	 * @return void
	 * @exception
	 */
	@Autowired
	public void setInicisRestTemplate(
			@Qualifier("inicisRestTemplate")
			RestTemplate inicisRestTemplate) {

		this.inicisRestTemplate = inicisRestTemplate;
	}

	/**
	 * KG이니시스 통합인증 결과 조회.
	 *
	 * <pre>
	 * 처리순서
	 *
	 * 1. 외부에서 전달받은 authRequestUrl 검증
	 * 2. mid, txId JSON Request 생성
	 * 3. Connection Pool에서 Connection 획득
	 * 4. Proxy를 통한 KG이니시스 HTTPS POST 호출
	 * 5. Response Body 전체 수신
	 * 6. Connection을 Pool에 반환
	 * 7. JSON Response를 VO로 변환
	 * </pre>
	 *
	 * @param authRequestUrl KG이니시스 인증결과 조회 URL
	 * @param mid 상점 MID
	 * @param txId 인증 Transaction ID
	 * @return InicisAuthResponseVO
	 * @exception InicisAuthException 외부 연계 오류
	 *
	 * <pre>
	 * << 개정이력(Modification Information) >>
	 *
	 * 수정일         수정자                 수정내용
	 * ----------  ------------- --------------------
	 * 2026. 9. 1.    김원태       최초 생성
	 * </pre>
	 */
	public InicisAuthResponseVO requestAuthResult(
			String authRequestUrl,
			String mid,
			String txId) {

		URI requestUri = validateAuthRequestUrl(authRequestUrl);

		JSONObject requestJson = new JSONObject();
		requestJson.put("mid", mid);
		requestJson.put("txId", txId);

		HttpHeaders headers = new HttpHeaders();

		headers.setContentType(
				new MediaType(
						MediaType.APPLICATION_JSON,
						StandardCharsets.UTF_8));

		headers.setAccept(
				Collections.singletonList(
						MediaType.APPLICATION_JSON));

		HttpEntity<String> requestEntity =
				new HttpEntity<>(
						requestJson.toJSONString(),
						headers);

		try {

			ResponseEntity<String> response =
					inicisRestTemplate.exchange(
							requestUri,
							HttpMethod.POST,
							requestEntity,
							String.class);

			String responseBody = response.getBody();

			if (StringUtils.isBlank(responseBody)) {
				throw new InicisAuthException(
						Type.INVALID_RESPONSE,
						"KG이니시스 Response Body가 없습니다.");
			}

			return convertResponse(responseBody);

		} catch (HttpStatusCodeException e) {

			LOGGER.error(
					"KG이니시스 HTTP 오류. status={}",
					e.getRawStatusCode());

			throw new InicisAuthException(
					Type.HTTP_ERROR,
					"KG이니시스 HTTP 응답 오류.",
					e);

		} catch (ResourceAccessException e) {

			/*
			 * 다음 오류가 이 경로로 전달될 수 있다.
			 *
			 * - Connection Pool 획득 Timeout
			 * - Proxy/TCP Connect Timeout
			 * - Socket Read Timeout
			 */
			LOGGER.error(
					"KG이니시스 Connection 오류.",
					e);

			throw new InicisAuthException(
					Type.CONNECTION_ERROR,
					"KG이니시스 Connection 오류.",
					e);

		} catch (InicisAuthException e) {
			throw e;

		} catch (RestClientException e) {

			LOGGER.error(
					"KG이니시스 HTTP Client 오류.",
					e);

			throw new InicisAuthException(
					Type.CONNECTION_ERROR,
					"KG이니시스 HTTP Client 오류.",
					e);
		}
	}

	/**
	 * KG이니시스 인증결과 조회 URL 유효성 검증.
	 *
	 * 외부 Request Parameter로 전달되는 URL을 그대로 호출할 경우
	 * SSRF 취약점이 발생할 수 있으므로 허용된 Domain만 호출한다.
	 *
	 * @param authRequestUrl 인증결과 조회 URL
	 * @return 검증 완료 URI
	 * @exception InicisAuthException URL 검증 실패
	 */
	private URI validateAuthRequestUrl(String authRequestUrl) {

		if (StringUtils.isBlank(authRequestUrl)) {
			throw new InicisAuthException(
					Type.INVALID_REQUEST,
					"authRequestUrl이 없습니다.");
		}

		try {

			URI uri = new URI(authRequestUrl);

			// KG이니시스 결과조회는 HTTPS만 허용
			if (!"https".equalsIgnoreCase(uri.getScheme())) {
				throw new InicisAuthException(
						Type.INVALID_REQUEST,
						"HTTPS URL만 허용됩니다.");
			}

			String host = uri.getHost();

			if (StringUtils.isBlank(host)) {
				throw new InicisAuthException(
						Type.INVALID_REQUEST,
						"Host 정보가 없습니다.");
			}

			// URL내 user:password 형식 차단
			if (uri.getUserInfo() != null) {
				throw new InicisAuthException(
						Type.INVALID_REQUEST,
						"UserInfo가 포함된 URL은 허용되지 않습니다.");
			}

			// 443 이외의 임의 Port 사용 차단
			if (uri.getPort() != -1
					&& uri.getPort() != 443) {

				throw new InicisAuthException(
						Type.INVALID_REQUEST,
						"허용되지 않은 Port입니다.");
			}

			String allowedHosts =
					EgovProperties.getProperty(
							ALLOWED_HOST_PROPERTY);

			// Allowlist가 없으면 보안을 위해 통신 자체를 차단
			if (StringUtils.isBlank(allowedHosts)) {
				throw new InicisAuthException(
						Type.INVALID_REQUEST,
						"KG이니시스 허용 Domain 설정이 없습니다.");
			}

			boolean allowed =
					Arrays.stream(allowedHosts.split(","))
							.map(String::trim)
							.filter(StringUtils::isNotBlank)
							.anyMatch(
									allowedHost ->
											host.equalsIgnoreCase(
													allowedHost));

			if (!allowed) {
				throw new InicisAuthException(
						Type.INVALID_REQUEST,
						"허용되지 않은 인증결과 URL입니다.");
			}

			return uri;

		} catch (URISyntaxException e) {

			throw new InicisAuthException(
					Type.INVALID_REQUEST,
					"인증결과 URL 형식이 올바르지 않습니다.",
					e);
		}
	}

	/**
	 * JSON Response를 InicisAuthResponseVO로 변환.
	 *
	 * @param responseBody KG이니시스 Response Body
	 * @return InicisAuthResponseVO
	 * @exception InicisAuthException JSON Parsing 오류
	 */
	private InicisAuthResponseVO convertResponse(
			String responseBody) {

		try {

			JSONParser parser = new JSONParser();

			Object parsedObject =
					parser.parse(responseBody);

			if (!(parsedObject instanceof JSONObject)) {
				throw new InicisAuthException(
						Type.INVALID_RESPONSE,
						"KG이니시스 Response 형식이 올바르지 않습니다.");
			}

			JSONObject responseJson =
					(JSONObject) parsedObject;

			InicisAuthResponseVO responseVO =
					new InicisAuthResponseVO();

			responseVO.setResultCode(
					getString(responseJson, "resultCode"));

			responseVO.setResultMsg(
					getString(responseJson, "resultMsg"));

			responseVO.setTxId(
					getString(responseJson, "txId"));

			responseVO.setUserName(
					getString(responseJson, "userName"));

			responseVO.setUserCi(
					getString(responseJson, "userCi"));

			responseVO.setUserPhone(
					getString(responseJson, "userPhone"));

			return responseVO;

		} catch (ParseException e) {

			throw new InicisAuthException(
					Type.INVALID_RESPONSE,
					"KG이니시스 Response JSON Parsing 오류.",
					e);
		}
	}

	/**
	 * JSONObject String 값 반환.
	 *
	 * @param jsonObject JSON 객체
	 * @param key Key
	 * @return String
	 * @exception
	 */
	private String getString(
			JSONObject jsonObject,
			String key) {

		Object value = jsonObject.get(key);

		return value == null
				? ""
				: String.valueOf(value);
	}
}
```
기존 코드의:
```java
for (Object key : resJson.keySet()) {
	userName = ...
	userCi = ...
	userPhone = ...
}
```
는 완전히 제거합니다.
---
## 7. `SellerAuthUserVO.java`
```java

/**
 * KG이니시스 인증결과 복호화 사용자 정보 VO.
 *
 * @author 김원태
 * @since 2026. 9. 1.
 * @version 0.1
 * @see
 *
 * <pre>
 * << 개정이력(Modification Information) >>
 *
 * 수정일         수정자                 수정내용
 * ----------  ------------- --------------------
 * 2026. 9. 1.    김원태       최초 생성
 * </pre>
 */
public class SellerAuthUserVO {

	private String userName;
	private String userCi;
	private String userPhone;

	public String getUserName() {
		return userName;
	}

	public void setUserName(String userName) {
		this.userName = userName;
	}

	public String getUserCi() {
		return userCi;
	}

	public void setUserCi(String userCi) {
		this.userCi = userCi;
	}

	public String getUserPhone() {
		return userPhone;
	}

	public void setUserPhone(String userPhone) {
		this.userPhone = userPhone;
	}
}
```
---
## 8. `InicisSeedCryptoService.java`
현재 이니시스 공식 문서도 token 사용 시 개인정보 필드가 SEED 암호화됨을 명시합니다. 
현재 Controller의 `decrypt(buffersNm, pbUserKey, bszIV, decoder)` 형식과 일반적인 KISA SEED CBC 복호화 방식도 일치합니다. 다만 **`KISA_SEED_CBC`의 실제 package는 현재 질문에 포함되지 않았으므로 기존 Controller의 import를 그대로 사용해야 합니다.**
```java
import java.nio.charset.StandardCharsets;
import java.util.Base64;
import java.util.Base64.Decoder;

import org.apache.commons.lang3.StringUtils;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.springframework.stereotype.Service;

import bk.app.mm.exception.InicisAuthException;
import bk.app.mm.exception.InicisAuthException.Type;
import bk.app.mm.vo.InicisAuthResponseVO;
import bk.app.mm.vo.SellerAuthUserVO;
import egovframework.com.cmm.util.EgovProperties;

/*
 * 기존 Controller의 decrypt()에서 사용 중인
 * KISA_SEED_CBC 클래스의 실제 import를 그대로 추가한다.
 *
 * 예)
 * import xxx.xxx.KISA_SEED_CBC;
 */

/**
 * KG이니시스 통합인증 결과 SEED 복호화 서비스.
 *
 * token을 Base64 Decode하여 SEED Key를 구하고,
 * 설정된 IV와 함께 사용자 개인정보를 복호화한다.
 *
 * @author 김원태
 * @since 2026. 9. 1.
 * @version 0.1
 * @see
 *
 * <pre>
 * << 개정이력(Modification Information) >>
 *
 * 수정일         수정자                 수정내용
 * ----------  ------------- --------------------
 * 2026. 9. 1.    김원태       최초 생성
 * </pre>
 */
@Service("inicisSeedCryptoService")
public class InicisSeedCryptoService {

	private static final Logger LOGGER =
			LogManager.getLogger(InicisSeedCryptoService.class);

	private static final String IV_PROPERTY =
			"Globlas.seller.auth.bszIV";

	private static final int SEED_KEY_LENGTH = 16;
	private static final int SEED_IV_LENGTH = 16;

	/**
	 * 이니시스 인증결과 사용자 개인정보 복호화.
	 *
	 * @param responseVO 이니시스 결과조회 Response
	 * @param token 암호화 Token
	 * @return SellerAuthUserVO
	 * @exception InicisAuthException 복호화 실패
	 *
	 * <pre>
	 * << 개정이력(Modification Information) >>
	 *
	 * 수정일         수정자                 수정내용
	 * ----------  ------------- --------------------
	 * 2026. 9. 1.    김원태       최초 생성
	 * </pre>
	 */
	public SellerAuthUserVO decryptUserInfo(
			InicisAuthResponseVO responseVO,
			String token) {

		if (StringUtils.isBlank(token)) {
			throw new InicisAuthException(
					Type.DECRYPT_ERROR,
					"인증 Token이 없습니다.");
		}

		try {

			Decoder decoder = Base64.getDecoder();

			// token은 Base64 Encoding 되어 전달됨
			byte[] userKey =
					decoder.decode(token);

			String ivString =
					EgovProperties.getProperty(
							IV_PROPERTY);

			if (StringUtils.isBlank(ivString)) {
				throw new InicisAuthException(
						Type.DECRYPT_ERROR,
						"SEED IV 설정정보가 없습니다.");
			}

			byte[] iv =
					ivString.getBytes(
							StandardCharsets.UTF_8);

			// SEED Key 및 IV 길이 검증
			if (userKey.length != SEED_KEY_LENGTH) {
				throw new InicisAuthException(
						Type.DECRYPT_ERROR,
						"SEED Key 길이가 올바르지 않습니다.");
			}

			if (iv.length != SEED_IV_LENGTH) {
				throw new InicisAuthException(
						Type.DECRYPT_ERROR,
						"SEED IV 길이가 올바르지 않습니다.");
			}

			SellerAuthUserVO userVO =
					new SellerAuthUserVO();

			userVO.setUserName(
					decrypt(
							responseVO.getUserName(),
							userKey,
							iv,
							decoder));

			// 테스트 계정 등 CI값이 제공되지 않는 경우에는 복호화하지 않음
			if (StringUtils.isNotBlank(
					responseVO.getUserCi())) {

				userVO.setUserCi(
						decrypt(
								responseVO.getUserCi(),
								userKey,
								iv,
								decoder));

			} else {
				userVO.setUserCi("");
			}

			userVO.setUserPhone(
					decrypt(
							responseVO.getUserPhone(),
							userKey,
							iv,
							decoder));

			return userVO;

		} catch (InicisAuthException e) {
			throw e;

		} catch (IllegalArgumentException e) {

			LOGGER.error(
					"KG이니시스 SEED Base64 Decode 실패.",
					e);

			throw new InicisAuthException(
					Type.DECRYPT_ERROR,
					"KG이니시스 인증결과 복호화 실패.",
					e);

		} catch (Exception e) {

			LOGGER.error(
					"KG이니시스 SEED 복호화 실패.",
					e);

			throw new InicisAuthException(
					Type.DECRYPT_ERROR,
					"KG이니시스 인증결과 복호화 실패.",
					e);
		}
	}

	/**
	 * SEED CBC 암호문 복호화.
	 *
	 * 기존 Controller에 존재하는 decrypt()와 동일한 처리이며,
	 * Connection 개선 작업에서는 암호 알고리즘 자체를 변경하지 않는다.
	 *
	 * @param encryptedValue Base64 Encoding된 SEED 암호문
	 * @param userKey SEED Key
	 * @param iv SEED IV
	 * @param decoder Base64 Decoder
	 * @return 복호화 문자열
	 * @exception Exception SEED 복호화 오류
	 */
	private String decrypt(
			String encryptedValue,
			byte[] userKey,
			byte[] iv,
			Decoder decoder) throws Exception {

		if (StringUtils.isBlank(encryptedValue)) {
			return "";
		}

		byte[] encryptedData =
				decoder.decode(
						encryptedValue.getBytes(
								StandardCharsets.UTF_8));

		/*
		 * 기존 프로젝트에서 사용하는
		 * KISA_SEED_CBC.SEED_CBC_Decrypt()를 그대로 사용한다.
		 */
		byte[] decryptedData =
				KISA_SEED_CBC.SEED_CBC_Decrypt(
						userKey,
						iv,
						encryptedData,
						0,
						encryptedData.length);

		return new String(
				decryptedData,
				StandardCharsets.UTF_8);
	}
}
```
- **여기서 유일하게 프로젝트 실제 소스에 맞춰야 하는 부분은 `KISA_SEED_CBC` import입니다.** 기존 `decrypt()`가 다른 SEED 구현 클래스를 사용하고 있다면 위 private `decrypt()`의 본문만 기존 소스를 그대로 옮기는 것이 안전합니다.
---
## 9. `SellerAuthResultVO.java`
```java
/**
 * Seller 인증 처리 결과 VO.
 *
 * Controller에 인증 업무 내부 구현을 노출하지 않고
 * 화면 출력에 필요한 값만 전달한다.
 *
 * @author 김원태
 * @since 2026. 9. 1.
 * @version 0.1
 * @see
 *
 * <pre>
 * << 개정이력(Modification Information) >>
 *
 * 수정일         수정자                 수정내용
 * ----------  ------------- --------------------
 * 2026. 9. 1.    김원태       최초 생성
 * </pre>
 */
public class SellerAuthResultVO {

	private String resultCode;
	private String resultMsg;
	private String userName;
	private long histSn;

	public static SellerAuthResultVO of(
			String resultCode,
			String resultMsg,
			String userName,
			long histSn) {

		SellerAuthResultVO resultVO =
				new SellerAuthResultVO();

		resultVO.setResultCode(resultCode);
		resultVO.setResultMsg(resultMsg);
		resultVO.setUserName(userName);
		resultVO.setHistSn(histSn);

		return resultVO;
	}

	public String getResultCode() {
		return resultCode;
	}

	public void setResultCode(String resultCode) {
		this.resultCode = resultCode;
	}

	public String getResultMsg() {
		return resultMsg;
	}

	public void setResultMsg(String resultMsg) {
		this.resultMsg = resultMsg;
	}

	public String getUserName() {
		return userName;
	}

	public void setUserName(String userName) {
		this.userName = userName;
	}

	public long getHistSn() {
		return histSn;
	}

	public void setHistSn(long histSn) {
		this.histSn = histSn;
	}
}
```
---
## 10. 핵심 `SellerAuthService.java`
이 클래스가 **Controller와 외부 Integration 계층 사이의 Application Service**입니다.
중요한 점은 클래스에 `@Transactional`을 붙이지 않는 것입니다.
```java
import java.util.Map;

import org.apache.commons.lang3.StringUtils;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.egovframe.rte.fdl.cmmn.EgovAbstractServiceImpl;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.dao.DataAccessException;
import org.springframework.stereotype.Service;

import bk.app.comm.constants.CommonConstants;
import bk.app.mm.client.InicisAuthClient;
import bk.app.mm.exception.InicisAuthException;
import bk.app.mm.vo.InicisAuthResponseVO;
import bk.app.mm.vo.SellerAuthResultVO;
import bk.app.mm.vo.SellerAuthUserVO;
import egovframework.com.cmm.util.EgovProperties;

/**
 * Seller KG이니시스 통합인증 처리 서비스.
 *
 * Controller와 외부 KG이니시스 연계 Client 사이에서
 * 인증 업무 Flow를 제어한다.
 *
 * <pre>
 * 처리순서
 *
 * Controller
 *      ↓
 * SellerAuthService
 *      ↓
 * InicisAuthClient
 *      ↓
 * KG이니시스 인증결과 조회
 *      ↓
 * 결과코드 및 txId 검증
 *      ↓
 * SEED 복호화
 *      ↓
 * DB 저장용 개인정보 암호화
 *      ↓
 * 인증이력 저장
 * </pre>
 *
 * 외부 HTTP 통신시간 동안 DB Transaction을 유지하지 않도록
 * 본 Service 전체에는 @Transactional을 설정하지 않는다.
 *
 * @author 김원태
 * @since 2026. 9. 1.
 * @version 0.1
 * @see
 *
 * <pre>
 * << 개정이력(Modification Information) >>
 *
 * 수정일         수정자                 수정내용
 * ----------  ------------- --------------------
 * 2026. 9. 1.    김원태       Connection Pool 및 연계 구조 개선
 * </pre>
 */
@Service("sellerAuthService")
public class SellerAuthService extends EgovAbstractServiceImpl {

	private static final Logger LOGGER =
			LogManager.getLogger(SellerAuthService.class);

	private static final String SUCCESS_CODE = "0000";
	private static final String NO_RESULT_CODE = "7777";
	private static final String AUTH_ERROR_CODE = "8888";

	private static final String MID_PROPERTY =
			"Globlas.seller.auth.mid";

	private InicisAuthClient inicisAuthClient;

	private InicisSeedCryptoService inicisSeedCryptoService;

	/*
	 * 현재 프로젝트에 존재하는 기존 Service 사용.
	 * 실제 package/import는 기존 Controller의 import를 그대로 사용한다.
	 */
	private JoinSellerService joinSellerService;

	/**
	 * KG이니시스 연계 Client 설정.
	 *
	 * @param inicisAuthClient KG이니시스 연계 Client
	 * @return void
	 * @exception
	 */
	@Autowired
	public void setInicisAuthClient(
			InicisAuthClient inicisAuthClient) {

		this.inicisAuthClient = inicisAuthClient;
	}

	/**
	 * KG이니시스 SEED 복호화 Service 설정.
	 *
	 * @param inicisSeedCryptoService SEED 복호화 Service
	 * @return void
	 * @exception
	 */
	@Autowired
	public void setInicisSeedCryptoService(
			InicisSeedCryptoService inicisSeedCryptoService) {

		this.inicisSeedCryptoService =
				inicisSeedCryptoService;
	}

	/**
	 * Seller 인증이력 Service 설정.
	 *
	 * @param joinSellerService Seller Service
	 * @return void
	 * @exception
	 */
	@Autowired
	public void setJoinSellerService(
			JoinSellerService joinSellerService) {

		this.joinSellerService =
				joinSellerService;
	}

	/**
	 * KG이니시스 인증결과 처리.
	 *
	 * @param paramMap KG이니시스 Callback Parameter
	 * @return SellerAuthResultVO 인증 처리결과
	 * @exception
	 *
	 * <pre>
	 * << 개정이력(Modification Information) >>
	 *
	 * 수정일         수정자                 수정내용
	 * ----------  ------------- --------------------
	 * 2026. 9. 1.    김원태       Connection Pool 및 연계 구조 개선
	 * </pre>
	 */
	public SellerAuthResultVO processInicisAuth(
			Map<String, String> paramMap) {

		String resultCode =
				paramMap.get("resultCode");

		String resultMsg =
				paramMap.get("resultMsg");

		String authRequestUrl =
				paramMap.get("authRequestUrl");

		String txId =
				paramMap.get("txId");

		String token =
				paramMap.get("token");

		String mid =
				EgovProperties.getProperty(
						MID_PROPERTY);

		long histSn = 0L;

		// KG이니시스 1차 인증 실패 시 결과조회 API를 호출하지 않음
		if (!SUCCESS_CODE.equals(resultCode)) {

			return SellerAuthResultVO.of(
					resultCode,
					resultMsg,
					"",
					histSn);
		}

		// 인증결과 조회에 필요한 필수정보 확인
		if (StringUtils.isBlank(authRequestUrl)
				|| StringUtils.isBlank(txId)) {

			return SellerAuthResultVO.of(
					AUTH_ERROR_CODE,
					"인증결과 요청정보가 없습니다.",
					"",
					histSn);
		}

		// Token 사용방식에서는 복호화를 위해 Token이 반드시 필요
		if (StringUtils.isBlank(token)) {

			return SellerAuthResultVO.of(
					AUTH_ERROR_CODE,
					"인증 고유번호가 없습니다.",
					"",
					histSn);
		}

		try {

			/*
			 * 외부 HTTP 통신.
			 *
			 * 이 시점에는 DB Transaction을 시작하지 않는다.
			 */
			InicisAuthResponseVO inicisResponse =
					inicisAuthClient.requestAuthResult(
							authRequestUrl,
							mid,
							txId);

			// 결과조회 API 자체의 처리결과 검증
			if (!SUCCESS_CODE.equals(
					inicisResponse.getResultCode())) {

				LOGGER.warn(
						"KG이니시스 결과조회 실패. txId={}",
						maskTxId(txId));

				return SellerAuthResultVO.of(
						inicisResponse.getResultCode(),
						inicisResponse.getResultMsg(),
						"",
						histSn);
			}

			/*
			 * 요청 txId와 결과조회 Response txId가 동일한지 검증한다.
			 *
			 * 다른 Transaction의 결과가 처리되는 것을 방지한다.
			 */
			if (StringUtils.isBlank(
					inicisResponse.getTxId())
					|| !txId.equals(
							inicisResponse.getTxId())) {

				LOGGER.error(
						"KG이니시스 txId 불일치. requestTxId={}",
						maskTxId(txId));

				return SellerAuthResultVO.of(
						AUTH_ERROR_CODE,
						"인증결과 검증에 실패했습니다.",
						"",
						histSn);
			}

			// 암호화된 사용자 개인정보 SEED 복호화
			SellerAuthUserVO userVO =
					inicisSeedCryptoService
							.decryptUserInfo(
									inicisResponse,
									token);

			/*
			 * 이름/전화번호/CI는 개인정보이므로
			 * DEBUG/INFO 로그에 평문을 출력하지 않는다.
			 */
			LOGGER.debug(
					"KG이니시스 인증정보 복호화 완료. txId={}",
					maskTxId(txId));

			// 인증이력 저장
			histSn =
					insertSellerAuthHist(
							txId,
							userVO);

			if (histSn <= 0) {

				return SellerAuthResultVO.of(
						NO_RESULT_CODE,
						"인증결과 값이 조회되지 않았습니다. 잠시 후 다시 시도해 주세요.",
						"",
						histSn);
			}

			LOGGER.debug(
					"KG이니시스 인증처리 완료. txId={}, histSn={}",
					maskTxId(txId),
					histSn);

			return SellerAuthResultVO.of(
					SUCCESS_CODE,
					resultMsg,
					userVO.getUserName(),
					histSn);

		} catch (InicisAuthException e) {

			LOGGER.error(
					"KG이니시스 인증처리 실패. type={}, txId={}",
					e.getType(),
					maskTxId(txId),
					e);

			return SellerAuthResultVO.of(
					AUTH_ERROR_CODE,
					getExternalErrorMessage(e),
					"",
					histSn);

		} catch (DataAccessException e) {

			LOGGER.error(
					"insertSellerAuthHist Data FAIL. txId={}",
					maskTxId(txId),
					e);

			return SellerAuthResultVO.of(
					CommonConstants.COMMON_RESULT_FAIL_CODE,
					"insertSellerAuthHist Data FAIL.",
					"",
					histSn);

		} catch (IllegalArgumentException
				| IllegalStateException e) {

			LOGGER.error(
					"Seller 인증정보 암호화 실패. txId={}",
					maskTxId(txId),
					e);

			return SellerAuthResultVO.of(
					AUTH_ERROR_CODE,
					"Encrypt Error",
					"",
					histSn);

		} catch (Exception e) {

			LOGGER.error(
					"Seller 인증처리 Exception. txId={}",
					maskTxId(txId),
					e);

			return SellerAuthResultVO.of(
					CommonConstants.COMMON_RESULT_FAIL_CODE,
					"인증처리 중 오류가 발생했습니다.",
					"",
					histSn);
		}
	}

	/**
	 * Seller 인증이력 저장.
	 *
	 * 평문 개인정보를 DB 저장용 CryptoUtil로 암호화한 후
	 * 기존 joinSellerService를 이용하여 인증이력을 저장한다.
	 *
	 * @param txId 인증 Transaction ID
	 * @param userVO 복호화 사용자정보
	 * @return 인증이력 순번
	 * @exception Exception 인증이력 저장 오류
	 */
	private long insertSellerAuthHist(
			String txId,
			SellerAuthUserVO userVO)
			throws Exception {

		CryptoUtil cryptoUtil =
				new CryptoUtil();

		String encUserName =
				cryptoUtil.encrypt(
						userVO.getUserName());

		String encUserCi =
				cryptoUtil.encrypt(
						userVO.getUserCi());

		SellerAuthHistVO sellerAuthHistVO =
				new SellerAuthHistVO();

		// 인증업체 : KG이니시스
		sellerAuthHistVO.setCertcoCd(
				CommonConstants.MM_AUTH_CMPN_CD);

		// 인증 Transaction ID
		sellerAuthHistVO.setTxidVal(txId);

		// 사용자 이름 암호화 저장
		sellerAuthHistVO.setCstmrName(
				encUserName);

		// CI 암호화 저장
		sellerAuthHistVO.setCiVal(
				encUserCi);

		sellerAuthHistVO.setRegtrId(
				CommonConstants.FO_AUTH_USER);

		sellerAuthHistVO.setUpdusrId(
				CommonConstants.FO_AUTH_USER);

		/*
		 * joinSellerService.insertSellerAuthHist()에
		 * 기존 Transaction 설정이 존재하면 그대로 사용한다.
		 *
		 * 외부 HTTP 통신 완료 후 DB Transaction이 시작되는 구조이다.
		 */
		return joinSellerService
				.insertSellerAuthHist(
						sellerAuthHistVO);
	}

	/**
	 * 외부연계 Exception 사용자 메시지 변환.
	 *
	 * 내부 Proxy 주소, URL, Connection 상태 등의
	 * 상세 기술정보를 사용자 화면에 노출하지 않는다.
	 *
	 * @param e KG이니시스 연계 Exception
	 * @return 사용자 메시지
	 * @exception
	 */
	private String getExternalErrorMessage(
			InicisAuthException e) {

		switch (e.getType()) {

			case INVALID_REQUEST:
				return "인증결과 요청정보가 올바르지 않습니다.";

			case CONNECTION_ERROR:
			case HTTP_ERROR:
				return "인증기관 통신 중 오류가 발생했습니다. 잠시 후 다시 시도해 주세요.";

			case INVALID_RESPONSE:
				return "인증결과 검증에 실패했습니다.";

			case DECRYPT_ERROR:
				return "인증결과 처리 중 오류가 발생했습니다.";

			default:
				return "인증처리 중 오류가 발생했습니다.";
		}
	}

	/**
	 * 로그 출력용 Transaction ID Masking.
	 *
	 * @param txId Transaction ID
	 * @return Masking Transaction ID
	 * @exception
	 */
	private String maskTxId(String txId) {

		if (StringUtils.isBlank(txId)) {
			return "";
		}

		if (txId.length() <= 8) {
			return "****";
		}

		return txId.substring(0, 4)
				+ "****"
				+ txId.substring(
						txId.length() - 4);
	}
}
```
## 11. Transaction에서 특히 중요한 부분
`SellerAuthService`에는 다음을 붙이지 않습니다.
```java
@Transactional
@Service("sellerAuthService")
public class SellerAuthService {
```
이렇게 하면:
```text
Transaction START
 ↓
DB Connection 획득 가능
 ↓
이니시스 HTTP 5초 대기
 ↓
DB INSERT
 ↓
COMMIT
```
구조가 발생할 여지가 있습니다.
권장:
```text
SellerAuthService
 │
 ├─ InicisAuthClient          ← Transaction 없음
 │      ↓
 │   외부 통신
 │
 └─ joinSellerService
        ↓
     @Transactional
        ↓
     INSERT
        ↓
     COMMIT
```
즉 기존 `joinSellerService.insertSellerAuthHist()`가 Transaction Service라면 그대로 유지하면 됩니다.
- 만약 현재 해당 메소드에 Transaction이 없다면 **그 메소드 또는 별도 DB 저장 Service에만 `@Transactional`을 적용**하는 것이 좋습니다.
---
## 12. 기존 Controller 최종 수정
기존 150라인 이상 로직은 다음 수준까지 줄어듭니다.
```java
private SellerAuthService sellerAuthService;

/**
 *
 * Seller KG이니시스 통합인증 처리 Service 설정.
 *
 * @param sellerAuthService Seller 인증 Service
 * @return void
 * @exception
 *
 * <pre>
 * << 개정이력(Modification Information) >>
 *
 * 수정일         수정자                 수정내용
 * ----------  ------------- --------------------
 * 2026. 9. 1.    김원태       KG이니시스 연계 구조 개선
 * </pre>
 */
@Autowired
public void setSellerAuthService(
		SellerAuthService sellerAuthService) {

	this.sellerAuthService =
			sellerAuthService;
}

/**
 *
 * KG이니시스 통합인증 결과 처리.
 *
 * Controller에서는 Request Parameter 수신 및
 * View Model 설정만 담당하고 실제 인증 업무는
 * SellerAuthService에 위임한다.
 *
 * @param map KG이니시스 인증 Callback Parameter
 * @param model 화면 Model
 * @param view View 정보
 * @return 인증결과 화면
 * @exception
 *
 * <pre>
 * << 개정이력(Modification Information) >>
 *
 * 수정일         수정자                 수정내용
 * ----------  ------------- --------------------
 * 2026. 9. 1.    김원태       Connection Pool 및 연계 구조 개선
 * </pre>
 */
@RequestMapping(
		value = "/none/mm/resultSuccessFromInicis.do",
		method = RequestMethod.POST)
public String resultInfoFromInicis(
		@RequestParam Map<String, String> map,
		ModelMap model,
		ViewBaseVO view) {

	SellerAuthResultVO resultVO =
			sellerAuthService
					.processInicisAuth(map);

	model.addAttribute(
			"histSn",
			resultVO.getHistSn());

	model.addAttribute(
			"resultCode",
			resultVO.getResultCode());

	model.addAttribute(
			"resultMsg",
			resultVO.getResultMsg());

	model.addAttribute(
			"userName",
			resultVO.getUserName());

	return view.tiles(
			"none/mm/mim/authInfoPopup.tiles");
}
```
여기에는 더 이상 다음 코드가 존재하지 않습니다.
```java
@UseProxy
HttpURLConnection
URL.openConnection()
getOutputStream()
BufferedReader
conn.connect()
conn.disconnect()
JSONParser
decrypt(...)
CryptoUtil
SellerAuthHistVO
joinSellerService
```
Controller는 실제로:
```text
Request
  ↓
Service
  ↓
Model
  ↓
View
```
- 만 담당합니다.
---
## 13. Properties 권장값
```properties
#=========================================================
## KG이니시스 인증 연계
#=========================================================

## 기존 Property Key 유지
Globlas.seller.auth.mid=실제MID

## 기존 SEED IV
Globlas.seller.auth.bszIV=실제IV

## KG이니시스 결과조회 허용 Domain
Globals.seller.auth.allowedHosts=fcsa.inicis.com,kssa.inicis.com

## Proxy 사용 여부
## local 환경에서 직접 통신이면 N
## dev/prd에서 사내 Proxy 사용이면 Y
Globals.seller.auth.proxy.use=Y

## Proxy
Globals.proxy.host.url=10.0.0.1
Globals.proxy.host.port=8080
```
이니시스 현재 공식 연동정보에도 운영 대상이 `fcsa.inicis.com`, `kssa.inicis.com`, PORT 443이고 TLS 1.2 이상을 요구한다고 명시되어 있으므로 Allowlist로 관리하는 것이 현재 연동 규격과도 맞습니다. 
## `Globlas` 오타 주의
현재 코드가:
```java
EgovProperties.getProperty("Globlas.seller.auth.mid");
EgovProperties.getProperty("Globlas.seller.auth.bszIV");
```
를 사용하고 있으므로 이번 수정에서 **임의로 `Globals`로 변경하지 않았습니다.**
실제 properties에도 `Globlas`로 등록되어 있다면 그대로 유지해야 합니다.
장기적으로:
```text
Globlas → Globals
```
- 로 정리하는 것이 좋지만 코드와 property를 동시에 변경해야 합니다.
---
## 14. 기존 코드 대비 중요한 추가 개선
|항목|기존|최종 코드|
|---|---|---|
|Controller 외부연계|직접 호출|Service→Client|
|Connection|요청마다 직접 처리|Pool 재사용|
|Proxy|`@UseProxy` 가능성|HttpClient 전용|
|System Property|변경 가능|사용 안 함|
|Connect Timeout|3초|3초|
|Read Timeout|약 10초|**5초**|
|Pool Wait Timeout|없음|2초|
|Idle Connection|관리 없음|30초 제거|
|Connection TTL|없음|5분|
|Retry|불명확|POST 자동 Retry 차단|
|Redirect|불명확|차단|
|SSRF|`containsDomain()`|정확한 Host Allowlist|
|Response resultCode|검증 부족|검증|
|Response txId|검증 없음|**요청 txId와 비교**|
|JSON Loop|불필요한 반복|제거|
|개인정보 Log|평문 CI/전화/성명|**전부 제거**|
|Charset|Default Charset 혼용|UTF-8|
|DB Transaction|경계 불명확|외부연계 후 DB Transaction|
|Connection close|수동|Framework 반환|
|Controller 책임|과다|HTTP/View로 제한|

특히 공식 규격상 결과조회 응답에도 `resultCode`, `txId`가 존재하므로 단순히 HTTP 200만 확인하고 개인정보를 사용하는 것보다 **결과 `resultCode`와 요청/응답 `txId`를 모두 검증한 후 복호화하는 구조**가 안전합니다. 
## 최종 적용 시 확인할 3가지
1. `KISA_SEED_CBC`의 실제 package import는 현재 Controller의 기존 `decrypt()` 구현에서 사용 중인 것을 그대로 적용합니다. 공식 규격 자체도 token 사용 시 해당 개인정보가 SEED 암호화됨을 명시합니다. 
2. `JoinSellerService.insertSellerAuthHist()`의 기존 `@Transactional` 적용 여부를 확인합니다. `SellerAuthService.processInicisAuth()` 전체에는 Transaction을 걸지 않는 것이 핵심입니다.
3. 기존 `@UseProxy`가 `System.setProperty()` 방식이라면 **이 Controller에서는 반드시 제거**하고 새 `inicisHttpClient`의 Proxy만 사용해야 합니다.

이 구성은 기존 소스의 포맷과 업무 로직을 가능한 유지하면서 **Controller 책임 분리 + Connection Pool + Proxy 격리 + 자원 관리 + SSRF 대응 + Transaction 범위 축소 + 개인정보 로그 제거**까지 반영한 형태입니다. 