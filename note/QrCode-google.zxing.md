## Google ZXing 패키지 설명 및 QR 코드 생성 예제

### 1. ZXing 개요

ZXing은 Google에서 시작된 오픈소스 바코드/QR 코드 처리 라이브러리입니다. Java 기준으로 `core` 모듈은 바코드 인코딩/디코딩의 핵심 기능을 제공하고, `javase` 모듈은 `BitMatrix`를 이미지 파일·스트림·`BufferedImage`로 변환하는 Java SE 유틸리티를 제공합니다. Maven Central 기준 현재 `com.google.zxing:core`, `com.google.zxing:javase`의 최신 버전은 `3.5.4`입니다. ([Maven Central](https://central.sonatype.com/artifact/com.google.zxing/core "Maven Central: com.google.zxing:core"))

### 2. 주요 패키지 구조

|구분|패키지/클래스|설명|
|---|---|---|
|핵심|`com.google.zxing`|`BarcodeFormat`, `EncodeHintType`, `WriterException` 등 공통 타입|
|QR 생성|`com.google.zxing.qrcode.QRCodeWriter`|문자열을 QR 코드용 `BitMatrix`로 변환|
|이미지 변환|`com.google.zxing.client.j2se.MatrixToImageWriter`|`BitMatrix`를 PNG/JPG 이미지로 출력|
|오류 보정|`com.google.zxing.qrcode.decoder.ErrorCorrectionLevel`|QR 손상 복원 수준 지정|
|`QRCodeWriter#encode()`는 문자열, 바코드 포맷, 가로/세로 크기, 추가 힌트를 받아 `BitMatrix`를 반환합니다. 공식 Javadoc상 `contents`, `BarcodeFormat`, `width`, `height`, `hints`를 인자로 받을 수 있고, 인코딩이 불가능하면 `WriterException`이 발생합니다. ([zxing.github.io](https://zxing.github.io/zxing/apidocs/com/google/zxing/qrcode/QRCodeWriter.html "QRCodeWriter (ZXing 3.5.4 API)"))|||

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

|레벨|복원력|데이터 밀도|실무 용도|
|---|--:|--:|---|
|L|낮음|낮음|화면 전용, 깨끗한 환경|
|M|보통|보통|일반 웹/모바일 QR 기본값|
|Q|높음|높음|인쇄물, 약간의 오염 가능성|
|H|매우 높음|매우 높음|로고 삽입, 훼손 가능성 있는 QR|
|주의할 점은 오류 보정 레벨을 높이면 복원력은 좋아지지만 QR 패턴이 복잡해집니다. 데이터가 긴 상태에서 `H`를 쓰면 오히려 인식이 어려워질 수 있으므로, 실무 기본값은 `M`, 로고 삽입이나 인쇄물은 `Q` 또는 `H`를 검토하는 방식이 적절합니다.||||

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