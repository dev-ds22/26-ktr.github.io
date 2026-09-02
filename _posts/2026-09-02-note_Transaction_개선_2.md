---
layout: single
title: "Transaction_개선_2"
excerpt: "Transaction_개선_2"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-09-02"
last_modified_at: "2026-09-02 09:47:53 +0900"
mermaid: false
---
## 1. 최종 권장 구조
이번 수정에서는 단순히 `CstmrDmstcDentpDAO` 호출을 Service로 옮기는 것에 그치지 않고, **Controller → Application Service → Business Service → DAO** 계층을 명확히 하는 것이 좋습니다.
```text
[변경 전]
Controller
 ├─ JoinSellerService
 ├─ UserInfoService
 ├─ CstmrDmstcDentpDAO             ← 잘못된 계층 접근
 ├─ CryptoUtil new                 ← 객체 직접 생성
 ├─ ObjectMapper new               ← 객체 직접 생성
 └─ Session 변경
[변경 후]
Controller
 │
 │  HTTP / Request / Response / Session만 담당
 ▼
JoinSellerApplicationService       ← Transaction Boundary
 │
 ├─ JoinSellerService
 │    └─ JoinSellerDAO
 │
 ├─ CstmrDmstcDentpService         ← 신규
 │    └─ CstmrDmstcDentpDAO
 │
 ├─ UserInfoService
 │    └─ UserInfoDAO
 │
 └─ File 관련 Service
      └─ File DAO / Storage
```
핵심 원칙은 다음입니다.

|계층|책임|Transaction|
|---|---|---|
|Controller|HTTP 요청/응답, Session|없음|
|Application Service|한 Use Case 전체 조정|**REQUIRED 시작점**|
|Business Service|도메인별 업무 처리|REQUIRED 참여|
|DAO|SQL 실행|Transaction 직접 제어 안 함|
|DB|실제 물리 Transaction|Application Service 단위로 Commit/Rollback|

이 구조에서는 `JoinSellerApplicationService.saveCompany()`가 하나의 트랜잭션을 시작하고, 기존 `JoinSellerService`, 신규 `CstmrDmstcDentpService`, `UserInfoService`의 `REQUIRED`가 모두 동일 트랜잭션에 참여합니다.
## 2. 이번에 같이 수정해야 하는 기존 구조 문제
현재 코드에는 Transaction 외에도 Architecture 관점에서 개선해야 할 부분이 있습니다.

|현재 구현|문제|개선|
|---|---|---|
|Controller → DAO 직접 호출|Layer 위반|Service 경유|
|Controller가 여러 Service 호출|Transaction 경계 분산|Application Service 추가|
|`new CryptoUtil()`|DI 불가, 테스트 어려움|Spring Bean 주입|
|`new ObjectMapper()`|설정 중복, 객체 반복 생성|Bean 주입|
|Controller에서 VO 업무 가공|Controller 책임 과다|Application Service 이동|
|Controller에서 암호화|업무처리 책임 혼재|Application Service 이동|
|`vo.getDentpInfoSn()` 재사용|INSERT 반환값과 불일치 가능|실제 반환 `dentpInfoSn` 사용|
|수정 시 `setEntpId=""` 가능|Session 데이터 오류|실제 entpId 조회|
|DAO 결과건수 미검증|0건 UPDATE도 성공 취급|Service에서 affected row 검증|
|Exception catch 후 무시 가능성|부분 Commit 원인|반드시 throw|
|파일과 DB를 같은 Transaction처럼 취급|파일은 DB Rollback 대상 아님|별도 보상처리 필요|
|Session 변경 시점 불명확|DB 실패와 Session 불일치 가능|Commit 완료 후 변경|

