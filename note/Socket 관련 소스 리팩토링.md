### 리팩토링 방향

| 변경 전                                                   | 변경 후                                                         |
| ------------------------------------------------------ | ------------------------------------------------------------ |
| `sendTopSearch`, `sendAutoSearch`에 동일한 OkHttp 호출 로직 반복 | 공통 `postJson()` private 메소드로 통합                              |
| URL, 응답 VO 타입만 다름                                      | URL은 `Supplier<String>`, 응답 변환은 `Function<String, R>` 람다로 처리 |
| 로그 placeholder 개수 불일치 가능                               | 로그 포맷 정리                                                     |
| 매 요청마다 `MediaType.parse()` 수행                          | `static final MediaType`로 재사용                                |
| 응답/요청 전체 로그 출력 가능                                      | `maskResponse()` 사용 유지, 필요 시 길이 제한 권장                        |
| `@SuppressWarnings("null")` 사용                         | 제거 가능                                                        |
|                                                        |                                                              |
|                                                        |                                                              |

### 리팩토링 후 소스

```java
import java.io.IOException;
import java.net.SocketException;
import java.net.SocketTimeoutException;
import java.util.function.Function;
import java.util.function.Supplier;
import okhttp3.MediaType;
import okhttp3.Request;
import okhttp3.RequestBody;
import okhttp3.Response;
import okhttp3.ResponseBody;
```

```java
private static final MediaType JSON_MEDIA_TYPE =
        MediaType.parse("application/json; charset=utf-8");
```

```java
private SearchCommTopRespVO sendTopSearch(SearchCommTopVO searchVo) throws Exception {
    return postJson(
            "sendTopSearch",
            () -> GOODS_SEARCH_TOP_URL,
            searchVo,
            responseText -> gson.fromJson(responseText, SearchCommTopRespVO.class)
    );
}
```

```java
/**
 * 자동완성
 */
private SearchCommAutoRespVO sendAutoSearch(SearchCommAutoVO searchVo) throws Exception {
    return postJson(
            "sendAutoSearch",
            () -> GOODS_SEARCH_AUTO_URL,
            searchVo,
            responseText -> gson.fromJson(responseText, SearchCommAutoRespVO.class)
    );
}
```

```java
private <T, R> R postJson(
        String apiName,
        Supplier<String> urlSupplier,
        T requestVo,
        Function<String, R> responseMapper
) throws Exception {
    String url = urlSupplier.get();
    String requestJson = gson.toJson(requestVo);
    RequestBody body = RequestBody.create(JSON_MEDIA_TYPE, requestJson);
    Request request = new Request.Builder()
            .url(url)
            .post(body)
            .addHeader("Accept", "application/json")
            .build();
    long start = System.currentTimeMillis();
    try (Response response = productRecommenderOkHttpClient.newCall(request).execute()) {
        logConnectionPoolIfDebug();
        int status = response.code();
        ResponseBody responseBody = response.body();
        if (responseBody == null) {
            throw new IOException(apiName + " API response body is null. status=" + status);
        }
        String responseText = responseBody.string();
        long elapsedMs = System.currentTimeMillis() - start;
        if (!response.isSuccessful()) {
            log.debug(
                    "{} HTTP ERROR. status={}, elapsedMs={}, url={}, response={}",
                    apiName,
                    status,
                    elapsedMs,
                    url,
                    maskResponse(responseText)
            );
            throw new IOException(apiName + " API HTTP error. status=" + status);
        }
        log.debug(
                "{} SUCCESS. status={}, elapsedMs={}, url={}, response={}",
                apiName,
                status,
                elapsedMs,
                url,
                maskResponse(responseText)
        );
        return responseMapper.apply(responseText);
    } catch (EcException e) {
        log.error(
                "{} EcException. elapsedMs={}, url={}, request={}",
                apiName,
                System.currentTimeMillis() - start,
                url,
                maskResponse(requestJson),
                e
        );
        throw e;
    } catch (SocketTimeoutException e) {
        log.error(
                "{} TIMEOUT. elapsedMs={}, url={}, request={}",
                apiName,
                System.currentTimeMillis() - start,
                url,
                maskResponse(requestJson),
                e
        );
        throw e;
    } catch (SocketException e) {
        log.error(
                "{} SOCKET ERROR. elapsedMs={}, url={}, request={}",
                apiName,
                System.currentTimeMillis() - start,
                url,
                maskResponse(requestJson),
                e
        );
        throw e;
    } catch (IOException e) {
        log.error(
                "{} IO ERROR. elapsedMs={}, url={}, request={}",
                apiName,
                System.currentTimeMillis() - start,
                url,
                maskResponse(requestJson),
                e
        );
        throw e;
    } catch (Exception e) {
        log.error(
                "{} ERROR. elapsedMs={}, url={}, request={}",
                apiName,
                System.currentTimeMillis() - start,
                url,
                maskResponse(requestJson),
                e
        );
        throw e;
    }
}
```

