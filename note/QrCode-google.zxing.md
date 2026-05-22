## Google ZXing 패키지 설명 및 QR 코드 생성 예제

### 1. ZXing 개요

ZXing은 Google에서 시작된 오픈소스 바코드/QR 코드 처리 라이브러리입니다. Java 기준으로 `core` 모듈은 바코드 인코딩/디코딩의 핵심 기능을 제공하고, `javase` 모듈은 `BitMatrix`를 이미지 파일·스트림·`BufferedImage`로 변환하는 Java SE 유틸리티를 제공합니다. Maven Central 기준 현재 `com.google.zxing:core`, `com.google.zxing:javase`의 최신 버전은 `3.5.4`입니다. ([Maven Central](https://central.sonatype.com/artifact/com.google.zxing/core "Maven Central: com.google.zxing:core"))

### 2. 주요 패키지 구조

| 구분     | 패키지/클래스                                                | 설명                                                           |
| ------ | ------------------------------------------------------ | ------------------------------------------------------------ |
| 핵심     | `com.google.zxing`                                     | `BarcodeFormat`, `EncodeHintType`, `WriterException` 등 공통 타입 |
| QR 생성  | `com.google.zxing.qrcode.QRCodeWriter`                 | 문자열을 QR 코드용 `BitMatrix`로 변환                                  |
| 이미지 변환 | `com.google.zxing.client.j2se.MatrixToImageWriter`     | `BitMatrix`를 PNG/JPG 이미지로 출력                                 |
| 오류 보정  | `com.google.zxing.qrcode.decoder.ErrorCorrectionLevel` | QR 손상 복원 수준 지정                                               |

- `QRCodeWriter#encode()`는 문자열, 바코드 포맷, 가로/세로 크기, 추가 힌트를 받아 `BitMatrix`를 반환합니다. 공식 Javadoc상 `contents`, `BarcodeFormat`, `width`, `height`, `hints`를 인자로 받을 수 있고, 인코딩이 불가능하면 `WriterException`이 발생합니다. ([zxing.github.io](https://zxing.github.io/zxing/apidocs/com/google/zxing/qrcode/QRCodeWriter.html "QRCodeWriter (ZXing 3.5.4 API)"))
### 3. Maven 의존성

```xml
<properties>
    <zxing.version>3.5.4</zxing.version>
</properties>
<dependencies>
    <dependency>
        <groupId>com.google.zxing</groupId>
        <artifactId>core</artifactId>
        <version>${zxing.version}</version>
    </dependency>
    <dependency>
        <groupId>com.google.zxing</groupId>
        <artifactId>javase</artifactId>
        <version>${zxing.version}</version>
    </dependency>
</dependencies>
```

`javase` 모듈은 Java SE 이미지 처리용 확장 모듈이며 내부적으로 `core`를 의존합니다. 다만 실무에서는 `core`와 `javase` 버전을 명시적으로 동일하게 맞추는 편이 `NoSuchMethodError`, `ClassNotFoundException` 예방에 유리합니다. Maven Central POM에서도 `javase`가 `core`를 의존하는 구조로 확인됩니다. ([Maven Central](https://central.sonatype.com/artifact/com.google.zxing/javase "Maven Central: com.google.zxing:javase"))

### 4. 실무용 QR 코드 생성 유틸리티

```java
package com.example.common.qr;
import com.google.zxing.BarcodeFormat;
import com.google.zxing.EncodeHintType;
import com.google.zxing.WriterException;
import com.google.zxing.client.j2se.MatrixToImageWriter;
import com.google.zxing.common.BitMatrix;
import com.google.zxing.qrcode.QRCodeWriter;
import com.google.zxing.qrcode.decoder.ErrorCorrectionLevel;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.EnumMap;
import java.util.Map;
public final class QrCodeUtil {
    private QrCodeUtil() {
    }
    public static byte[] generatePng(String contents, int width, int height) {
        validate(contents, width, height);
        try {
            Map<EncodeHintType, Object> hints = new EnumMap<>(EncodeHintType.class);
            hints.put(EncodeHintType.CHARACTER_SET, StandardCharsets.UTF_8.name());
            hints.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.M);
            hints.put(EncodeHintType.MARGIN, 2);
            BitMatrix matrix = new QRCodeWriter().encode(
                    contents,
                    BarcodeFormat.QR_CODE,
                    width,
                    height,
                    hints
            );
            try (ByteArrayOutputStream out = new ByteArrayOutputStream()) {
                MatrixToImageWriter.writeToStream(matrix, "PNG", out);
                return out.toByteArray();
            }
        } catch (WriterException | IOException e) {
            throw new IllegalStateException("QR 코드 생성 실패", e);
        }
    }
    private static void validate(String contents, int width, int height) {
        if (contents == null || contents.trim().isEmpty()) {
            throw new IllegalArgumentException("QR 코드 내용은 필수입니다.");
        }
        if (width < 100 || height < 100) {
            throw new IllegalArgumentException("QR 코드 크기는 최소 100px 이상을 권장합니다.");
        }
        if (contents.length() > 1000) {
            throw new IllegalArgumentException("QR 코드 내용이 너무 깁니다. URL 또는 짧은 식별자 사용을 권장합니다.");
        }
    }
}
```

### 5. Spring Framework 5.3 MVC Controller 예제

```java
package com.example.web.qr;
import com.example.common.qr.QrCodeUtil;
import org.springframework.http.CacheControl;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import java.nio.charset.StandardCharsets;
import java.time.Duration;
@Controller
public class QrCodeController {
    @GetMapping(value = "/common/qr", produces = MediaType.IMAGE_PNG_VALUE)
    public ResponseEntity<byte[]> qr(@RequestParam("data") String data) {
        byte[] image = QrCodeUtil.generatePng(data, 300, 300);
        return ResponseEntity.ok()
                .contentType(MediaType.IMAGE_PNG)
                .contentLength(image.length)
                .cacheControl(CacheControl.noStore())
                .header(HttpHeaders.CONTENT_DISPOSITION, "inline; filename=\"qr.png\"")
                .body(image);
    }
}
```

브라우저 호출 예시는 다음과 같습니다.

```text
/common/qr?data=https%3A%2F%2Fexample.com%2Forder%2F12345
```

### 6. 실무 적용 시 권장 구조

|구분|권장 방식|이유|
|---|---|---|
|QR 내용|긴 JSON보다 짧은 URL/식별자|QR 밀도 감소, 인식률 향상|
|개인정보|QR에 직접 저장 금지|이미지 캡처·공유 시 정보 유출 가능|
|주문/쿠폰 QR|서버 검증용 일회성 토큰 사용|위조·재사용 방지|
|이미지 형식|PNG 권장|QR처럼 선명한 흑백 이미지에 적합|
|캐시|개인화 QR은 `no-store`|타 사용자 노출 방지|
|크기|최소 200~300px 이상 권장|모바일 스캔 안정성 확보|
|여백|`MARGIN` 유지|QR Quiet Zone이 잘리면 인식률 저하|

### 7. 오류 보정 레벨 선택 기준

| 레벨  |   복원력 | 데이터 밀도 | 실무 용도               |
| --- | ----: | -----: | ------------------- |
| L   |    낮음 |     낮음 | 화면 전용, 깨끗한 환경       |
| M   |    보통 |     보통 | 일반 웹/모바일 QR 기본값     |
| Q   |    높음 |     높음 | 인쇄물, 약간의 오염 가능성     |
| H   | 매우 높음 |  매우 높음 | 로고 삽입, 훼손 가능성 있는 QR |

- 주의할 점은 오류 보정 레벨을 높이면 복원력은 좋아지지만 QR 패턴이 복잡해집니다. 데이터가 긴 상태에서 `H`를 쓰면 오히려 인식이 어려워질 수 있으므로, 실무 기본값은 `M`, 로고 삽입이나 인쇄물은 `Q` 또는 `H`를 검토하는 방식이 적절합니다.

### 8. 실무 주의점

#### 8-1. QR에 민감정보를 직접 넣지 말 것

```text
비권장: 이름, 전화번호, 주문번호, 회원ID, 인증토큰 원문
권장: 서버에서 검증 가능한 짧은 난수 토큰 또는 단축 URL
```

QR 코드는 사실상 평문 이미지입니다. 누구나 스캔하면 내용을 볼 수 있으므로 결제, 회원, 주문, 출입 인증에는 반드시 서버 검증 로직을 둬야 합니다.

#### 8-2. 일회성/만료 정책 적용

```text
예: https://example.com/q/8f3a9c2d
```

서버에서는 해당 토큰에 대해 다음 항목을 검증하는 것이 안전합니다.

```text
존재 여부
만료 시간
사용 여부
요청자 권한
상태값
서명값 또는 위변조 여부
```

#### 8-3. URL은 반드시 인코딩해서 전달

브라우저에서 QR 생성 API를 GET으로 호출할 경우 `data` 값에 `&`, `?`, `=`, 한글, 공백이 포함될 수 있습니다. 클라이언트에서는 `encodeURIComponent()`를 사용해야 합니다.

```javascript
const url = "/common/qr?data=" + encodeURIComponent("https://example.com/order?id=123&name=홍길동");
```

#### 8-4. 버전 충돌 주의

`MatrixToImageWriter`는 `javase` 모듈에 있습니다. `core`만 추가하면 `QRCodeWriter`는 사용할 수 있어도 이미지 변환 클래스가 없어서 컴파일 오류가 날 수 있습니다. 또한 `core`와 `javase` 버전이 다르면 런타임 오류가 발생할 수 있으므로 동일 버전으로 고정하는 것이 안전합니다.

#### 8-5. 대량 생성 시 파일 저장보다 바이트 응답 우선

단순 조회용 QR은 매번 파일로 저장하지 말고 `byte[]` 또는 스트림으로 바로 응답하는 편이 운영 관리가 쉽습니다. 다만 동일 QR을 반복적으로 제공하고 트래픽이 많다면 서버 검증 정책을 고려해 캐시 가능한 정적 이미지로 분리할 수 있습니다.

### 9. 커머스 시스템 적용 예시

|업무|QR 내용|주의점|
|---|---|---|
|주문 상세|주문 상세 URL + 만료 토큰|회원 권한 검증 필수|
|쿠폰 사용|쿠폰 사용 토큰|1회 사용 처리 필요|
|오프라인 픽업|픽업 확인 토큰|상태값: 대기/수령/취소 관리|
|반품 접수|반품 접수 URL|주문자 권한 확인|
|이벤트 응모|응모 페이지 URL|중복 응모 방지|

### 10. 결론

Spring Framework 5.3 기반 Java 웹 시스템에서는 `com.google.zxing:core`와 `com.google.zxing:javase`를 같은 버전으로 사용하고, `QRCodeWriter`로 `BitMatrix`를 생성한 뒤 `MatrixToImageWriter`로 PNG 응답을 만드는 방식이 가장 단순하고 실무 적용성이 높습니다. 운영에서는 QR 생성 자체보다 QR에 담는 값의 보안성, 토큰 만료, 재사용 방지, 개인정보 미포함, 캐시 정책이 더 중요합니다.

## QR 코드의 보안성, 토큰 만료, 재사용 방지, 캐시

### QR 보안 실무 로직 추천

#### 핵심 결론

QR에는 **주문번호, 회원ID, 전화번호, 인증값 원문, 쿠폰번호 원문**을 직접 넣지 말고, 서버에서 검증 가능한 **짧은 난수 토큰 URL**만 넣는 방식이 가장 안전합니다. OWASP는 세션/토큰성 식별자는 충분한 엔트로피와 CSPRNG 기반 난수성을 가져야 하며, 값 자체에는 민감정보나 PII를 포함하지 말고 의미 있는 업무 정보는 서버 측 저장소에 보관하라고 권장합니다. ([OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html "Session Management - OWASP Cheat Sheet Series"))

```text
권장 QR 값:
https://www.example.com/qr/pickup?t=랜덤토큰
비권장 QR 값:
주문번호=12345&회원ID=hong&전화번호=010...
```

### 1. 권장 아키텍처

```text
[사용자/직원 QR 스캔]
        ↓
[QR URL 접근]
        ↓
[토큰 HMAC 해시 생성]
        ↓
[DB에서 토큰 행 SELECT ... FOR UPDATE]
        ↓
[만료/상태/권한/업무상태 검증]
        ↓
[USED 처리]
        ↓
[주문수령/쿠폰사용/이벤트참여 처리]
```

#### 권장 정책

|구분|권장값|이유|
|---|---|---|
|QR 내용|랜덤 토큰 URL|QR 노출 시에도 업무정보 직접 유출 방지|
|토큰 생성|`SecureRandom` 32 bytes 이상|충분한 난수성 확보|
|DB 저장|토큰 원문 저장 금지, HMAC-SHA256 해시 저장|DB 유출 시 토큰 원문 사용 방지|
|만료|5~30분 기본|캡처/공유 위험 축소|
|재사용|1회 사용 후 `USED`|캡처 재사용 방지|
|검증|DB 트랜잭션 + Row Lock|동시 스캔 Race Condition 방지|
|응답 캐시|개인화 QR은 `Cache-Control: no-store`|브라우저/CDN 저장 방지|

### 2. DB 테이블 예제

MariaDB 10.6 기준 예시입니다.

```sql
CREATE TABLE qr_token (
    id BIGINT NOT NULL AUTO_INCREMENT,
    purpose VARCHAR(30) NOT NULL,
    target_type VARCHAR(30) NOT NULL,
    target_id BIGINT NOT NULL,
    user_id BIGINT NULL,
    token_hash CHAR(64) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'ISSUED',
    expires_at DATETIME(3) NOT NULL,
    used_at DATETIME(3) NULL,
    used_by VARCHAR(100) NULL,
    created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    PRIMARY KEY (id),
    UNIQUE KEY uk_qr_token_hash (token_hash),
    KEY ix_qr_target (purpose, target_type, target_id, status, expires_at),
    KEY ix_qr_expire (status, expires_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### 상태값 예시

|상태|의미|
|---|---|
|`ISSUED`|발급됨, 아직 사용 가능|
|`USED`|정상 사용 완료|
|`EXPIRED`|만료 처리됨|
|`REVOKED`|재발급 등으로 폐기됨|

### 3. 토큰 생성/HMAC 유틸

JDK 11 기준입니다. `HexFormat`은 JDK 17부터 제공되므로 직접 Hex 변환을 사용합니다.

```java
package com.example.qr;
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.nio.charset.StandardCharsets;
import java.security.SecureRandom;
import java.util.Base64;
public final class QrTokenCrypto {
    private static final SecureRandom RANDOM = new SecureRandom();
    private static final int TOKEN_BYTES = 32;
    private QrTokenCrypto() {
    }
    public static String newRawToken() {
        byte[] bytes = new byte[TOKEN_BYTES];
        RANDOM.nextBytes(bytes);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
    }
    public static String hmacSha256Hex(String rawToken, String secret) {
        try {
            Mac mac = Mac.getInstance("HmacSHA256");
            mac.init(new SecretKeySpec(secret.getBytes(StandardCharsets.UTF_8), "HmacSHA256"));
            byte[] digest = mac.doFinal(rawToken.getBytes(StandardCharsets.UTF_8));
            return toHex(digest);
        } catch (Exception e) {
            throw new IllegalStateException("QR 토큰 해시 생성 실패", e);
        }
    }
    private static String toHex(byte[] bytes) {
        char[] hexArray = "0123456789abcdef".toCharArray();
        char[] hexChars = new char[bytes.length * 2];
        for (int i = 0; i < bytes.length; i++) {
            int v = bytes[i] & 0xFF;
            hexChars[i * 2] = hexArray[v >>> 4];
            hexChars[i * 2 + 1] = hexArray[v & 0x0F];
        }
        return new String(hexChars);
    }
}
```

### 4. Repository 예제

```java
package com.example.qr;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;
import java.sql.Timestamp;
import java.time.LocalDateTime;
@Repository
public class QrTokenRepository {
    private final JdbcTemplate jdbcTemplate;
    public QrTokenRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
    public void revokeActiveTokens(String purpose, String targetType, long targetId) {
        String sql =
            "UPDATE qr_token " +
            "SET status = 'REVOKED' " +
            "WHERE purpose = ? AND target_type = ? AND target_id = ? AND status = 'ISSUED'";
        jdbcTemplate.update(sql, purpose, targetType, targetId);
    }
    public void insert(String purpose, String targetType, long targetId, Long userId, String tokenHash, LocalDateTime expiresAt) {
        String sql =
            "INSERT INTO qr_token " +
            "(purpose, target_type, target_id, user_id, token_hash, status, expires_at) " +
            "VALUES (?, ?, ?, ?, ?, 'ISSUED', ?)";
        jdbcTemplate.update(sql, purpose, targetType, targetId, userId, tokenHash, Timestamp.valueOf(expiresAt));
    }
    public QrTokenRow findForUpdate(String tokenHash) {
        String sql =
            "SELECT id, purpose, target_type, target_id, user_id, token_hash, status, expires_at, used_at " +
            "FROM qr_token " +
            "WHERE token_hash = ? " +
            "FOR UPDATE";
        return jdbcTemplate.query(sql, rs -> {
            if (!rs.next()) {
                return null;
            }
            QrTokenRow row = new QrTokenRow();
            row.setId(rs.getLong("id"));
            row.setPurpose(rs.getString("purpose"));
            row.setTargetType(rs.getString("target_type"));
            row.setTargetId(rs.getLong("target_id"));
            long userId = rs.getLong("user_id");
            row.setUserId(rs.wasNull() ? null : userId);
            row.setTokenHash(rs.getString("token_hash"));
            row.setStatus(rs.getString("status"));
            row.setExpiresAt(rs.getTimestamp("expires_at").toLocalDateTime());
            Timestamp usedAt = rs.getTimestamp("used_at");
            row.setUsedAt(usedAt == null ? null : usedAt.toLocalDateTime());
            return row;
        }, tokenHash);
    }
    public int markUsed(long id, String usedBy) {
        String sql =
            "UPDATE qr_token " +
            "SET status = 'USED', used_at = CURRENT_TIMESTAMP(3), used_by = ? " +
            "WHERE id = ? AND status = 'ISSUED'";
        return jdbcTemplate.update(sql, usedBy, id);
    }
}
```

```java
package com.example.qr;
import java.time.LocalDateTime;
public class QrTokenRow {
    private long id;
    private String purpose;
    private String targetType;
    private long targetId;
    private Long userId;
    private String tokenHash;
    private String status;
    private LocalDateTime expiresAt;
    private LocalDateTime usedAt;
    public long getId() { return id; }
    public void setId(long id) { this.id = id; }
    public String getPurpose() { return purpose; }
    public void setPurpose(String purpose) { this.purpose = purpose; }
    public String getTargetType() { return targetType; }
    public void setTargetType(String targetType) { this.targetType = targetType; }
    public long getTargetId() { return targetId; }
    public void setTargetId(long targetId) { this.targetId = targetId; }
    public Long getUserId() { return userId; }
    public void setUserId(Long userId) { this.userId = userId; }
    public String getTokenHash() { return tokenHash; }
    public void setTokenHash(String tokenHash) { this.tokenHash = tokenHash; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public LocalDateTime getExpiresAt() { return expiresAt; }
    public void setExpiresAt(LocalDateTime expiresAt) { this.expiresAt = expiresAt; }
    public LocalDateTime getUsedAt() { return usedAt; }
    public void setUsedAt(LocalDateTime usedAt) { this.usedAt = usedAt; }
}
```

### 5. Service 예제

```java
package com.example.qr;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.LocalDateTime;
@Service
public class QrTokenService {
    private static final String PURPOSE_PICKUP = "PICKUP";
    private static final String TARGET_ORDER = "ORDER";
    private final QrTokenRepository qrTokenRepository;
    private final String qrHmacSecret = "change-this-secret-from-secure-config";
    private final String qrBaseUrl = "https://www.example.com/qr/pickup";
    public QrTokenService(QrTokenRepository qrTokenRepository) {
        this.qrTokenRepository = qrTokenRepository;
    }
    @Transactional
    public String issuePickupQrUrl(long orderId, long userId) {
        String rawToken = QrTokenCrypto.newRawToken();
        String tokenHash = QrTokenCrypto.hmacSha256Hex(rawToken, qrHmacSecret);
        LocalDateTime expiresAt = LocalDateTime.now().plusMinutes(10);
        qrTokenRepository.revokeActiveTokens(PURPOSE_PICKUP, TARGET_ORDER, orderId);
        qrTokenRepository.insert(PURPOSE_PICKUP, TARGET_ORDER, orderId, userId, tokenHash, expiresAt);
        return qrBaseUrl + "?t=" + rawToken;
    }
    @Transactional
    public QrConsumeResult consumePickupToken(String rawToken, String usedBy) {
        if (rawToken == null || rawToken.trim().isEmpty()) {
            return QrConsumeResult.invalid();
        }
        String tokenHash = QrTokenCrypto.hmacSha256Hex(rawToken, qrHmacSecret);
        QrTokenRow row = qrTokenRepository.findForUpdate(tokenHash);
        if (row == null) {
            return QrConsumeResult.invalid();
        }
        if (!PURPOSE_PICKUP.equals(row.getPurpose())) {
            return QrConsumeResult.invalid();
        }
        if (!"ISSUED".equals(row.getStatus())) {
            return QrConsumeResult.alreadyUsedOrRevoked();
        }
        if (LocalDateTime.now().isAfter(row.getExpiresAt())) {
            return QrConsumeResult.expired();
        }
        /*
         * 실무에서는 여기서 주문 상태를 추가 검증해야 함.
         * 예:
         * - 주문이 존재하는가?
         * - 주문 상태가 PICKUP_READY 인가?
         * - 이미 수령 완료된 주문은 아닌가?
         * - 스캔한 직원/지점이 해당 주문을 처리할 권한이 있는가?
         */
        int updated = qrTokenRepository.markUsed(row.getId(), usedBy);
        if (updated != 1) {
            return QrConsumeResult.invalid();
        }
        return QrConsumeResult.success(row.getTargetId());
    }
}
```

```java
package com.example.qr;
public class QrConsumeResult {
    private final boolean success;
    private final String code;
    private final Long targetId;
    private QrConsumeResult(boolean success, String code, Long targetId) {
        this.success = success;
        this.code = code;
        this.targetId = targetId;
    }
    public static QrConsumeResult success(long targetId) {
        return new QrConsumeResult(true, "SUCCESS", targetId);
    }
    public static QrConsumeResult invalid() {
        return new QrConsumeResult(false, "INVALID", null);
    }
    public static QrConsumeResult expired() {
        return new QrConsumeResult(false, "EXPIRED", null);
    }
    public static QrConsumeResult alreadyUsedOrRevoked() {
        return new QrConsumeResult(false, "ALREADY_USED_OR_REVOKED", null);
    }
    public boolean isSuccess() { return success; }
    public String getCode() { return code; }
    public Long getTargetId() { return targetId; }
}
```

#### Service 로직 핵심

|항목|실무 판단|
|---|---|
|`revokeActiveTokens()`|같은 주문의 이전 QR을 폐기하여 최신 QR만 사용 가능하게 함|
|`findForUpdate()`|동시 스캔 시 하나의 트랜잭션만 먼저 처리하도록 Row Lock 적용|
|`markUsed()`|사용 완료 상태로 변경|
|외부 응답|가능하면 상세 사유 노출 최소화|
|내부 로그|만료/재사용/권한오류 상세 기록|

### 6. Spring MVC Controller 예제

```java
package com.example.qr;
import com.example.common.qr.QrCodeUtil;
import org.springframework.http.CacheControl;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.*;
@Controller
public class QrController {
    private final QrTokenService qrTokenService;
    public QrController(QrTokenService qrTokenService) {
        this.qrTokenService = qrTokenService;
    }
    @GetMapping(value = "/orders/{orderId}/pickup-qr.png", produces = MediaType.IMAGE_PNG_VALUE)
    public ResponseEntity<byte[]> issuePickupQr(@PathVariable long orderId) {
        long loginUserId = 1001L; // 실무에서는 세션/인증 객체에서 취득
        String qrUrl = qrTokenService.issuePickupQrUrl(orderId, loginUserId);
        byte[] png = QrCodeUtil.generatePng(qrUrl, 300, 300);
        return ResponseEntity.ok()
                .contentType(MediaType.IMAGE_PNG)
                .contentLength(png.length)
                .cacheControl(CacheControl.noStore())
                .header(HttpHeaders.PRAGMA, "no-cache")
                .header(HttpHeaders.EXPIRES, "0")
                .header("Referrer-Policy", "no-referrer")
                .body(png);
    }
    @GetMapping("/qr/pickup")
    @ResponseBody
    public ResponseEntity<QrConsumeResponse> consumePickupQr(@RequestParam("t") String token) {
        String usedBy = "staff001"; // 실무에서는 로그인 직원/스캐너 ID
        QrConsumeResult result = qrTokenService.consumePickupToken(token, usedBy);
        QrConsumeResponse body = QrConsumeResponse.from(result);
        return ResponseEntity.ok()
                .cacheControl(CacheControl.noStore())
                .header(HttpHeaders.PRAGMA, "no-cache")
                .header(HttpHeaders.EXPIRES, "0")
                .header("Referrer-Policy", "no-referrer")
                .body(body);
    }
}
```

```java
package com.example.qr;
public class QrConsumeResponse {
    private boolean success;
    private String message;
    public static QrConsumeResponse from(QrConsumeResult result) {
        QrConsumeResponse response = new QrConsumeResponse();
        response.success = result.isSuccess();
        if (result.isSuccess()) {
            response.message = "정상 처리되었습니다.";
        } else {
            response.message = "사용할 수 없는 QR입니다.";
        }
        return response;
    }
    public boolean isSuccess() { return success; }
    public String getMessage() { return message; }
}
```

Spring `CacheControl.noStore()`는 `Cache-Control: no-store` 지시어 생성을 지원하고, `ResponseEntity`는 `cacheControl()`로 캐시 정책을 설정할 수 있습니다. ([docs.enterprise.spring.io](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/http/class-use/CacheControl.html "Uses of Class org.springframework.http.CacheControl (Spring Framework 5.3.46 API)")) Spring 공식 문서에서도 `CacheControl.noStore()`는 캐싱 방지 용도로 제시됩니다. ([Home](https://docs.spring.io/spring-framework/reference/web/webflux/caching.html "HTTP Caching :: Spring Framework"))

### 7. 캐시 정책 추천

HTTP `no-store`는 private/shared cache 모두 응답을 저장하지 말라는 의미이고, `no-cache`는 저장 자체를 막는 것이 아니라 재사용 전 원 서버 재검증을 요구하는 의미입니다. 개인화 QR, 일회성 QR, 인증성 QR에는 `no-cache`보다 `no-store`가 적합합니다. ([MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control "Cache-Control header - HTTP | MDN"))

|대상|권장 Cache-Control|설명|
|---|---|---|
|개인 주문 QR 이미지|`no-store`|브라우저/CDN 저장 금지|
|픽업/쿠폰 검증 결과|`no-store`|검증 결과 재사용 방지|
|로그인 후 QR 페이지|`no-store` 또는 최소 `private, no-store`|사용자별 응답 보호|
|마케팅용 고정 QR|`public, max-age=86400` 가능|민감정보 없고 모두에게 동일한 경우|
|정적 QR 안내 이미지|`public, max-age` 가능|토큰 미포함 시 가능|

### 8. 목적별 만료/재사용 정책

|업무|만료|재사용|권장 검증|
|---|--:|---|---|
|오프라인 픽업|5~10분|1회|주문상태, 지점, 직원권한|
|쿠폰 사용|10~30분|1회|쿠폰상태, 회원, 사용기간|
|반품 접수|10~30분|1회 또는 제한적 다회|주문자 권한, 반품 가능 상태|
|이벤트 출석|1~5분|1회|행사/장소/시간 검증|
|상품 상세 공유|장기 가능|다회|민감정보 없음|

### 9. 운영 주의점

#### 9-1. 토큰은 URL에 노출된다

QR은 대부분 URL로 열리므로 토큰이 브라우저 히스토리, 웹서버 Access Log, WAF Log, CDN Log에 남을 수 있습니다. 따라서 토큰에는 의미 있는 정보를 넣지 말고, 짧은 만료와 1회 사용 정책을 반드시 적용하는 것이 안전합니다.

#### 9-2. 외부 응답은 단순화한다

```text
내부 로그:
EXPIRED, USED, REVOKED, ORDER_NOT_READY, BRANCH_MISMATCH
외부 응답:
사용할 수 없는 QR입니다.
```

공격자에게 “존재하는 토큰이지만 만료됨”, “이미 사용됨” 같은 힌트를 자세히 주지 않는 편이 좋습니다.

#### 9-3. DB 트랜잭션이 반드시 필요하다

`SELECT ... FOR UPDATE`는 트랜잭션 안에서 의미가 있습니다. Spring Framework 5.3 기준으로는 `@Transactional`이 적용되는 public service method에서 Repository를 호출해야 하며, 같은 클래스 내부 self-invocation으로 호출하면 프록시 기반 트랜잭션이 적용되지 않을 수 있습니다.

#### 9-4. JWT보다 서버 저장형 토큰이 유리한 경우

QR의 재사용 방지, 즉시 폐기, 상태 관리가 중요하면 JWT보다 DB 저장형 opaque token이 단순하고 안전합니다. JWT는 자체적으로 서명 검증은 가능하지만, “1회 사용 후 즉시 무효화”나 “관리자 폐기”를 하려면 결국 서버 측 상태 저장소가 필요합니다.

### 10. 최종 권장안

커머스 시스템의 주문 픽업/쿠폰/반품 QR은 다음 구조를 권장합니다.

```text
QR 값:
https://도메인/qr/{업무}?t={랜덤토큰}
서버 저장:
HMAC-SHA256(랜덤토큰), purpose, target_id, user_id, status, expires_at
검증:
1. 토큰 HMAC 해시 계산
2. DB row lock
3. ISSUED 상태 확인
4. 만료시간 확인
5. 주문/쿠폰/권한 상태 확인
6. USED 상태 변경
7. 업무 처리
응답:
Cache-Control: no-store
Pragma: no-cache
Expires: 0
Referrer-Policy: no-referrer
```

이 방식은 QR 이미지가 유출되더라도 원문 개인정보가 없고, 토큰이 짧은 시간 후 만료되며, 한 번 사용되면 재사용이 차단되기 때문에 Spring Framework 5.3 기반 실무 시스템에 적용하기 적절합니다.

## QR생성 개선

```java
package sample.util;

import java.awt.AlphaComposite;
import java.awt.Graphics2D;
import java.awt.image.BufferedImage;
import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.IOException;
import java.util.Base64;
import java.util.Hashtable;

import javax.imageio.ImageIO;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import com.google.zxing.BarcodeFormat;
import com.google.zxing.EncodeHintType;
import com.google.zxing.MultiFormatWriter;
import com.google.zxing.WriterException;
import com.google.zxing.client.j2se.MatrixToImageWriter;
import com.google.zxing.common.BitMatrix;
import com.google.zxing.qrcode.QRCodeWriter;
import com.google.zxing.qrcode.decoder.ErrorCorrectionLevel;

@Component
public class QrCodeUtil {
    private static final Logger LOGGER = LoggerFactory.getLogger(QrCodeUtil.class);
    private static String path = EgovProperties.getProperty("Globals.qrFileStorePath");
    private static String logoImgPath = EgovProperties.getProperty("Globals.qrLogoPath");
    private static final int MAX_QR_DIMENSION = 2000;

    public static String createQrCode(String url, int width, int height) {
        if (url == null || url.isBlank()) {
            LOGGER.warn("QR 코드 생성 실패: URL이 null이거나 빈 문자열입니다.");
            return "";
        }
        if (width <= 0 || height <= 0) {
            LOGGER.warn("QR 코드 생성 실패: width와 height는 0보다 커야 합니다. width={}, height={}", width, height);
            return "";
        }
        if (width > MAX_QR_DIMENSION || height > MAX_QR_DIMENSION) {
            LOGGER.warn("QR 코드 생성 실패: width 또는 height가 허용 최대치를 초과했습니다. 최대={}, width={}, height={}", MAX_QR_DIMENSION,
                    width, height);
            return "";
        }

        try {
            BitMatrix bitMatrix = new MultiFormatWriter().encode(url, BarcodeFormat.QR_CODE, width, height);

            try (ByteArrayOutputStream outputStream = new ByteArrayOutputStream()) {
                MatrixToImageWriter.writeToStream(bitMatrix, "PNG", outputStream);
                byte[] imageBytes = outputStream.toByteArray();
                return Base64.getEncoder().encodeToString(imageBytes);
            }
        } catch (WriterException e) {
            LOGGER.error("createQrCode - WriterException. url={}, width={}, height={}", url, width, height, e);
            return "";
        } catch (NullPointerException e) {
            LOGGER.error("createQrCode - NullPointerException. url={}, width={}, height={}", url, width, height, e);
            return "";
        } catch (IllegalArgumentException e) {
            LOGGER.error("createQrCode - IllegalArgumentException. url={}, width={}, height={}", url, width, height, e);
            return "";
        } catch (IOException e) {
            LOGGER.error("createQrCode - IOException. url={}, width={}, height={}", url, width, height, e);
            return "";
        } catch (RuntimeException e) {
            LOGGER.error("createQrCode - RuntimeException. url={}, width={}, height={}", url, width, height, e);
            return "";
        } catch (Exception e) {
            LOGGER.error("createQrCode - Exception. url={}, width={}, height={}", url, width, height, e);
            return "";
        }
    }

    public static String createQrCodeInLogo (String url, int width, int height, String imgPath, String fileName) throws Exception {
        
        String logoPath = path+logoImgPath+"/logo.png";
        if(!imgPath.isEmpty() && !fileName.isEmpty()) {
            
            //로컬인경우 upload 추가
            if(!path.contains("bkshare")){
                logoPath = path+"/upload"+imgPath+"/"+fileName;
            } else {
                logoPath = path+imgPath+"/"+fileName;
            }
            
        }
        
        // QR 코드 생성
        Hashtable<EncodeHintType, Object> hintMap = new Hashtable<>();
        hintMap.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.L);
        hintMap.put(EncodeHintType.QR_VERSION, 10);
        
        BitMatrix bitMatrix = new MultiFormatWriter().encode(url, BarcodeFormat.QR_CODE, width, height);
        
        File logoImg = new File(logoPath);
        
        if(logoImg.exists()) {
            BufferedImage qrImage = MatrixToImageWriter.toBufferedImage(bitMatrix);
            BufferedImage logoImage = ImageIO.read(logoImg);

            if(qrImage != null && logoImage != null ) {
                //qr 비율에 맞게 로고 이미지 resize
                int qrWidth = qrImage.getWidth();
                int qrHeight = qrImage.getHeight();
        
                
                // 로고 이미지를 QR 코드 중앙에 삽입
                int logoWidth = logoImage.getWidth();
                int logoHeight = logoImage.getHeight();
                
                int newLogoWidth = 0;
                int newLogoHeight = 0;
        
                
                if(logoWidth <=110 && logoHeight <= 30) {
                    newLogoWidth = logoWidth;
                    newLogoHeight = logoHeight;
                } else {
                    newLogoWidth = (qrWidth/2)-60;
                    newLogoHeight = 0;
                    
                    if(logoWidth > newLogoWidth) {
                        newLogoHeight = (logoHeight * newLogoWidth) / logoWidth;
                    }
                }
                
                int posX = (qrWidth - newLogoWidth) / 2;
                int posY = (qrHeight - newLogoHeight) / 2;
                
                BufferedImage combined = new BufferedImage(qrImage.getHeight(), qrImage.getWidth(), BufferedImage.TYPE_INT_ARGB);
                
                Graphics2D graph = (Graphics2D)combined.getGraphics();
                graph.drawImage(qrImage, 0, 0, null);
                graph.setComposite(AlphaComposite.getInstance(AlphaComposite.SRC_OVER, 1f));
                graph.drawImage(logoImage, posX, posY, newLogoWidth, newLogoHeight, null);
                graph.dispose();
                
                String base64Image =  "";

                // QR 코드 이미지 출력;
                try (ByteArrayOutputStream outputStream = new ByteArrayOutputStream()) {
                    ImageIO.write(combined, "PNG", outputStream);
                    byte[] imageBytes = outputStream.toByteArray();
                    base64Image = Base64.getEncoder().encodeToString(imageBytes);
                } catch (NullPointerException e) {
                    LOGGER.error("createQrCodeInLogo - NullPointerException: {}", e.getMessage(), e);
                } catch (IllegalArgumentException e) {
                    LOGGER.error("createQrCodeInLogo - IllegalArgumentException: {}", e.getMessage(), e);
                } catch (IOException e) {
                    LOGGER.error("createQrCodeInLogo - IOException: {}", e.getMessage(), e);
                } catch (RuntimeException e) {
                    LOGGER.error("createQrCodeInLogo - RuntimeException: {}", e.getMessage(), e);
                } catch (Exception e) {
                    LOGGER.error("createQrCodeInLogo - Unexpected Exception: {}", e.getMessage(), e);
                }

                return base64Image;
            }
            return null;
        } else {
            throw new Exception("이미지가 없음");
        }
    }

    public static BufferedImage ImageResize(BufferedImage image, int width, int height) {
        BufferedImage outputImage = new BufferedImage(width, height, image.getType());
        
        Graphics2D graphics2D = outputImage.createGraphics();
        graphics2D.drawImage(outputImage, 0, 0, width, height, null);
        graphics2D.dispose();
        
        return outputImage;
        
    }
}
```
### QR Code 생성 소스 개선점점

첨부 소스는 `createQrCode()`와 `createQrCodeInLogo()`로 QR 이미지를 생성하고 Base64 문자열로 반환하는 구조입니다. 기본 QR 생성은 동작 가능성이 있지만, **로고 삽입 QR의 인식률 저하**, **설정 미적용**, **예외 처리 불명확**, **파일 경로 보안**, **민감 URL 로그 노출** 측면에서 실무 개선이 필요합니다.

### 1. 핵심 문제 요약

|구분|현재 문제|실무 영향|개선 방향|
|---|---|---|---|
|힌트 미적용|`hintMap` 생성 후 `encode()`에 전달하지 않음|오류 보정/QR 버전 설정이 실제 적용되지 않음|`encode(..., hints)` 사용|
|오류 보정|로고 삽입인데 `ErrorCorrectionLevel.L` 사용|로고가 QR 일부를 가리면 인식률 저하|로고 QR은 `Q` 또는 `H` 권장|
|입력 검증|`createQrCodeInLogo()`는 `url`, `imgPath`, `fileName`, 크기 검증 부족|NPE, 비정상 QR, 장애 원인 추적 어려움|공통 검증 메서드 적용|
|반환값|`""`, `null`, `throw Exception` 혼재|호출부에서 실패 원인 판단 어려움|일관되게 예외 또는 결과 객체 사용|
|로그|오류 로그에 `url` 원문 출력|QR URL에 토큰 포함 시 로그 유출 위험|URL 원문 로그 금지|
|파일 경로|`imgPath`, `fileName` 직접 문자열 결합|Path Traversal 가능성|`Path.resolve().normalize()` 사용|
|로고 크기|로고 크기 계산이 불안정|로고가 너무 크거나 높이 0 가능|QR 대비 비율 제한|
|이미지 생성|`BufferedImage(qrImage.getHeight(), qrImage.getWidth())`|정사각형이 아니면 가로/세로 뒤바뀜|`width, height` 순서 사용|
|Resize 버그|`drawImage(outputImage, ...)` 사용|원본 이미지가 아니라 빈 이미지 자체를 그림|`drawImage(image, ...)`로 수정|

### 2. 현재 소스의 주요 문제점

#### 2-1. `hintMap`이 실제 QR 생성에 사용되지 않음

현재 `createQrCodeInLogo()`에서는 아래처럼 힌트를 만들고 있습니다.

```java
Hashtable<EncodeHintType, Object> hintMap = new Hashtable<>();
hintMap.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.L);
hintMap.put(EncodeHintType.QR_VERSION, 10);
BitMatrix bitMatrix = new MultiFormatWriter().encode(url, BarcodeFormat.QR_CODE, width, height);
```

문제는 `hintMap`을 만들었지만 `encode()` 호출에 넘기지 않는다는 점입니다. 따라서 `ERROR_CORRECTION`, `QR_VERSION` 설정은 실제 QR 생성에 반영되지 않습니다.  
개선:

```java
BitMatrix bitMatrix = new MultiFormatWriter().encode(
        url,
        BarcodeFormat.QR_CODE,
        width,
        height,
        hintMap
);
```

#### 2-2. 로고 삽입 QR에 `ErrorCorrectionLevel.L`은 부적절

로고를 QR 중앙에 덮어씌우면 QR 코드의 일부 데이터 영역이 가려집니다. 그런데 현재 코드는 로고 삽입용 힌트에서 `ErrorCorrectionLevel.L`을 사용하고 있습니다.  
실무 권장:

|상황|권장 오류 보정|
|---|---|
|일반 QR|`M`|
|로고 없는 짧은 URL QR|`M`|
|로고 삽입 QR|`Q` 또는 `H`|
|인쇄물/오염 가능성 있음|`Q` 또는 `H`|
|단, 오류 보정 레벨을 높이면 QR 패턴이 복잡해질 수 있으므로 **QR에 담는 문자열은 짧게 유지**해야 합니다.||

#### 2-3. `QR_VERSION` 고정은 실무에서 권장하지 않음

현재 코드에는 `QR_VERSION = 10` 설정이 있습니다. 다만 실제로는 힌트가 적용되지 않고 있습니다.  
실무에서는 QR 버전을 고정하기보다 ZXing이 데이터 길이에 맞게 자동 선택하게 두는 편이 안전합니다. QR 버전을 고정하면 데이터가 길어졌을 때 생성 실패 또는 비효율적인 QR이 될 수 있습니다.  
권장:

```java
hints.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.H);
hints.put(EncodeHintType.CHARACTER_SET, "UTF-8");
hints.put(EncodeHintType.MARGIN, 2);
// QR_VERSION은 기본적으로 지정하지 않음
```

#### 2-4. `createQrCodeInLogo()` 입력값 검증 부족

`createQrCode()`에는 `url`, `width`, `height`, 최대 크기 검증이 있습니다. 반면 `createQrCodeInLogo()`에는 동일한 검증이 없습니다. 특히 아래 코드는 `imgPath` 또는 `fileName`이 `null`이면 바로 NPE가 발생합니다.

```java
if(!imgPath.isEmpty() && !fileName.isEmpty()) {
```

개선:

```java
if (imgPath != null && !imgPath.isBlank() && fileName != null && !fileName.isBlank()) {
```

또는 더 좋게는 공통 검증 메서드를 사용합니다.

#### 2-5. 실패 반환값이 일관되지 않음

현재 실패 시 반환 방식이 섞여 있습니다.

|메서드|실패 시 반환|
|---|---|
|`createQrCode()`|`""`|
|`createQrCodeInLogo()`|`null`, `""`, `throw Exception` 혼재|
|이 방식은 호출부에서 실패 원인을 알기 어렵습니다. 실무에서는 아래 중 하나로 통일하는 것이 좋습니다.||
|방식|권장도|
|---|---:|
|예외 발생|높음|
|결과 객체 반환|높음|
|빈 문자열 반환|낮음|
|권장:||

```java
throw new IllegalStateException("QR 코드 생성 실패", e);
```

#### 2-6. URL 원문 로그 출력은 위험

현재 `createQrCode()` 예외 로그에는 `url` 원문이 출력됩니다.

```java
LOGGER.error("createQrCode - WriterException. url={}, width={}, height={}", url, width, height, e);
```

QR URL에 주문 토큰, 쿠폰 토큰, 인증 토큰이 들어가면 로그에 민감 값이 남습니다.  
개선:

```java
LOGGER.error("QR 코드 생성 실패. width={}, height={}", width, height, e);
```

필요하면 URL 전체가 아니라 업무 ID, 요청 ID, 토큰 해시 일부만 기록합니다.

#### 2-7. 파일 경로 조립 방식이 위험

현재 로고 경로는 문자열 결합으로 만들어집니다.

```java
logoPath = path + "/upload" + imgPath + "/" + fileName;
```

이 방식은 `imgPath`나 `fileName`에 `../`가 들어가는 경우 의도하지 않은 파일 접근이 가능해질 수 있습니다.  
실무 개선:

```java
Path baseDir = Paths.get(path, "upload").toAbsolutePath().normalize();
Path logoFile = baseDir.resolve(imgPath).resolve(fileName).normalize();
if (!logoFile.startsWith(baseDir)) {
    throw new IllegalArgumentException("허용되지 않은 파일 경로입니다.");
}
```

#### 2-8. 로고 크기 계산 로직이 불안정

현재 로고 크기 계산은 아래 문제가 있습니다.

```java
newLogoWidth = (qrWidth / 2) - 60;
newLogoHeight = 0;
if (logoWidth > newLogoWidth) {
    newLogoHeight = (logoHeight * newLogoWidth) / logoWidth;
}
```

문제:

- QR 크기가 작으면 `newLogoWidth`가 0 이하가 될 수 있음
    
- `logoWidth <= newLogoWidth`인 경우 `newLogoHeight`가 0으로 남을 수 있음
    
- 로고가 QR의 너무 큰 영역을 가릴 수 있음
    
- 로고 뒤 흰색 배경이 없어 QR 패턴과 로고가 섞일 수 있음  
    실무 권장:
    

```text
로고 가로/세로는 QR 전체 크기의 약 15~20% 이내로 제한
로고 뒤에는 흰색 배경을 먼저 그림
오류 보정 레벨은 Q 또는 H 사용
```

#### 2-9. `BufferedImage` 생성 시 width/height 순서 오류 가능

현재 코드:

```java
BufferedImage combined = new BufferedImage(qrImage.getHeight(), qrImage.getWidth(), BufferedImage.TYPE_INT_ARGB);
```

`BufferedImage` 생성자는 `(width, height, type)` 순서입니다. QR을 항상 정사각형으로 만들면 문제를 못 느낄 수 있지만, 가로/세로가 다른 값으로 들어오면 이미지 크기가 뒤바뀝니다.  
개선:

```java
BufferedImage combined = new BufferedImage(qrImage.getWidth(), qrImage.getHeight(), BufferedImage.TYPE_INT_ARGB);
```

#### 2-10. `ImageResize()` 메서드에 명확한 버그 존재

현재 코드:

```java
graphics2D.drawImage(outputImage, 0, 0, width, height, null);
```

여기서는 원본 `image`가 아니라 새로 만든 `outputImage`를 자기 자신에게 그리고 있습니다. 따라서 정상적인 리사이즈가 아닙니다.  
수정:

```java
graphics2D.drawImage(image, 0, 0, width, height, null);
```

### 3. 최소 수정안

현재 구조를 크게 바꾸지 않고 당장 안정성을 높이려면 아래 정도는 반영하는 것이 좋습니다.

```java
Map<EncodeHintType, Object> hints = new EnumMap<>(EncodeHintType.class);
hints.put(EncodeHintType.CHARACTER_SET, StandardCharsets.UTF_8.name());
hints.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.H);
hints.put(EncodeHintType.MARGIN, 2);
BitMatrix bitMatrix = new MultiFormatWriter().encode(
        url,
        BarcodeFormat.QR_CODE,
        width,
        height,
        hints
);
```

그리고 `createQrCodeInLogo()` 시작 부분에 다음 검증을 추가합니다.

```java
if (url == null || url.isBlank()) {
    throw new IllegalArgumentException("QR URL은 필수입니다.");
}
if (width <= 0 || height <= 0 || width > MAX_QR_DIMENSION || height > MAX_QR_DIMENSION) {
    throw new IllegalArgumentException("QR 이미지 크기가 허용 범위를 벗어났습니다.");
}
if (width != height) {
    throw new IllegalArgumentException("QR 이미지는 정사각형 크기를 권장합니다.");
}
```

### 4. 실무 개선 예제

아래 예제는 Java 11, Spring Framework 5.3 환경에서 사용할 수 있는 형태입니다. 핵심은 **입력 검증**, **UTF-8**, **오류 보정 레벨**, **로고 크기 제한**, **경로 검증**, **일관된 예외 처리**입니다.

```java
package sample.util;
import com.google.zxing.BarcodeFormat;
import com.google.zxing.EncodeHintType;
import com.google.zxing.MultiFormatWriter;
import com.google.zxing.client.j2se.MatrixToImageWriter;
import com.google.zxing.common.BitMatrix;
import com.google.zxing.qrcode.decoder.ErrorCorrectionLevel;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import javax.imageio.ImageIO;
import java.awt.AlphaComposite;
import java.awt.Color;
import java.awt.Graphics2D;
import java.awt.RenderingHints;
import java.awt.image.BufferedImage;
import java.io.ByteArrayOutputStream;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Base64;
import java.util.EnumMap;
import java.util.Map;
public final class QrCodeUtil {
    private static final Logger LOGGER = LoggerFactory.getLogger(QrCodeUtil.class);
    private static final int MAX_QR_DIMENSION = 2000;
    private static final int MIN_QR_DIMENSION = 100;
    private QrCodeUtil() {
    }
    public static String createQrBase64(String contents, int width, int height) {
        validateContentsAndSize(contents, width, height);
        try {
            BitMatrix matrix = createQrMatrix(contents, width, height, ErrorCorrectionLevel.M);
            try (ByteArrayOutputStream out = new ByteArrayOutputStream()) {
                MatrixToImageWriter.writeToStream(matrix, "PNG", out);
                return Base64.getEncoder().encodeToString(out.toByteArray());
            }
        } catch (Exception e) {
            LOGGER.error("QR 코드 생성 실패. width={}, height={}", width, height, e);
            throw new IllegalStateException("QR 코드 생성 실패", e);
        }
    }
    public static String createQrWithLogoBase64(String contents, int size, Path logoFile) {
        validateContentsAndSize(contents, size, size);
        validateLogoFile(logoFile);
        try {
            BitMatrix matrix = createQrMatrix(contents, size, size, ErrorCorrectionLevel.H);
            BufferedImage qrImage = MatrixToImageWriter.toBufferedImage(matrix);
            BufferedImage logoImage = ImageIO.read(logoFile.toFile());
            if (logoImage == null) {
                throw new IllegalArgumentException("지원하지 않는 로고 이미지 형식입니다.");
            }
            BufferedImage combined = drawLogo(qrImage, logoImage);
            try (ByteArrayOutputStream out = new ByteArrayOutputStream()) {
                ImageIO.write(combined, "PNG", out);
                return Base64.getEncoder().encodeToString(out.toByteArray());
            }
        } catch (Exception e) {
            LOGGER.error("로고 포함 QR 코드 생성 실패. size={}, logoFile={}", size, safeFileName(logoFile), e);
            throw new IllegalStateException("로고 포함 QR 코드 생성 실패", e);
        }
    }
    private static BitMatrix createQrMatrix(String contents, int width, int height, ErrorCorrectionLevel level) throws Exception {
        Map<EncodeHintType, Object> hints = new EnumMap<>(EncodeHintType.class);
        hints.put(EncodeHintType.CHARACTER_SET, StandardCharsets.UTF_8.name());
        hints.put(EncodeHintType.ERROR_CORRECTION, level);
        hints.put(EncodeHintType.MARGIN, 2);
        return new MultiFormatWriter().encode(
                contents,
                BarcodeFormat.QR_CODE,
                width,
                height,
                hints
        );
    }
    private static BufferedImage drawLogo(BufferedImage qrImage, BufferedImage logoImage) {
        int qrWidth = qrImage.getWidth();
        int qrHeight = qrImage.getHeight();
        int maxLogoWidth = qrWidth / 5;
        int maxLogoHeight = qrHeight / 5;
        int logoWidth = logoImage.getWidth();
        int logoHeight = logoImage.getHeight();
        double scale = Math.min(
                (double) maxLogoWidth / logoWidth,
                (double) maxLogoHeight / logoHeight
        );
        int resizedLogoWidth = Math.max(1, (int) Math.round(logoWidth * scale));
        int resizedLogoHeight = Math.max(1, (int) Math.round(logoHeight * scale));
        int posX = (qrWidth - resizedLogoWidth) / 2;
        int posY = (qrHeight - resizedLogoHeight) / 2;
        int padding = Math.max(4, qrWidth / 50);
        BufferedImage combined = new BufferedImage(qrWidth, qrHeight, BufferedImage.TYPE_INT_ARGB);
        Graphics2D g = combined.createGraphics();
        try {
            g.drawImage(qrImage, 0, 0, null);
            g.setRenderingHint(RenderingHints.KEY_INTERPOLATION, RenderingHints.VALUE_INTERPOLATION_BILINEAR);
            g.setComposite(AlphaComposite.getInstance(AlphaComposite.SRC_OVER, 1f));
            g.setColor(Color.WHITE);
            g.fillRect(
                    posX - padding,
                    posY - padding,
                    resizedLogoWidth + padding * 2,
                    resizedLogoHeight + padding * 2
            );
            g.drawImage(logoImage, posX, posY, resizedLogoWidth, resizedLogoHeight, null);
        } finally {
            g.dispose();
        }
        return combined;
    }
    private static void validateContentsAndSize(String contents, int width, int height) {
        if (contents == null || contents.isBlank()) {
            throw new IllegalArgumentException("QR 내용은 필수입니다.");
        }
        if (contents.length() > 1000) {
            throw new IllegalArgumentException("QR 내용이 너무 깁니다. 짧은 URL 또는 토큰 사용을 권장합니다.");
        }
        if (width < MIN_QR_DIMENSION || height < MIN_QR_DIMENSION) {
            throw new IllegalArgumentException("QR 크기는 최소 " + MIN_QR_DIMENSION + "px 이상이어야 합니다.");
        }
        if (width > MAX_QR_DIMENSION || height > MAX_QR_DIMENSION) {
            throw new IllegalArgumentException("QR 크기가 허용 최대값을 초과했습니다.");
        }
    }
    private static void validateLogoFile(Path logoFile) {
        if (logoFile == null) {
            throw new IllegalArgumentException("로고 파일 경로는 필수입니다.");
        }
        if (!Files.isRegularFile(logoFile)) {
            throw new IllegalArgumentException("로고 파일이 존재하지 않습니다.");
        }
    }
    private static String safeFileName(Path path) {
        return path == null || path.getFileName() == null ? "" : path.getFileName().toString();
    }
}
```

### 5. 파일 경로 보안 개선 예제

사용자가 업로드한 로고 또는 서버에 저장된 로고를 사용할 경우, 문자열 결합 대신 `Path` 기반으로 검증해야 합니다.

```java
public static Path resolveLogoPath(Path baseDir, String imgPath, String fileName) {
    if (baseDir == null) {
        throw new IllegalArgumentException("기준 경로는 필수입니다.");
    }
    if (imgPath == null || imgPath.isBlank()) {
        throw new IllegalArgumentException("이미지 경로는 필수입니다.");
    }
    if (fileName == null || fileName.isBlank()) {
        throw new IllegalArgumentException("파일명은 필수입니다.");
    }
    if (!fileName.endsWith(".png") && !fileName.endsWith(".jpg") && !fileName.endsWith(".jpeg")) {
        throw new IllegalArgumentException("허용되지 않은 로고 파일 형식입니다.");
    }
    Path normalizedBase = baseDir.toAbsolutePath().normalize();
    Path resolved = normalizedBase.resolve(imgPath).resolve(fileName).normalize();
    if (!resolved.startsWith(normalizedBase)) {
        throw new IllegalArgumentException("허용되지 않은 파일 경로입니다.");
    }
    return resolved;
}
```

### 6. QR에 담는 값 개선

현재 유틸은 전달받은 `url`을 그대로 QR에 넣습니다. 실무에서는 QR에 업무 데이터를 직접 넣지 말고 **짧은 토큰 URL**을 넣는 것이 안전합니다.

```text
비권장:
https://example.com/order/pickup?orderId=12345&userId=1001&phone=010...
권장:
https://example.com/qr/pickup?t=eyJhbGci...
더 권장:
https://example.com/qr/pickup?t=난수토큰
```

권장 서버 검증:

```text
1. QR 토큰 수신
2. 토큰 해시 조회
3. 만료 시간 확인
4. 사용 여부 확인
5. 주문/쿠폰/픽업 상태 확인
6. USED 처리
7. 업무 처리
```

### 7. Spring Controller 반환 방식 개선

Base64 문자열만 반환할지, PNG 바이트를 직접 반환할지 명확히 정해야 합니다. 웹 화면에서 `<img>`에 직접 넣을 목적이면 Base64도 가능하지만, API/이미지 응답으로는 `byte[]`가 더 단순합니다.

#### PNG 직접 응답 권장 예

```java
@GetMapping(value = "/qr/pickup.png", produces = MediaType.IMAGE_PNG_VALUE)
public ResponseEntity<byte[]> qr() {
    String qrUrl = "https://example.com/qr/pickup?t=random-token";
    byte[] png = QrImageService.createQrPng(qrUrl, 300, 300);
    return ResponseEntity.ok()
            .contentType(MediaType.IMAGE_PNG)
            .cacheControl(CacheControl.noStore())
            .header(HttpHeaders.PRAGMA, "no-cache")
            .header(HttpHeaders.EXPIRES, "0")
            .body(png);
}
```

개인화 QR, 주문 QR, 쿠폰 QR은 `Cache-Control: no-store`를 권장합니다.

### 8. 최종 개선 우선순위

|순위|개선 항목|이유|
|--:|---|---|
|1|`hintMap`을 `encode()`에 전달|현재 설정이 적용되지 않는 핵심 버그|
|2|로고 QR 오류 보정 `H` 또는 `Q` 적용|스캔 인식률 개선|
|3|`createQrCodeInLogo()` 입력값 검증 추가|NPE/비정상 QR 방지|
|4|URL 원문 로그 제거|토큰/개인정보 로그 유출 방지|
|5|파일 경로 `Path.normalize()` 검증|Path Traversal 방지|
|6|반환값 정책 통일|호출부 장애 처리 명확화|
|7|로고 크기 15~20% 제한|QR 패턴 훼손 최소화|
|8|`ImageResize()` 버그 수정 또는 삭제|오동작 코드 제거|
|9|`@Component + static` 구조 정리|테스트/설정 주입 개선|
|10|QR 내용 토큰화|업무 보안 강화|

### 9. 결론

현재 소스는 **단순 QR Base64 생성**은 가능하지만, 로고 포함 QR은 `hintMap` 미적용, 낮은 오류 보정 레벨, 로고 크기 계산 불안정 때문에 실무 인식률 문제가 발생할 수 있습니다. 또한 QR URL 원문 로그, 문자열 기반 파일 경로 조립, 실패 반환값 혼재는 운영 보안과 장애 대응 측면에서 개선이 필요합니다. 우선은 **힌트 적용**, **오류 보정 `H`**, **입력 검증**, **URL 로그 제거**, **경로 검증**, **반환 정책 통일**부터 반영하는 것이 가장 효과적입니다.

## BitMatrix, hintMap

### BitMatrix와 hintMap 상세 설명

#### 출력 전 신뢰성 검증 결과

|검증 항목|확인 결과|
|---|---|
|공식 Javadoc 확인|`QRCodeWriter#encode()`는 `BitMatrix`를 반환하며, `hints` 인자를 받는 overload가 존재함을 확인했습니다. ([ZXing](https://zxing.github.io/zxing/apidocs/com/google/zxing/qrcode/QRCodeWriter.html "QRCodeWriter (ZXing 3.5.4 API)"))|
|공식 소스 확인|`QRCodeWriter#encode(contents, format, width, height)`는 내부적으로 `encode(..., null)`을 호출합니다. 즉, 4개 인자 메서드를 쓰면 hint가 전달되지 않습니다. ([GitHub](https://github.com/zxing/zxing/blob/master/core/src/main/java/com/google/zxing/qrcode/QRCodeWriter.java "zxing/core/src/main/java/com/google/zxing/qrcode/QRCodeWriter.java at master · zxing/zxing · GitHub"))|
|기본값 확인|ZXing `QRCodeWriter` 소스상 기본 오류 보정 레벨은 `L`, 기본 Quiet Zone 값은 `4`입니다. ([GitHub](https://github.com/zxing/zxing/blob/master/core/src/main/java/com/google/zxing/qrcode/QRCodeWriter.java "zxing/core/src/main/java/com/google/zxing/qrcode/QRCodeWriter.java at master · zxing/zxing · GitHub"))|
|첨부 코드 확인|첨부 코드에서는 `hintMap`을 만들고 `ERROR_CORRECTION`, `QR_VERSION`을 넣었지만, 실제 `encode()` 호출에는 `hintMap`을 전달하지 않습니다. 따라서 해당 설정은 현재 코드에서 적용되지 않습니다.|

### 1. BitMatrix란?

`BitMatrix`는 ZXing에서 QR 코드 결과를 표현하는 **2차원 비트 행렬**입니다. 공식 Javadoc 기준으로 `BitMatrix`는 2D matrix of bits이며, 좌표는 `x, y` 순서이고 원점은 좌상단입니다. ([ZXing](https://zxing.github.io/zxing/apidocs/com/google/zxing/common/BitMatrix.html?utm_source=chatgpt.com "BitMatrix (ZXing 3.5.4 API)"))  
쉽게 말하면 다음과 같습니다.

```text
BitMatrix = QR 코드의 검은 칸/흰 칸 정보를 가진 2차원 배열
true  = 검은색 영역
false = 흰색 영역
```

QR 생성 흐름에서는 보통 아래 순서로 사용됩니다.

```text
문자열 URL/토큰
→ MultiFormatWriter 또는 QRCodeWriter
→ BitMatrix
→ MatrixToImageWriter
→ PNG/JPG/BufferedImage/Base64
```

첨부 코드에서도 `MultiFormatWriter().encode(...)` 결과를 `BitMatrix`로 받은 뒤, `MatrixToImageWriter.writeToStream(...)` 또는 `MatrixToImageWriter.toBufferedImage(...)`로 이미지화하고 있습니다.

### 2. BitMatrix가 실제로 하는 역할

| 단계             | 역할                                                                      |
| -------------- | ----------------------------------------------------------------------- |
| QR 인코딩 결과 보관   | URL 문자열을 QR 패턴으로 변환한 결과를 보관                                             |
| 검은색/흰색 정보 제공   | 각 좌표의 칸이 검정인지 흰색인지 표현                                                   |
| 이미지 변환 전 중간 결과 | PNG, JPG, `BufferedImage`로 바꾸기 전 원본 데이터 역할                              |
| 로고 합성 기준 이미지   | `MatrixToImageWriter.toBufferedImage(bitMatrix)`로 QR 이미지를 만든 뒤 로고 합성 가능 |

- 즉, `BitMatrix` 자체는 이미지 파일이 아닙니다. 이미지 파일로 저장하거나 브라우저에 응답하려면 반드시 `MatrixToImageWriter`, `ImageIO`, 직접 픽셀 변환 등의 후처리가 필요합니다.
### 3. 현재 코드에서 BitMatrix 사용 부분

첨부 코드의 기본 QR 생성은 아래 구조입니다.

```java
BitMatrix bitMatrix = new MultiFormatWriter().encode(url, BarcodeFormat.QR_CODE, width, height);
MatrixToImageWriter.writeToStream(bitMatrix, "PNG", outputStream);
```

로고 포함 QR 생성도 먼저 `BitMatrix`를 만들고, 이를 `BufferedImage`로 바꾼 뒤 로고를 덮어씌우는 구조입니다.

```java
BitMatrix bitMatrix = new MultiFormatWriter().encode(url, BarcodeFormat.QR_CODE, width, height);
BufferedImage qrImage = MatrixToImageWriter.toBufferedImage(bitMatrix);
```

문제는 로고 포함 QR 코드에서 아래처럼 `hintMap`을 만들었지만 실제 QR 생성에는 사용하지 않았다는 점입니다.

```java
Hashtable<EncodeHintType, Object> hintMap = new Hashtable<>();
hintMap.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.L);
hintMap.put(EncodeHintType.QR_VERSION, 10);
BitMatrix bitMatrix = new MultiFormatWriter().encode(url, BarcodeFormat.QR_CODE, width, height);
```

이 경우 `hintMap`은 메모리에만 존재하고, QR 생성 결과에는 아무 영향도 주지 않습니다.

### 4. hintMap이란?

`hintMap`은 ZXing 인코더에게 전달하는 **추가 생성 옵션 Map**입니다.  
정식 타입은 보통 다음처럼 작성합니다.

```java
Map<EncodeHintType, Object> hints = new EnumMap<>(EncodeHintType.class);
```

또는 기존 코드처럼 아래처럼 작성할 수도 있습니다.

```java
Hashtable<EncodeHintType, Object> hintMap = new Hashtable<>();
```

하지만 실무에서는 `Hashtable`보다 `EnumMap`을 권장합니다.

| 구분         | Hashtable | EnumMap |
| ---------- | --------- | ------- |
| 사용 가능 여부   | 가능        | 가능      |
| 동기화        | 기본 동기화    |         |
| 키 타입       | 일반 객체     |         |
| Enum 키 최적화 | 아님        | 예       |
| 실무 권장      | 낮음        | 높음      |

- `EncodeHintType`은 enum이므로 `EnumMap`이 의미상 더 적절합니다. 메서드 내부에서 일회성으로 만들고 바로 넘기는 설정값이라면 `Hashtable`의 동기화 장점도 거의 없습니다.
### 5. hintMap이 적용되는 정확한 위치

ZXing의 `QRCodeWriter`에는 두 가지 주요 `encode()`가 있습니다.

```java
encode(String contents, BarcodeFormat format, int width, int height)
encode(String contents, BarcodeFormat format, int width, int height, Map<EncodeHintType, ?> hints)
```

공식 Javadoc에서도 4개 인자 버전은 기본 설정으로 인코딩하고, 5개 인자 버전은 추가 파라미터인 `hints`를 받을 수 있다고 설명합니다. ([ZXing](https://zxing.github.io/zxing/apidocs/com/google/zxing/qrcode/QRCodeWriter.html "QRCodeWriter (ZXing 3.5.4 API)"))  
따라서 hint를 적용하려면 반드시 아래처럼 호출해야 합니다.

```java
BitMatrix bitMatrix = new MultiFormatWriter().encode(
        url,
        BarcodeFormat.QR_CODE,
        width,
        height,
        hints
);
```

현재 첨부 코드처럼 아래 호출을 사용하면 hint가 적용되지 않습니다.

```java
BitMatrix bitMatrix = new MultiFormatWriter().encode(
        url,
        BarcodeFormat.QR_CODE,
        width,
        height
);
```

### 6. QRCodeWriter 내부 동작 기준 핵심

ZXing `QRCodeWriter` 소스 기준으로 4개 인자 `encode()`는 바로 5개 인자 `encode(..., null)`을 호출합니다. 이후 `hints`가 존재할 때만 `ERROR_CORRECTION`, `MARGIN`을 읽습니다. 기본값은 `ErrorCorrectionLevel.L`, Quiet Zone `4`입니다. ([GitHub](https://github.com/zxing/zxing/blob/master/core/src/main/java/com/google/zxing/qrcode/QRCodeWriter.java "zxing/core/src/main/java/com/google/zxing/qrcode/QRCodeWriter.java at master · zxing/zxing · GitHub"))  
정리하면 다음과 같습니다.

```text
4개 인자 encode()
→ hints = null
→ 기본 오류 보정 L
→ 기본 margin/quiet zone 4
```

```text
5개 인자 encode(..., hints)
→ ERROR_CORRECTION 있으면 적용
→ MARGIN 있으면 적용
→ Encoder.encode(contents, errorCorrectionLevel, hints)에 hints 전달
```

즉, `hintMap`은 **만드는 것만으로는 의미가 없고**, 반드시 `encode(..., hints)`에 전달되어야 합니다.

### 7. 주요 hintMap 설정값

#### 7-1. `EncodeHintType.CHARACTER_SET`

문자 인코딩을 지정합니다. 공식 Javadoc에서도 `CHARACTER_SET`은 적용 가능한 경우 사용할 문자 인코딩을 지정하는 값이며 타입은 `String`이라고 설명합니다. ([ZXing](https://zxing.github.io/zxing/apidocs/com/google/zxing/EncodeHintType.html "EncodeHintType (ZXing 3.5.4 API)"))

```java
hints.put(EncodeHintType.CHARACTER_SET, StandardCharsets.UTF_8.name());
```

실무 권장:

```text
한글, 특수문자, 다국어 URL 가능성이 있으면 UTF-8 명시 권장
```

주의:

```text
QR에 넣는 값이 URL이면 URL 자체를 먼저 정상 인코딩해야 함
예: query string에 한글, &, =, 공백이 들어가는 경우
```

#### 7-2. `EncodeHintType.ERROR_CORRECTION`

QR 코드의 오류 보정 수준을 지정합니다. 공식 Javadoc 기준으로 QR Code에서는 `ErrorCorrectionLevel` 타입을 사용합니다. ([ZXing](https://zxing.github.io/zxing/apidocs/com/google/zxing/EncodeHintType.html "EncodeHintType (ZXing 3.5.4 API)"))

```java
hints.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.H);
```

ZXing 공식 소스의 `ErrorCorrectionLevel`은 QR 표준의 네 가지 오류 보정 레벨을 캡슐화하며, 각 레벨은 대략 `L=7%`, `M=15%`, `Q=25%`, `H=30%` 보정 수준으로 주석 처리되어 있습니다. ([GitHub](https://github.com/zxing/zxing/blob/master/core/src/main/java/com/google/zxing/qrcode/decoder/ErrorCorrectionLevel.java?utm_source=chatgpt.com "zxing/core/src/main/java/com/google/zxing/qrcode/decoder ..."))

| 레벨  | 보정 수준 | 데이터 수용량 | 권장 상황                  |
| --- | ----: | ------: | ---------------------- |
| `L` |  약 7% |    가장 큼 | 로고 없는 단순 QR, 깨끗한 화면    |
| `M` | 약 15% |       큼 | 일반 웹/모바일 QR 기본값        |
| `Q` | 약 25% |     작아짐 | 인쇄물, 약간의 훼손 가능성        |
| `H` | 약 30% |   가장 작음 | 로고 삽입 QR, 훼손 가능성 높은 QR |

- 첨부 코드에서는 로고를 QR 중앙에 삽입하면서 `ErrorCorrectionLevel.L`을 설정하려고 했습니다. 하지만 로고가 QR 일부를 가리기 때문에 `L`은 실무적으로 낮습니다. 더구나 현재는 `hintMap`이 전달되지 않아 `L` 설정조차 코드상 명시 효과가 없습니다.
#### 7-3. `EncodeHintType.MARGIN`

QR 주변 여백, 즉 Quiet Zone을 지정합니다. 공식 `EncodeHintType` 소스에서 `MARGIN`은 바코드 생성 시 사용할 margin을 지정하며, 값 타입은 `Integer` 또는 정수 문자열이라고 설명합니다. ([GitHub](https://github.com/zxing/zxing/blob/master/core/src/main/java/com/google/zxing/EncodeHintType.java "zxing/core/src/main/java/com/google/zxing/EncodeHintType.java at master · zxing/zxing · GitHub"))

```java
hints.put(EncodeHintType.MARGIN, 2);
```

실무 판단:

|   값 | 판단                    |
| --: | --------------------- |
| `0` | 비권장. 스캐너 인식 실패 가능성 증가 |
| `1` | 공간이 매우 부족한 경우만 검토     |
| `2` | 웹 화면용에서 자주 사용하는 절충값   |
| `4` | ZXing 기본값, 인식 안정성 우선  |

- 로고가 들어가거나 인쇄물로 나갈 QR이면 margin을 너무 줄이지 않는 편이 안전합니다.
#### 7-4. `EncodeHintType.QR_VERSION`

QR 코드 버전을 강제로 지정합니다. 공식 소스에서 `QR_VERSION`은 “exact version of QR code to be encoded”를 지정하며 타입은 `Integer` 또는 정수 문자열입니다. ([GitHub](https://github.com/zxing/zxing/blob/master/core/src/main/java/com/google/zxing/EncodeHintType.java "zxing/core/src/main/java/com/google/zxing/EncodeHintType.java at master · zxing/zxing · GitHub"))

```java
hints.put(EncodeHintType.QR_VERSION, 10);
```

다만 실무에서는 보통 권장하지 않습니다.

| 판단       | 설명                                     |
| -------- | -------------------------------------- |
| 자동 선택 권장 | ZXing이 데이터 길이에 맞는 QR 버전을 선택하게 두는 편이 안전 |
| 고정 사용 주의 | 데이터가 길어지면 생성 실패 또는 비효율적인 QR 발생 가능      |
| 사용 가능 상황 | 물리 출력 크기, 디자인 가이드, 테스트 기준이 엄격히 고정된 경우  |

- 첨부 코드의 `QR_VERSION = 10`은 현재 전달되지 않아 적용되지 않고 있습니다. 설령 전달되더라도 특별한 이유가 없다면 제거하는 편이 낫습니다.
#### 7-5. `EncodeHintType.QR_MASK_PATTERN`

QR 마스크 패턴을 강제로 지정합니다. 공식 소스에 따르면 허용값은 `0..QRCode.NUM_MASK_PATTERNS-1`이고, 기본적으로는 최적 마스크 패턴을 자동 선택합니다. ([GitHub](https://github.com/zxing/zxing/blob/master/core/src/main/java/com/google/zxing/EncodeHintType.java "zxing/core/src/main/java/com/google/zxing/EncodeHintType.java at master · zxing/zxing · GitHub"))

```java
hints.put(EncodeHintType.QR_MASK_PATTERN, 3);
```

실무에서는 거의 설정하지 않는 편이 좋습니다.

```text
권장: 자동 선택
비권장: 특별한 검증 없이 QR_MASK_PATTERN 고정
```

마스크 패턴은 QR의 검정/흰색 분포를 조정해 스캔성을 높이는 내부 알고리즘 영역에 가깝습니다. 강제로 고정하면 오히려 품질이 나빠질 수 있습니다.

#### 7-6. `EncodeHintType.QR_COMPACT`

QR compact mode 사용 여부입니다. 공식 소스에서는 `QR_COMPACT`가 QR code compact mode 사용 여부를 지정하며, 타입은 `Boolean` 또는 문자열이라고 설명합니다. 또한 compact 수행 시 비 ISO-8859-1 문자는 가장 작은 문자 인코딩을 선택할 수 있고, 일부 스캐너는 특정 인코딩을 지원하지 않을 수 있어 `CHARACTER_SET`으로 UTF-8을 강제할 수 있다고 설명합니다. ([GitHub](https://github.com/zxing/zxing/blob/master/core/src/main/java/com/google/zxing/EncodeHintType.java "zxing/core/src/main/java/com/google/zxing/EncodeHintType.java at master · zxing/zxing · GitHub"))  
실무에서는 일반 URL QR 생성 시 굳이 사용하지 않는 편이 안전합니다.

### 8. 실무 권장 hintMap 설정

#### 일반 QR

```java
Map<EncodeHintType, Object> hints = new EnumMap<>(EncodeHintType.class);
hints.put(EncodeHintType.CHARACTER_SET, StandardCharsets.UTF_8.name());
hints.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.M);
hints.put(EncodeHintType.MARGIN, 2);
BitMatrix bitMatrix = new MultiFormatWriter().encode(
        contents,
        BarcodeFormat.QR_CODE,
        width,
        height,
        hints
);
```

#### 로고 포함 QR

```java
Map<EncodeHintType, Object> hints = new EnumMap<>(EncodeHintType.class);
hints.put(EncodeHintType.CHARACTER_SET, StandardCharsets.UTF_8.name());
hints.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.H);
hints.put(EncodeHintType.MARGIN, 2);
BitMatrix bitMatrix = new MultiFormatWriter().encode(
        contents,
        BarcodeFormat.QR_CODE,
        size,
        size,
        hints
);
```

#### 설정 판단표

|용도|`CHARACTER_SET`|`ERROR_CORRECTION`|`MARGIN`|`QR_VERSION`|
|---|---|---|--:|---|
|일반 URL QR|`UTF-8`|`M`|`2~4`|미지정|
|주문/픽업 QR|`UTF-8`|`M` 또는 `Q`|`2~4`|미지정|
|쿠폰 QR|`UTF-8`|`M` 또는 `Q`|`2~4`|미지정|
|로고 포함 QR|`UTF-8`|`H`|`2~4`|미지정|
|인쇄물 QR|`UTF-8`|`Q` 또는 `H`|`4` 권장|특수한 경우만|

### 9. 첨부 코드 기준 수정 포인트

현재 코드:

```java
Hashtable<EncodeHintType, Object> hintMap = new Hashtable<>();
hintMap.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.L);
hintMap.put(EncodeHintType.QR_VERSION, 10);
BitMatrix bitMatrix = new MultiFormatWriter().encode(url, BarcodeFormat.QR_CODE, width, height);
```

최소 수정:

```java
Map<EncodeHintType, Object> hints = new EnumMap<>(EncodeHintType.class);
hints.put(EncodeHintType.CHARACTER_SET, StandardCharsets.UTF_8.name());
hints.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.H);
hints.put(EncodeHintType.MARGIN, 2);
BitMatrix bitMatrix = new MultiFormatWriter().encode(
        url,
        BarcodeFormat.QR_CODE,
        width,
        height,
        hints
);
```

변경 이유:

|변경|이유|
|---|---|
|`Hashtable` → `EnumMap`|`EncodeHintType`이 enum이므로 더 적합|
|`CHARACTER_SET` 추가|한글/다국어/특수문자 안정성 확보|
|`ERROR_CORRECTION.L` → `H`|로고 삽입으로 가려지는 영역 보정|
|`QR_VERSION` 제거|자동 선택이 일반적으로 안전|
|`encode(..., hints)` 사용|설정값 실제 적용|

### 10. BitMatrix와 이미지 크기 관련 주의점

`width`, `height`는 최종 QR 이미지로 렌더링할 선호 크기입니다. `QRCodeWriter` 공식 Javadoc은 `renderResult()`가 QRCode를 `BitMatrix`로 렌더링하고 지정된 크기에 맞게 스케일링한다고 설명합니다. ([ZXing](https://zxing.github.io/zxing/apidocs/com/google/zxing/qrcode/QRCodeWriter.html "QRCodeWriter (ZXing 3.5.4 API)"))  
주의할 점:

```text
- QR은 가능하면 정사각형 크기 사용
- 너무 작은 크기에서는 스캐너 인식률 저하
- 로고 삽입 시 QR 크기 300px 이상 권장
- margin을 0으로 줄이면 인식률 저하 가능
- Base64로 반환하면 원본 PNG보다 문자열 크기가 커짐
```

### 11. 최종 정리

`BitMatrix`는 ZXing이 생성한 QR 패턴의 2차원 비트 결과물입니다. 이 결과를 `MatrixToImageWriter`나 `ImageIO`로 PNG/BufferedImage/Base64로 바꿔야 실제 화면이나 파일에서 사용할 수 있습니다.  
`hintMap`은 QR 생성 옵션입니다. 하지만 **반드시 `encode(..., hints)`에 전달해야 적용**됩니다. 첨부 코드에서는 `hintMap`을 만들고도 전달하지 않아 `ERROR_CORRECTION`, `QR_VERSION` 설정이 적용되지 않습니다. 실무에서는 일반 QR은 `UTF-8 + ErrorCorrectionLevel.M + MARGIN 2~4`, 로고 포함 QR은 `UTF-8 + ErrorCorrectionLevel.H + MARGIN 2~4`를 권장하며, `QR_VERSION`, `QR_MASK_PATTERN`은 특별한 검증 사유가 없으면 자동 선택에 맡기는 것이 안전합니다.

# 4. NegativeArraySizeException
### NegativeArraySizeException 발생 가능 경우

`NegativeArraySizeException`은 Java에서 **음수 크기의 배열을 생성하려고 할 때 발생하는 RuntimeException**입니다. Oracle JDK 11 공식 문서도 이 예외를 “negative size 배열 생성 시 발생”한다고 정의합니다. ([Oracle Docs](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/NegativeArraySizeException.html "NegativeArraySizeException (Java SE 11 & JDK 11 )"))

```java
int size = -1;
byte[] arr = new byte[size]; // NegativeArraySizeException
```

### 현재 QR 코드 소스에서의 판단

첨부된 QR 코드 소스 안에는 `new byte[-1]`, `new int[-1]`처럼 **직접 음수 배열을 생성하는 코드는 없습니다**. 따라서 현재 코드에서 `NegativeArraySizeException`이 난다면, 대부분은 ZXing, AWT/ImageIO, Base64, 내부 이미지 버퍼 생성 과정에서 **전달된 크기값 또는 계산된 버퍼 크기가 음수가 되는 경우**를 의심해야 합니다. 첨부 코드에서는 기본 QR 생성 메서드 `createQrCode()`에는 `width <= 0 || height <= 0` 검증이 있지만, 로고 포함 메서드 `createQrCodeInLogo()`에는 동일한 크기 검증이 없습니다.

### 발생 가능성이 높은 케이스

|케이스|발생 조건|현재 코드 관련성|판단|
|---|---|---|---|
|직접 배열 생성|`new byte[negative]`, `new int[negative]`|직접 코드에는 없음|낮음|
|QR width/height 음수|`createQrCodeInLogo(url, -1, 300, ...)`|로고 메서드에 검증 없음|가능|
|이미지 버퍼 크기 overflow|`width * height * pixelSize` 계산이 int 범위 초과|`MAX_QR_DIMENSION` 검증이 로고 메서드에 없음|가능|
|로고 크기 계산 음수|`newLogoWidth = (qrWidth / 2) - 60` 결과가 음수|QR 크기가 작으면 가능|가능|
|BufferedImage 생성|음수 width/height로 이미지 생성|`ImageResize()`에서 검증 없음|가능성 있음|
|ZXing 내부 BitMatrix 생성|내부에서 `int[] bits = new int[...]` 같은 버퍼 생성|크기값 비정상 또는 overflow 시 가능|가능성 있음|

### 1. `createQrCodeInLogo()`에 width/height 검증이 없음

`createQrCode()`는 아래처럼 크기를 검증합니다.

```java
if (width <= 0 || height <= 0) {
    return "";
}
if (width > MAX_QR_DIMENSION || height > MAX_QR_DIMENSION) {
    return "";
}
```

하지만 `createQrCodeInLogo()`는 같은 검증 없이 바로 QR 생성을 수행합니다.

```java
BitMatrix bitMatrix = new MultiFormatWriter().encode(url, BarcodeFormat.QR_CODE, width, height);
```

ZXing `QRCodeWriter` 소스 기준으로 `width < 0 || height < 0`이면 `IllegalArgumentException`을 던지도록 되어 있습니다. 즉, ZXing QR 생성 단계에서는 음수 크기가 보통 `NegativeArraySizeException`이 아니라 `IllegalArgumentException`으로 잡힐 가능성이 큽니다. ([GitHub](https://github.com/zxing/zxing/blob/master/core/src/main/java/com/google/zxing/qrcode/QRCodeWriter.java "zxing/core/src/main/java/com/google/zxing/qrcode/QRCodeWriter.java at master · zxing/zxing · GitHub"))  
그래도 실무적으로는 `createQrCodeInLogo()`에도 반드시 같은 검증을 넣어야 합니다.

```java
private static void validateQrSize(int width, int height) {
    if (width <= 0 || height <= 0) {
        throw new IllegalArgumentException("QR 크기는 0보다 커야 합니다.");
    }
    if (width > MAX_QR_DIMENSION || height > MAX_QR_DIMENSION) {
        throw new IllegalArgumentException("QR 크기가 허용 최대값을 초과했습니다.");
    }
}
```

### 2. 너무 큰 width/height로 인한 int overflow

`NegativeArraySizeException`은 단순히 음수 파라미터만이 아니라, 내부 계산 결과가 음수가 될 때도 발생할 수 있습니다.  
예:

```java
int width = 100000;
int height = 100000;
int size = width * height * 4; // int overflow로 음수 가능
byte[] buffer = new byte[size]; // NegativeArraySizeException 가능
```

현재 `createQrCode()`는 `MAX_QR_DIMENSION = 2000` 검증이 있어 어느 정도 방어가 됩니다. 하지만 `createQrCodeInLogo()`와 `ImageResize()`에는 이 검증이 없습니다.  
따라서 외부 입력으로 큰 값이 들어오면 다음 계층에서 문제가 날 수 있습니다.

```text
MultiFormatWriter / BitMatrix 내부 배열
BufferedImage 내부 Raster/DataBuffer
ImageIO 내부 인코딩 버퍼
Base64 인코딩 대상 byte[]
```

### 3. 로고 크기 계산에서 음수 크기 발생 가능

첨부 코드에는 다음 로직이 있습니다.

```java
newLogoWidth = (qrWidth / 2) - 60;
newLogoHeight = 0;

if (logoWidth > newLogoWidth) {
    newLogoHeight = (logoHeight * newLogoWidth) / logoWidth;
}
```

예를 들어 QR 이미지가 작으면 다음처럼 됩니다.

```text
qrWidth = 100
newLogoWidth = (100 / 2) - 60 = -10
```

그 다음:

```java
newLogoHeight = (logoHeight * -10) / logoWidth;
```

즉 `newLogoWidth`, `newLogoHeight`가 음수가 될 수 있습니다.  
이 값은 바로 배열 생성에 쓰이지는 않지만, 아래 이미지 그리기 API에 전달됩니다.

```java
graph.drawImage(logoImage, posX, posY, newLogoWidth, newLogoHeight, null);
```

`Graphics.drawImage()`는 음수 width/height를 뒤집기 처리처럼 해석할 수도 있어 즉시 `NegativeArraySizeException`을 보장하지는 않습니다. 하지만 이후 이미지 처리, 리사이즈, 다른 API와 결합될 때 비정상 크기 전파의 원인이 됩니다.  
실무적으로는 아래처럼 제한해야 합니다.

```java
int maxLogoWidth = Math.max(1, qrWidth / 5);
int maxLogoHeight = Math.max(1, qrHeight / 5);

double scale = Math.min(
        (double) maxLogoWidth / logoWidth,
        (double) maxLogoHeight / logoHeight
);

int newLogoWidth = Math.max(1, (int) Math.round(logoWidth * scale));
int newLogoHeight = Math.max(1, (int) Math.round(logoHeight * scale));
```

### 4. `BufferedImage` 생성 시 음수/0 크기

현재 코드에는 `BufferedImage` 생성이 두 곳 있습니다.

```java
BufferedImage combined = new BufferedImage(qrImage.getHeight(), qrImage.getWidth(), BufferedImage.TYPE_INT_ARGB);
```

```java
BufferedImage outputImage = new BufferedImage(width, height, image.getType());
```

`BufferedImage(int width, int height, int imageType)`는 지정한 width/height로 이미지를 생성하는 생성자입니다. 공식 Javadoc에서 이 생성자는 `width`, `height`, `imageType`을 인자로 받는다고 정의되어 있습니다. ([Oracle Docs](https://docs.oracle.com/javase/8/docs/api/java/awt/image/BufferedImage.html "BufferedImage (Java Platform SE 8 )"))  
일반적으로 JDK의 `BufferedImage` 생성자는 width/height가 0 이하이면 `IllegalArgumentException`으로 터지는 경우가 많습니다. 다만 내부 구현이나 관련 Raster/DataBuffer 생성 과정에서 계산된 배열 크기가 음수가 되면 `NegativeArraySizeException`이 발생할 수 있습니다. 그래서 실무에서는 생성자 호출 전에 직접 검증하는 것이 맞습니다.

```java
private static void validateImageSize(int width, int height) {
    if (width <= 0 || height <= 0) {
        throw new IllegalArgumentException("이미지 크기는 0보다 커야 합니다.");
    }
    if (width > MAX_QR_DIMENSION || height > MAX_QR_DIMENSION) {
        throw new IllegalArgumentException("이미지 크기가 허용 최대값을 초과했습니다.");
    }
}
```

### 5. 현재 코드에서 가장 의심해야 할 위치

#### 5-1. 1순위: `createQrCodeInLogo()`의 크기 검증 누락

```java
public static String createQrCodeInLogo(String url, int width, int height, ...)
```

이 메서드는 외부에서 받은 `width`, `height`를 검증 없이 사용합니다.  
발생 가능 입력:

```java
createQrCodeInLogo(url, -1, 300, imgPath, fileName);
createQrCodeInLogo(url, 999999, 999999, imgPath, fileName);
createQrCodeInLogo(url, 100, 100, imgPath, fileName);
```

첫 번째는 ZXing 단계에서 `IllegalArgumentException` 가능성이 크고, 두 번째는 내부 버퍼 계산 overflow로 `NegativeArraySizeException` 또는 `OutOfMemoryError` 가능성이 있습니다. 세 번째는 로고 크기 계산에서 `newLogoWidth`가 음수가 될 수 있습니다.

#### 5-2. 2순위: 로고 크기 계산

```java
newLogoWidth = (qrWidth / 2) - 60;
```

QR 크기가 `119px` 이하이면 `newLogoWidth`는 음수 또는 0이 됩니다.

```text
qrWidth = 100 → newLogoWidth = -10
qrWidth = 120 → newLogoWidth = 0
qrWidth = 200 → newLogoWidth = 40
```

즉, 현재 로직은 최소 QR 크기를 강제하지 않으면 매우 쉽게 비정상 로고 크기를 만들 수 있습니다.

#### 5-3. 3순위: `ImageResize()`

```java
public static BufferedImage ImageResize(BufferedImage image, int width, int height) {
    BufferedImage outputImage = new BufferedImage(width, height, image.getType());
    ...
}
```

이 메서드는 `width`, `height`, `image`에 대한 검증이 없습니다. 또한 내부에서 원본 `image`가 아니라 `outputImage`를 그리고 있어 리사이즈 로직 자체도 잘못되어 있습니다.  
수정:

```java
public static BufferedImage resize(BufferedImage image, int width, int height) {
    if (image == null) {
        throw new IllegalArgumentException("image는 필수입니다.");
    }
    validateImageSize(width, height);

    BufferedImage outputImage = new BufferedImage(width, height, BufferedImage.TYPE_INT_ARGB);
    Graphics2D g = outputImage.createGraphics();
    try {
        g.drawImage(image, 0, 0, width, height, null);
    } finally {
        g.dispose();
    }
    return outputImage;
}
```

### 6. 현재 코드에 바로 적용할 방어 코드

```java
private static final int MIN_QR_DIMENSION = 200;
private static final int MAX_QR_DIMENSION = 2000;

private static void validateQrInput(String url, int width, int height) {
    if (url == null || url.isBlank()) {
        throw new IllegalArgumentException("QR URL은 필수입니다.");
    }
    if (width != height) {
        throw new IllegalArgumentException("QR 이미지는 정사각형을 권장합니다.");
    }
    if (width < MIN_QR_DIMENSION || height < MIN_QR_DIMENSION) {
        throw new IllegalArgumentException("QR 크기는 최소 " + MIN_QR_DIMENSION + "px 이상이어야 합니다.");
    }
    if (width > MAX_QR_DIMENSION || height > MAX_QR_DIMENSION) {
        throw new IllegalArgumentException("QR 크기가 허용 최대값을 초과했습니다.");
    }
}
```

로고 크기는 QR 대비 비율로 계산합니다.

```java
private static int[] calculateLogoSize(BufferedImage qrImage, BufferedImage logoImage) {
    int qrWidth = qrImage.getWidth();
    int qrHeight = qrImage.getHeight();
    int logoWidth = logoImage.getWidth();
    int logoHeight = logoImage.getHeight();

    if (logoWidth <= 0 || logoHeight <= 0) {
        throw new IllegalArgumentException("로고 이미지 크기가 비정상입니다.");
    }

    int maxLogoWidth = Math.max(1, qrWidth / 5);
    int maxLogoHeight = Math.max(1, qrHeight / 5);

    double scale = Math.min(
            (double) maxLogoWidth / logoWidth,
            (double) maxLogoHeight / logoHeight
    );

    int resizedWidth = Math.max(1, (int) Math.round(logoWidth * scale));
    int resizedHeight = Math.max(1, (int) Math.round(logoHeight * scale));

    return new int[] { resizedWidth, resizedHeight };
}
```

### 7. 예외 처리 관점

`NegativeArraySizeException`은 `RuntimeException` 하위 예외이므로 현재 코드의 `catch (RuntimeException e)` 또는 뒤의 `catch (Exception e)`에 잡힐 수 있습니다. 하지만 잡아서 빈 문자열을 반환하는 방식은 원인 파악을 어렵게 합니다.  
권장:

```java
try {
    ...
} catch (WriterException | IOException e) {
    throw new IllegalStateException("QR 이미지 생성 실패", e);
} catch (RuntimeException e) {
    LOGGER.error("QR 이미지 처리 중 런타임 오류. width={}, height={}", width, height, e);
    throw e;
}
```

특히 `NegativeArraySizeException`이 발생했다면 대부분 **입력 크기 검증 누락 또는 내부 크기 계산 overflow**이므로, catch로 숨기기보다 로그를 남기고 실패를 명확히 올리는 편이 맞습니다.

### 8. 결론

현재 첨부 코드에서 `NegativeArraySizeException`이 직접 발생할 만한 배열 생성 코드는 보이지 않습니다. 하지만 `createQrCodeInLogo()`와 `ImageResize()`에는 `width`, `height` 검증이 없고, 로고 크기 계산에서 `newLogoWidth = (qrWidth / 2) - 60`이 음수가 될 수 있습니다. 따라서 실제 발생 원인은 **비정상 width/height 입력**, **큰 이미지 크기 계산 overflow**, **로고 크기 계산 결과 음수 전파** 중 하나일 가능성이 큽니다. 우선 `createQrCodeInLogo()` 시작 부분에 `url`, `width`, `height` 검증을 추가하고, 로고 크기를 QR 대비 15~20% 비율로 제한하는 방식으로 수정하는 것이 가장 안전합니다.