## 3. Transaction XML 권장 설정
기존 Service Transaction은 그대로 유지합니다. 운영 시스템에서 기존 설정을 대규모 변경하는 것은 위험합니다.
신규 Application Service용 Transaction 경계를 별도로 추가하는 것을 권장합니다.
```xml
<!-- =========================================================
     기존 Business Service Transaction
     기존 운영 Service의 동작을 그대로 유지
========================================================= -->
<tx:advice id="txAdvice" transaction-manager="txManager">
    <tx:attributes>
        <tx:method name="*" propagation="REQUIRED"
                   rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
<aop:config>
    <aop:pointcut id="requiredTx"
        expression="execution(* app..service.*Service.*(..))"/>
    <aop:advisor advice-ref="txAdvice"
                 pointcut-ref="requiredTx"/>
</aop:config>
<!-- =========================================================
     Application Service Transaction
     여러 Business Service를 하나의 Transaction으로 통합
========================================================= -->
<tx:advice id="applicationTxAdvice"
           transaction-manager="txManager">
    <tx:attributes>
        <tx:method name="*" propagation="REQUIRED"
                   rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
<aop:config>
    <aop:pointcut id="applicationServiceTx"
        expression="
            execution(public * app..service.application..*ApplicationService.*(..))
        "/>
    <aop:advisor advice-ref="applicationTxAdvice"
                 pointcut-ref="applicationServiceTx"/>
</aop:config>
```
구조:
```text
app.xxx.service
 ├─ JoinSellerService
 ├─ CstmrDmstcDentpService
 ├─ UserInfoService
 │
 └─ application
      └─ JoinSellerApplicationService
```
신규 Application Service는 기존 `requiredTx`와 별개로 `applicationTxAdvice`를 적용받습니다.
그 안에서 기존 Service를 호출하면:
```text
JoinSellerApplicationService
 REQUIRED
 TX-100 BEGIN
      │
      ├─ JoinSellerService
      │      REQUIRED → TX-100 참여
      │
      ├─ CstmrDmstcDentpService
      │      REQUIRED → TX-100 참여
      │
      └─ UserInfoService
             REQUIRED → TX-100 참여
                     │
                     ▼
                 전체 성공
                     │
                     ▼
               TX-100 COMMIT
```
## 4. 신규 `CstmrDmstcDentpService`
Controller에서 직접 호출했던:
```java
cstmrDmstcDentpDAO.updateCompanyInfomation(sellerDentpInfoVO);
```
를 다음 구조로 변경합니다.
```text
Controller
  X
  └─ DAO 직접 접근 금지
Application Service
       ↓
CstmrDmstcDentpService
       ↓
CstmrDmstcDentpDAO
```
### 4-1. Service Interface
```java
/**
 * 판매자 기업정보 Service.
 *
 * <pre>
 * << 개정이력(Modification Information) >>
 *
 *   수정일          수정내용
 *  ----------      -----------------------------------------
 *  2026.09.02      Controller DAO 직접호출 제거를 위한 Service 추가
 * </pre>
 *
 * @author
 * @since 2026.09.02
 * @version 1.0
 * @see
 */
public interface CstmrDmstcDentpService {
    /**
     * 기업정보를 수정한다.
     *
     * @param sellerDentpInfoVO 기업정보
     * @exception SellerCompanyProcessException 기업정보 수정 실패
     */
    void updateCompanyInformation(
            SellerDentpInfoVO sellerDentpInfoVO);
}
```
### 4-2. Service 구현
```java
import org.springframework.stereotype.Service;
/**
 * 판매자 기업정보 Service 구현체.
 *
 * <pre>
 * << 개정이력(Modification Information) >>
 *
 *   수정일          수정내용
 *  ----------      -----------------------------------------
 *  2026.09.02      최초 생성
 * </pre>
 *
 * @author
 * @since 2026.09.02
 * @version 1.0
 * @see CstmrDmstcDentpService
 */
@Service
public class CstmrDmstcDentpServiceImpl
        implements CstmrDmstcDentpService {
    private final CstmrDmstcDentpDAO cstmrDmstcDentpDAO;
    /**
     * 생성자.
     *
     * @param cstmrDmstcDentpDAO 기업정보 DAO
     */
    public CstmrDmstcDentpServiceImpl(
            CstmrDmstcDentpDAO cstmrDmstcDentpDAO) {
        this.cstmrDmstcDentpDAO = cstmrDmstcDentpDAO;
    }
    /**
     * 기업정보를 수정한다.
     *
     * @param sellerDentpInfoVO 기업정보
     * @exception SellerCompanyProcessException 수정 실패
     */
    @Override
    public void updateCompanyInformation(
            SellerDentpInfoVO sellerDentpInfoVO) {
        /*
         * 기업정보 UPDATE 실행
         */
        int updateCount =
                cstmrDmstcDentpDAO.updateCompanyInfomation(
                        sellerDentpInfoVO);
        /*
         * UPDATE 대상이 정확히 1건인지 검증한다.
         *
         * 0건 UPDATE를 정상으로 처리하면
         * 이후 다른 테이블만 Commit되는 논리 오류가 발생할 수 있다.
         */
        if (updateCount != 1) {
            throw new SellerCompanyProcessException(
                    "기업정보 수정 결과가 올바르지 않습니다."
                    + " updateCount=" + updateCount);
        }
    }
}
```
여기서 중요한 개선점은 DAO가 단순히 성공/실패만 수행하는 것이 아니라 **영향받은 Row 수를 Service가 검증한다는 것**입니다.
## 5. DAO 개선
DAO는 비즈니스 판단이나 Exception 복구를 하지 않습니다.
```java
@Repository
public class CstmrDmstcDentpDAO extends EgovAbstractMapper {
    /**
     * 기업정보를 수정한다.
     *
     * @param sellerDentpInfoVO 기업정보
     * @return 수정된 건수
     */
    public int updateCompanyInfomation(
            SellerDentpInfoVO sellerDentpInfoVO) {
        return update(
                "CstmrDmstcDentpDAO.updateCompanyInfomation",
                sellerDentpInfoVO);
    }
}
```
가능하면 메소드 오타:
```java
updateCompanyInfomation
```
도 장기적으로:
```java
updateCompanyInformation
```
으로 수정하는 것이 맞습니다.
다만 Mapper ID까지 영향을 받는다면 이번 Transaction 장애 개선과 분리하여 적용하는 것이 안전합니다.
## 6. Exception 클래스
Application/Business Layer에서 사용할 RuntimeException을 정의하는 것을 권장합니다.
```java
/**
 * 판매자 기업정보 처리 Exception.
 *
 * <pre>
 * << 개정이력(Modification Information) >>
 *
 *   수정일          수정내용
 *  ----------      -----------------------------------------
 *  2026.09.02      최초 생성
 * </pre>
 *
 * @author
 * @since 2026.09.02
 * @version 1.0
 */
public class SellerCompanyProcessException
        extends RuntimeException {
    private static final long serialVersionUID = 1L;
    /**
     * 생성자.
     *
     * @param message 오류 메시지
     */
    public SellerCompanyProcessException(String message) {
        super(message);
    }
    /**
     * 생성자.
     *
     * @param message 오류 메시지
     * @param cause 원인 Exception
     */
    public SellerCompanyProcessException(
            String message,
            Throwable cause) {
        super(message, cause);
    }
}
```
`RuntimeException`으로 만드는 이유는 Transaction과 잘 맞기 때문입니다.
현재 XML은:
```xml
rollback-for="Exception"
```
이므로 checked Exception도 Rollback 대상이지만, 업무 실패를 표현하는 Application Exception은 RuntimeException 계열을 사용하는 것이 일반적으로 관리하기 쉽습니다.
## 7. 처리 결과 객체
Controller와 Service 사이에서는 여러 값을 흩어서 관리하지 않고 Result 객체로 반환합니다.
```java
/**
 * 기업정보 등록/수정 처리 결과.
 *
 * @since 2026.09.02
 * @version 1.0
 */
public class JoinSellerResult {
    private final long dentpInfoSn;
    private final String entpId;
    private final String atchSn;
    public JoinSellerResult(
            long dentpInfoSn,
            String entpId,
            String atchSn) {
        this.dentpInfoSn = dentpInfoSn;
        this.entpId = entpId;
        this.atchSn = atchSn;
    }
    public long getDentpInfoSn() {
        return dentpInfoSn;
    }
    public String getEntpId() {
        return entpId;
    }
    public String getAtchSn() {
        return atchSn;
    }
}
```
## 8. 최종 `JoinSellerApplicationService`
이 Service가 **실제 하나의 Commit 단위**가 됩니다.
```java
import java.util.List;
import java.util.Map;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.dao.DataAccessException;
import org.springframework.stereotype.Service;
import org.springframework.util.StringUtils;
import org.springframework.web.multipart.MultipartFile;
/**
 * 판매자 기업 등록/수정 Application Service.
 *
 * <p>
 * 기업정보 등록/수정과 관련된 복수 Business Service를
 * 하나의 Transaction 단위로 처리한다.
 * </p>
 *
 * <pre>
 * << 개정이력(Modification Information) >>
 *
 *   수정일          수정내용
 *  ----------      -----------------------------------------
 *  2026.09.02      Transaction 통합 처리를 위해 최초 생성
 * </pre>
 *
 * @author
 * @since 2026.09.02
 * @version 1.0
 * @see JoinSellerService
 * @see CstmrDmstcDentpService
 * @see UserInfoService
 */
@Service
public class JoinSellerApplicationService {
    private final JoinSellerService joinSellerService;
    private final CstmrDmstcDentpService cstmrDmstcDentpService;
    private final UserInfoService userInfoService;
    private final CryptoUtil cryptoUtil;
    private final ObjectMapper objectMapper;
    /**
     * 생성자.
     */
    public JoinSellerApplicationService(
            JoinSellerService joinSellerService,
            CstmrDmstcDentpService cstmrDmstcDentpService,
            UserInfoService userInfoService,
            CryptoUtil cryptoUtil,
            ObjectMapper objectMapper) {
        this.joinSellerService = joinSellerService;
        this.cstmrDmstcDentpService =
                cstmrDmstcDentpService;
        this.userInfoService = userInfoService;
        this.cryptoUtil = cryptoUtil;
        this.objectMapper = objectMapper;
    }
    /**
     * 기업정보를 신규 등록하거나 수정한다.
     *
     * <p>
     * 본 메소드가 하나의 Transaction Boundary이다.
     * 내부에서 호출되는 REQUIRED Service는 모두 동일한
     * 물리 Transaction에 참여한다.
     * </p>
     *
     * @param vo 기업정보
     * @param mberSn 회원 일련번호
     * @param loginId 로그인 ID
     * @param files 첨부파일
     * @return 처리 결과
     * @exception SellerCompanyProcessException 처리 실패
     */
    public JoinSellerResult saveCompany(
            CstmrDentpInfoVO vo,
            String mberSn,
            String loginId,
            Map<String, MultipartFile> files) {
        try {
            /*
             * 1. 요청 데이터 업무용 변환
             */
            prepareCompanyInfo(
                    vo,
                    loginId);
            long dentpInfoSn =
                    vo.getDentpInfoSn();
            String entpId;
            String atchSn = "";
            /*
             * 2. 신규 기업 등록
             */
            if (dentpInfoSn == 0L) {
                dentpInfoSn =
                        insertNewCompany(
                                vo,
                                mberSn,
                                loginId);
            } else {
                /*
                 * 3. 기존 기업정보 수정
                 */
                updateCompany(
                        vo,
                        loginId);
            }
            /*
             * INSERT 또는 UPDATE에 사용된 실제 기업번호를
             * VO에도 동일하게 설정한다.
             */
            vo.setDentpInfoSn(
                    dentpInfoSn);
            /*
             * 4. 실제 기업 ID 조회
             *
             * 기존 구현은 신규일 때만 entpId를 조회하여
             * 수정 시 "" 값이 Session에 저장될 수 있었다.
             */
            entpId =
                    joinSellerService.selectEntpId(
                            dentpInfoSn);
            if (!StringUtils.hasText(entpId)) {
                throw new SellerCompanyProcessException(
                        "기업 ID 조회에 실패했습니다."
                        + " dentpInfoSn=" + dentpInfoSn);
            }
            /*
             * 5. 첨부정보 처리
             */
            if (files != null
                    && !files.isEmpty()) {
                atchSn =
                        processAttachment(
                                files,
                                dentpInfoSn,
                                loginId);
            }
            /*
             * 이 메소드가 정상 Return하면
             * Spring Transaction Interceptor가 Commit한다.
             */
            return new JoinSellerResult(
                    dentpInfoSn,
                    entpId,
                    atchSn);
        } catch (SellerCompanyProcessException e) {
            /*
             * 이미 업무 의미를 포함하여 생성된 Exception은
             * 원형 그대로 상위 계층에 전달한다.
             *
             * 반드시 다시 throw 해야 Rollback된다.
             */
            throw e;
        } catch (DataAccessException e) {
            /*
             * Spring/MyBatis 계열 DB Exception에
             * 현재 Use Case 정보를 추가하여 전달한다.
             */
            throw new SellerCompanyProcessException(
                    "기업정보 DB 처리 중 오류가 발생했습니다.",
                    e);
        } catch (Exception e) {
            /*
             * 암호화, 파일 처리 등 checked Exception도
             * RuntimeException으로 변환하여 상위에 전달한다.
             */
            throw new SellerCompanyProcessException(
                    "기업정보 처리 중 오류가 발생했습니다.",
                    e);
        }
    }
    /**
     * 기업정보 입력값을 업무 데이터로 변환한다.
     */
    private void prepareCompanyInfo(
            CstmrDentpInfoVO vo,
            String loginId) throws Exception {
        vo.setEntpId(
                CommonConstants.MM_SLLR_PREFIX_STR);
        vo.setZip(
                vo.getCmZip());
        vo.setAdres(
                vo.getCmAdres());
        vo.setDtlAdres(
                vo.getCmDtlAdres());
        vo.setEngAdres(
                vo.getCmEngAdres());
        vo.setEngDtlAdres(
                vo.getCmEngDtlAdres());
        /*
         * 대표 전화번호 암호화
         */
        if (StringUtils.hasText(
                vo.getRpsntTelno())) {
            vo.setRpsntTelno(
                    cryptoUtil.encrypt(
                            vo.getRpsntTelno(),
                            "SEED"));
        }
        /*
         * 대표 이메일 암호화
         */
        if (StringUtils.hasText(
                vo.getRpsntEmail())) {
            vo.setRpsntEmail(
                    cryptoUtil.encrypt(
                            vo.getRpsntEmail(),
                            "SEED"));
        }
        vo.setRegtrId(
                loginId);
        vo.setUpdusrId(
                loginId);
    }
    /**
     * 신규 기업정보를 등록한다.
     */
    private long insertNewCompany(
            CstmrDentpInfoVO vo,
            String mberSn,
            String loginId) {
        /*
         * 기업정보 등록
         */
        long dentpInfoSn =
                joinSellerService.insertNewCompany(
                        vo);
        if (dentpInfoSn <= 0L) {
            throw new SellerCompanyProcessException(
                    "기업정보 등록 결과가 올바르지 않습니다.");
        }
        /*
         * 기업정보 History 등록
         */
        CstmrDentpInfoVO historyVO =
                new CstmrDentpInfoVO();
        historyVO.setDentpInfoSn(
                dentpInfoSn);
        historyVO.setRegtrId(
                loginId);
        joinSellerService.insertDentpInfoHistory(
                historyVO);
        /*
         * 기업 ID 조회
         */
        String entpId =
                joinSellerService.selectEntpId(
                        dentpInfoSn);
        if (!StringUtils.hasText(entpId)) {
            throw new SellerCompanyProcessException(
                    "신규 기업 ID 조회에 실패했습니다.");
        }
        /*
         * 회원 Master 기업정보 수정
         */
        CstmrDmstcMberMstVO mberVO =
                new CstmrDmstcMberMstVO();
        mberVO.setDmstcMberSn(
                Long.parseLong(mberSn));
        mberVO.setEntpId(
                entpId);
        userInfoService.updateEntpMst(
                mberVO);
        /*
         * 회원-기업 관계정보 등록
         */
        CstmrDmstcMberEntpRelVO relVO =
                new CstmrDmstcMberEntpRelVO();
        relVO.setDmstcMberSn(
                Long.parseLong(mberSn));
        relVO.setDentpInfoSn(
                dentpInfoSn);
        relVO.setAfflcoMberSttsCd(
                CommonConstants.AFFLCO_MBER_STTS_CD_10);
        relVO.setResgnYn("N");
        relVO.setDelYn("N");
        relVO.setUseYn("Y");
        relVO.setRegtrId(
                loginId);
        relVO.setUpdusrId(
                loginId);
        userInfoService.insertCstmrDmstcMberEntpRel(
                relVO);
        return dentpInfoSn;
    }
    /**
     * 기존 기업정보를 수정한다.
     */
    private void updateCompany(
            CstmrDentpInfoVO vo,
            String loginId) {
        /*
         * ObjectMapper는 직접 생성하지 않고
         * Spring Bean으로 주입받은 인스턴스를 사용한다.
         */
        SellerDentpInfoVO sellerDentpInfoVO =
                objectMapper.convertValue(
                        vo,
                        SellerDentpInfoVO.class);
        sellerDentpInfoVO.setRegtrId(
                loginId);
        sellerDentpInfoVO.setUpdusrId(
                loginId);
        /*
         * Application Service가 DAO를 직접 호출하지 않고
         * 반드시 Business Service를 통한다.
         */
        cstmrDmstcDentpService
                .updateCompanyInformation(
                        sellerDentpInfoVO);
    }
    /**
     * 첨부정보를 처리한다.
     */
    private String processAttachment(
            Map<String, MultipartFile> files,
            long dentpInfoSn,
            String loginId) throws Exception {
        List<CstmrAtchImgListVO> atchList =
                joinSellerService.parseFileInf(
                        files,
                        true);
        if (atchList == null
                || atchList.isEmpty()) {
            throw new SellerCompanyProcessException(
                    "첨부파일 처리 결과가 없습니다.");
        }
        CstmrAtchImgListVO atchVO =
                atchList.get(0);
        /*
         * 기존 코드의 vo.getDentpInfoSn()이 아닌
         * 실제 INSERT/UPDATE 대상 기업번호를 사용한다.
         */
        atchVO.setDentpInfoSn(
                dentpInfoSn);
        atchVO.setUpdusrId(
                loginId);
        atchVO.setRegtrId(
                loginId);
        joinSellerService.updateFileInfs(
                atchVO);
        /*
         * 실제 VO에 첨부번호 Getter가 있다면 사용한다.
         * 프로젝트의 실제 필드명에 맞춰 조정한다.
         */
        return atchVO.getAtchSn();
    }
}
```
## 9. `ObjectMapper`, `CryptoUtil` Bean 설정
현재:
```java
CryptoUtil cryptoUtil = new CryptoUtil();
ObjectMapper mapper = new ObjectMapper();
```
는 제거하는 것이 좋습니다.
XML 기반 프로젝트라면 예를 들어:
```xml
<bean id="cryptoUtil"
      class="app.common.util.CryptoUtil"/>
<bean id="objectMapper"
      class="com.fasterxml.jackson.databind.ObjectMapper">
</bean>
```
만약 `FAIL_ON_UNKNOWN_PROPERTIES=false` 설정을 공통 적용하고 싶다면 Java Config에서는:
```java
@Bean
public ObjectMapper objectMapper() {
    ObjectMapper objectMapper =
            new ObjectMapper();
    objectMapper.configure(
            DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES,
            false);
    return objectMapper;
}
```
로 구성할 수 있습니다.
이미 프로젝트에 공통 `ObjectMapper` Bean이 있다면 새로 생성하지 말고 기존 Bean을 사용해야 합니다.
## 10. 수정된 Controller
Controller는 매우 단순해집니다.
```java
@RequestMapping("/test/insertNewCompany.do")
@ResponseBody
public Map<String, Object> insertNewCompany(
        MultipartHttpServletRequest multiRequest,
        ViewBaseVO view,
        CstmrDentpInfoVO vo) {
    ModelMap resultMap =
            new ModelMap();
    Map<String, MultipartFile> files =
            multiRequest.getFileMap();
    /*
     * =========================================================
     * 하나의 Application Service만 호출한다.
     *
     * 해당 호출이 정상적으로 반환되는 시점에는
     * Spring Transaction Interceptor의 Commit까지 완료된다.
     * =========================================================
     */
    JoinSellerResult result =
            joinSellerApplicationService.saveCompany(
                    vo,
                    view.getMberSn(),
                    view.getLoginId(),
                    files);
    /*
     * DB Commit 성공 후 Session 정보를 변경한다.
     */
    SessionVO sessionVO =
            sessionService.getSession();
    sessionVO.setIsDmstcEntpMst(
            true);
    sessionVO.setEntpId(
            result.getEntpId());
    sessionVO.setDentpSn(
            String.valueOf(
                    result.getDentpInfoSn()));
    view.setIsMaster(
            true);
    view.setEntpId(
            result.getEntpId());
    view.setDentpSn(
            String.valueOf(
                    result.getDentpInfoSn()));
    /*
     * Response 생성
     */
    resultMap.addAttribute(
            "dentpInfoSn",
            result.getDentpInfoSn());
    resultMap.addAttribute(
            "atchSn",
            result.getAtchSn());
    return resultMap;
}
```
Controller에서 이제 제거되는 항목은 다음과 같습니다.
```text
X DAO 호출
X 암호화
X ObjectMapper 변환
X 기업 등록 업무 흐름
X History 생성
X 회원 Master 수정
X 관계정보 등록
X 첨부 DB정보 처리
```
남는 것은:
```text
HTTP Request
 ↓
Application Service 호출
 ↓
Commit 성공
 ↓
Session 변경
 ↓
Response
```
입니다.
## 11. Exception 처리 원칙
이 부분이 Transaction 문제와 직접 연결됩니다.
### 11-1. DAO Exception 처리
#### 11-1-1. 잘못된 구현
```java
public int updateCompany(...) {
    try {
        return update(...);
    } catch (Exception e) {
        LOGGER.error("오류", e);
        return 0;
    }
}
```
가장 위험합니다.
DB에서 실제 오류가 발생했음에도 상위 Service에는:
```text
return 0
```
만 전달됩니다.
잘못 작성된 상위 코드가 0을 무시하면 Transaction이 정상 종료되어 Commit될 수 있습니다.
#### 11-1-2. 권장
```java
public int updateCompanyInfomation(
        SellerDentpInfoVO vo) {
    return update(
            "CstmrDmstcDentpDAO.updateCompanyInfomation",
            vo);
}
```
DB Exception을 **그대로 위로 올리는 것**이 가장 안전합니다.
MyBatis-Spring/`SqlSessionTemplate` 또는 Spring 기반 Mapper를 사용한다면 일반적으로 SQL 관련 Exception은 Spring `DataAccessException` 계열 RuntimeException으로 변환되어 전달됩니다.
따라서 DAO에서 다시:
```java
catch (Exception e)
```
할 필요가 없습니다.
## 12. Service Exception 처리
Service에는 두 가지 Exception이 있습니다.
#### 12-1-1. ① 기술적 Exception
예:
```text
Deadlock
Duplicate Key
Connection 오류
SQL 문법 오류
Timeout
```
이 경우 일반적으로 DAO의 Exception을 그대로 전달하거나 Application Service에서 의미 있는 Exception으로 한 번만 wrapping 합니다.
```java
catch (DataAccessException e) {
    throw new SellerCompanyProcessException(
            "기업정보 DB 처리 실패",
            e);
}
```
반드시:
```java
cause
```
를 보존해야 합니다.
잘못된 코드:
```java
throw new SellerCompanyProcessException(
        "기업정보 처리 실패");
```
이렇게 하면 실제:
```text
SQLState
MariaDB error code
Constraint name
Deadlock 정보
```
가 사라집니다.
올바른 코드:
```java
throw new SellerCompanyProcessException(
        "기업정보 처리 실패",
        e);
```
## 13. 업무적 Exception
예:
```java
int updateCount =
        dao.update(...);
if (updateCount != 1) {
    throw new SellerCompanyProcessException(
            "기업정보 수정 건수가 올바르지 않습니다.");
}
```
SQL은 정상 실행됐지만 업무적으로는 실패한 경우입니다.
이것도 반드시 Exception으로 처리해야 전체 Transaction이 Rollback됩니다.
예를 들어:
```text
기업정보 UPDATE       0건
회원 Master UPDATE    1건
관계정보 INSERT       1건
```
을 성공으로 처리하면 데이터 불일치가 발생합니다.
따라서 Service에서:
```text
expected = 1
actual   = 0
```
이면 Exception을 발생시켜야 합니다.
## 14. 절대 하면 안 되는 Exception 처리
### 14-1. Case 1. Catch 후 로그만 출력
```java
try {
    dao.insert(...);
} catch (Exception e) {
    LOGGER.error("등록 오류", e);
}
```
결과:
```text
Exception 발생
 ↓
Service가 Exception 제거
 ↓
Spring Transaction은 정상 종료로 인식
 ↓
COMMIT 가능
```
가장 위험합니다.
### 14-2. Case 2. Catch 후 정상값 반환
```java
try {
    return dao.insert(...);
} catch (Exception e) {
    return 0;
}
```
역시 동일합니다.
### 14-3. Case 3. Application Service가 Exception을 먹고 계속 진행
```java
try {
    userInfoService.updateEntpMst(...);
} catch (Exception e) {
    LOGGER.error("오류", e);
}
joinSellerService.updateFileInfs(...);
```
이런 구조도 금지해야 합니다.
## 15. 올바른 Exception 흐름
```text
MariaDB
  │
  │ SQL Exception
  ▼
DAO
  │
  │ DataAccessException
  │ catch 하지 않음
  ▼
Business Service
  │
  │ 필요하면 업무 조건 검증
  │ updateCount != 1
  ▼
Application Service
  │
  │ SellerCompanyProcessException
  ▼
Spring Transaction Interceptor
  │
  ├─ Exception 감지
  ├─ Rollback
  │
  ▼
Controller / GlobalExceptionHandler
```
## 16. 로그는 어느 계층에서 남기는 것이 좋은가
같은 Exception을 모든 계층에서 로그로 남기는 것도 좋지 않습니다.
잘못된 패턴:
```text
DAO                 ERROR
Service             ERROR
Application Service ERROR
Controller          ERROR
```
한 번의 장애가 로그 4건으로 중복 출력됩니다.
Stack Trace도 4번 나옵니다.
권장:

|계층|Error 로그|
|---|---|
|DAO|일반적으로 안 남김|
|Business Service|일반적으로 안 남김|
|Application Service|특별한 업무정보가 필요할 때만|
|Global Exception Handler|최종 ERROR 로그 1회|
예:
```java
@ExceptionHandler(SellerCompanyProcessException.class)
@ResponseBody
public ResponseEntity<Map<String, Object>>
        handleSellerCompanyException(
                SellerCompanyProcessException e) {
    LOGGER.error(
            "판매자 기업정보 처리 실패",
            e);
    Map<String, Object> result =
            new HashMap<String, Object>();
    result.put(
            "success",
            false);
    result.put(
            "message",
            "기업정보 처리 중 오류가 발생했습니다.");
    return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(result);
}
```
중요한 것은 DB의 상세 Exception:
```text
Duplicate entry ...
SQLState ...
constraint ...
```
를 사용자에게 그대로 보내지 않는 것입니다.
사용자 응답:
```text
기업정보 처리 중 오류가 발생했습니다.
```
서버 로그:
```text
SellerCompanyProcessException
 caused by DataIntegrityViolationException
 caused by SQLIntegrityConstraintViolationException
 ...
```
식으로 남아야 합니다.
## 17. `rollback-for="Exception"`과 현재 Exception 전략
현재:
```xml
<tx:method
    name="*"
    propagation="REQUIRED"
    rollback-for="Exception"/>
```
은 유지해도 좋습니다.
현재 구성에서는:
```text
RuntimeException
SellerCompanyProcessException
DataAccessException
checked Exception
```
모두 Rollback됩니다.
다만 신규 코드에서는 가능하면:
```java
SellerCompanyProcessException
extends RuntimeException
```
으로 통일하고, checked Exception을 Application Service에서 wrapping하는 방식을 권장합니다.
## 18. `UnexpectedRollbackException`이 발생할 수 있는 패턴
`REQUIRED` 환경에서 특히 주의해야 합니다.
```java
try {
    userInfoService.updateEntpMst(...);
} catch (Exception e) {
    LOGGER.error("무시", e);
}
/*
 * 계속 정상 Return
 */
return result;
```
내부 Service Transaction Advisor가 이미:
```text
rollback-only
```
로 표시했다면, 외부 Application Service가 Exception을 먹고 정상 종료해도 실제 Commit 시:
```text
UnexpectedRollbackException
```
이 발생할 수 있습니다.
즉 `REQUIRED` 환경에서는 더더욱:
> **중간 Exception을 catch한 뒤 업무를 계속 진행하지 않는 것**
이 중요합니다.
## 19. 파일 처리 Exception은 별도로 봐야 함
현재:
```java
joinSellerService.parseFileInf(files, true);
```
가 실제 물리 디스크에 파일을 저장한다면 DB Transaction과 완전히 다른 문제입니다.
```text
파일 저장 성공
 ↓
기업 INSERT 성공
 ↓
회원 UPDATE 실패
 ↓
DB ROLLBACK
 ↓
파일은 그대로 존재
```
파일시스템은 MariaDB Transaction에 참여하지 않습니다.
따라서 장기적으로는:
```text
1. 파일 임시 영역 저장
2. DB Transaction 실행
3. Commit 성공
4. 파일 최종 위치 이동
```
또는:
```text
1. 파일 저장
2. DB 처리
3. DB 실패
4. 저장파일 삭제 Compensation
```
가 필요합니다.
이 부분은 DB Transaction 통합과 별개의 Atomicity 문제입니다.
## 20. 최종 구조
```text
┌───────────────────────────────────────────────┐
│ Controller                                    │
│                                               │
│ - Request                                     │
│ - Response                                    │
│ - Session                                     │
│ - DB/DAO/업무처리 없음                         │
└──────────────────────┬────────────────────────┘
                       │
                       ▼
┌───────────────────────────────────────────────┐
│ JoinSellerApplicationService                  │
│                                               │
│ REQUIRED                                      │
│ ===== Transaction Boundary =====              │
└───────────┬───────────┬───────────┬───────────┘
            │           │           │
            ▼           ▼           ▼
   JoinSellerService  DentpService  UserInfoService
       REQUIRED        REQUIRED       REQUIRED
            │           │           │
            ▼           ▼           ▼
          DAO          DAO          DAO
            │           │           │
            └───────────┼───────────┘
                        ▼
                    MariaDB
                        │
               ┌────────┴─────────┐
               │                  │
             성공               Exception
               │                  │
             COMMIT             ROLLBACK
```
## 21. 이번 수정에서 반드시 적용할 우선순위
|우선순위|수정|필수 여부|
|---:|---|---|
|1|`JoinSellerApplicationService` 추가|필수|
|2|Controller 복수 Service 호출 제거|필수|
|3|Controller → DAO 직접 접근 제거|필수|
|4|`CstmrDmstcDentpService` 추가|필수|
|5|기존 하위 Service `REQUIRED` 유지|필수|
|6|DAO Exception swallow 제거|필수|
|7|Service 영향건수 검증|강력 권장|
|8|Application Exception RuntimeException화|강력 권장|
|9|`CryptoUtil`, `ObjectMapper` DI|권장|
|10|Session 변경을 Commit 이후로 이동|필수|
|11|파일 저장 Rollback/보상처리 설계|별도 필수 검토|
|12|DAO 메소드 오타 정리|2차 리팩터링|
이번 장애의 해결 관점에서는 **Controller를 얇게 만들고 `JoinSellerApplicationService` 하나를 Transaction Boundary로 설정한 후, 그 아래 모든 DB 작업을 Business Service → DAO로 통일하는 구조가 가장 적절합니다.**
특히 기존 `JoinSellerService`, `UserInfoService`의 `REQUIRED`를 제거하거나 `SUPPORTS`로 바꾸지 않는 것이 중요합니다. 기존 서비스 단독 호출의 Transaction 안정성을 보존하면서 신규 Application Service가 존재하는 경우에는 자연스럽게 동일 Transaction에 참여하게 만드는 방식이 **운영 소스 변경 최소화와 아키텍처 개선을 동시에 만족시키는 방법**입니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