```java
private void logConnectionPoolIfDebug() {
    if (!log.isDebugEnabled()) {
        return;
    }
    log.debug(
            "Connection Pool => idle Connection = [ {}/{} ]",
            productRecommenderOkHttpClient.connectionPool().idleConnectionCount(),
            productRecommenderOkHttpClient.connectionPool().connectionCount()
    );
}
```

### 주요 개선 사항

| 개선 항목                                   | 내용                                                 | 영향                     |
| --------------------------------------- | -------------------------------------------------- | ---------------------- |
| 공통 호출 메소드화                              | `postJson()` 하나로 OkHttp POST 호출 통합                 | 중복 제거, 유지보수성 향상        |
| 람다 기반 응답 변환                             | `responseText -> gson.fromJson(...)` 형태로 응답 타입별 파싱 | 함수형 구조 적용              |
| `Supplier<String>` 사용                   | URL을 람다로 전달                                        | URL 결정 로직이 복잡해져도 확장 가능 |
| `Function<String, R>` 사용                | 응답 문자열을 원하는 VO로 변환                                 | 응답 타입이 달라도 공통 처리 가능    |
| `MediaType` 상수화                         | `MediaType.parse()` 반복 제거                          | 미세하지만 불필요 객체 생성 감소     |
| `.method("POST", body)` → `.post(body)` | OkHttp 표준 POST 메소드 사용                              | 가독성 개선                 |
| `try-with-resources` 유지                 | `Response` 자동 close                                | 커넥션 반환 안정성 유지          |
| Debug 로그 조건 분리                          | `log.isDebugEnabled()` 확인 후 pool 로그 출력             | 불필요한 로그 연산 감소          |
| 로그 placeholder 정리                       | Throwable은 마지막 인자로 분리                              | Stack trace 정상 출력      |
| 응답 로그 mask 적용                           | 성공 응답에도 `maskResponse()` 적용                        | 개인정보/대용량 로그 위험 완화      |
|                                         |                                                    |                        |

### 리소스 관리 관점 검토

| 항목                      | 기존 상태                                  | 개선/판단                          |
| ----------------------- | -------------------------------------- | ------------------------------ |
| `Response` close        | `try-with-resources` 사용                | 적절함                            |
| `ResponseBody` close    | `Response.close()`로 함께 정리              | 적절함                            |
| 커넥션 풀 재사용               | 동일 `productRecommenderOkHttpClient` 사용 | 적절함                            |
| `ResponseBody.string()` | 전체 응답을 메모리에 로딩                         | 응답이 작은 추천 API면 허용, 대용량 응답이면 주의 |
| 요청/응답 로그                | 전체 JSON 로그 가능성                         | `maskResponse()` + 길이 제한 권장    |
| `MediaType.parse()`     | 매번 생성                                  | 상수화로 개선                        |

### 부하/속도 관리 개선 포인트

#### 1. 성공 응답 전체 로그는 운영에서 최소화

기존 코드:

```java
log.debug("Success. elapsedMs={}, url={}, response={} : {}", ...);
```

추천 API 응답이 커지면 로그 I/O가 성능에 영향을 줄 수 있습니다.
권장:

```java
log.debug(
        "{} SUCCESS. status={}, elapsedMs={}, url={}",
        apiName,
        status,
        elapsedMs,
        url
);
```

운영에서는 응답 전문 대신 `status`, `elapsedMs`, `url`, `요청 식별자`, `상품 수` 정도만 남기는 것이 좋습니다.

#### 2. `maskResponse()`에 길이 제한 추가 권장

```java
private String maskResponse(String value) {
    if (value == null) {
        return null;
    }
    String masked = value
            .replaceAll("(?i)(\"password\"\\s*:\\s*\")[^\"]*(\")", "$1***$2")
            .replaceAll("(?i)(\"token\"\\s*:\\s*\")[^\"]*(\")", "$1***$2");
    int maxLength = 2_000;
    if (masked.length() > maxLength) {
        return masked.substring(0, maxLength) + "...(truncated)";
    }
    return masked;
}
```

| 효과          | 설명                   |
| ----------- | -------------------- |
| 로그 파일 급증 방지 | 대량 응답이 반복 저장되는 문제 완화 |
| 디스크 I/O 감소  | 부하 상황에서 로그 병목 완화     |
| 개인정보 노출 완화  | token/password 등 마스킹 |

#### 3. OkHttpClient는 API 그룹 단위 Bean으로 유지

현재처럼 `productRecommenderOkHttpClient`를 공유하는 구조는 좋습니다.
단, `sendTopSearch`, `sendAutoSearch`가 같은 외부 상품추천 시스템이고 timeout/장애격리 정책이 같다면 같은 Client를 쓰는 것이 맞습니다.

| 기준                      | 권장                 |
| ----------------------- | ------------------ |
| 같은 외부 시스템               | 동일 OkHttpClient    |
| timeout 정책 동일           | 동일 OkHttpClient    |
| 장애 격리 정책 동일             | 동일 OkHttpClient    |
| API 중요도/호출량/timeout이 다름 | 별도 OkHttpClient 고려 |

### OkHttpClient 설정 예시

```java
@Bean
public OkHttpClient productRecommenderOkHttpClient() {
    return new OkHttpClient.Builder()
            .connectionPool(new ConnectionPool(20, 60, TimeUnit.SECONDS))
            .connectTimeout(1, TimeUnit.SECONDS)
            .readTimeout(2, TimeUnit.SECONDS)
            .writeTimeout(2, TimeUnit.SECONDS)
            .callTimeout(3, TimeUnit.SECONDS)
            .retryOnConnectionFailure(false)
            .build();
}
```

| 설정                                | 의미                    |
| --------------------------------- | --------------------- |
| `connectionPool(20, 60s)`         | 유휴 커넥션 최대 20개, 60초 유지 |
| `connectTimeout`                  | TCP 연결 제한 시간          |
| `readTimeout`                     | 응답 읽기 제한 시간           |
| `writeTimeout`                    | 요청 전송 제한 시간           |
| `callTimeout`                     | 전체 호출 제한 시간           |
| `retryOnConnectionFailure(false)` | 자동 재시도로 인한 중복 호출 방지   |

- 커머스 검색/추천 API는 사용자 요청 흐름에 직접 연결되는 경우가 많기 때문에 `callTimeout`을 반드시 두는 것이 좋습니다. `connectTimeout`, `readTimeout`만 있으면 전체 호출 시간이 예상보다 길어질 수 있습니다.

---
### 최종 판단

현재 코드는 `try-with-resources`를 사용하고 있어 리소스 반환 구조는 큰 문제 없습니다. 다만 중복 코드가 많고, 로그 포맷 오류 가능성, 응답 전문 로그, 매번 `MediaType.parse()` 수행, API별 예외 처리 반복이 개선 대상입니다. 위처럼 `postJson()` 공통 메소드로 통합하면 `sendTopSearch`, `sendAutoSearch`뿐 아니라 향후 `sendSearch`, `sendRelationSearch`도 같은 방식으로 확장할 수 있습니다.
