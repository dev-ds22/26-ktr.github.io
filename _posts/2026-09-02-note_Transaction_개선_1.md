---
layout: single
title: "Transaction_개선_1"
excerpt: "Transaction_개선_1"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-09-02"
last_modified_at: "2026-09-02 09:47:49 +0900"
mermaid: false
---
## 1. 결론
현재 구조에서 가장 안전하고 운영소스 변경량이 적은 개선 방법은 **기존 `joinSellerService`, `userInfoService`의 `REQUIRED` 설정을 그대로 유지하면서, 그 위에 하나의 상위 Transaction Service(Facade/Application Service)를 추가하여 Controller가 그 Service를 단 한 번만 호출하도록 변경하는 것**입니다.
```text
[현재]
Controller
 ├─ joinSellerService.insertNewCompany()            TX-1 → COMMIT
 ├─ joinSellerService.selectEntpId()                TX-2 → COMMIT
 ├─ joinSellerService.insertDentpInfoHistory()       TX-3 → COMMIT
 ├─ userInfoService.updateEntpMst()                 TX-4 → COMMIT
 ├─ userInfoService.insert...Rel()                  TX-5 → COMMIT
 └─ joinSellerService.updateFileInfs()              TX-6 → COMMIT
[개선]
Controller
 └─ JoinSellerTransactionService.saveCompany()      TX-1 시작
     ├─ joinSellerService.insertNewCompany()         ┐
     ├─ joinSellerService.selectEntpId()             │
     ├─ joinSellerService.insertDentpInfoHistory()   │
     ├─ userInfoService.updateEntpMst()              ├─ 동일 TX 참여
     ├─ userInfoService.insert...Rel()                │
     └─ joinSellerService.updateFileInfs()            ┘
                                                    ↓
                              전부 성공 → COMMIT 1회
                              하나라도 실패 → 전체 ROLLBACK
```
Spring 5.3의 `PROPAGATION_REQUIRED`는 외부 트랜잭션이 존재하면 그것에 참여하고, 없으면 새 트랜잭션을 생성합니다. Spring 문서에서도 여러 Repository/Service 작업을 하나로 묶는 대표적인 구조로 상위 Service Facade를 예시로 들고 있습니다. 
따라서 **기존 하위 Service들의 `REQUIRED`를 `SUPPORTS` 등으로 변경할 필요가 없습니다. 오히려 그렇게 하면 기존 호출 경로에 예상하지 못한 영향을 줄 가능성이 있습니다.**
## 2. 현재 코드가 부분 Commit되는 이유
현재 설정은 다음입니다.
```xml
<tx:advice id="txAdvice" transaction-manager="txManager">
    <tx:attributes>
        <tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
<aop:config>
    <aop:pointcut id="requiredTx"
        expression="execution(* app..service.*Service.*(..))"/>
    <aop:advisor advice-ref="txAdvice" pointcut-ref="requiredTx" />
</aop:config>
```
`REQUIRED`에 문제가 있는 것이 아니라 **트랜잭션 경계(Transaction Boundary)의 위치가 잘못된 것**입니다.
예를 들어 Controller에서:
```java
joinSellerService.insertNewCompany(vo);
joinSellerService.insertDentpInfoHistory(cstmrDentpInfoVO);
userInfoService.updateEntpMst(mberVO);
userInfoService.insertCstmrDmstcMberEntpRel(relVO);
```
각 호출은 Controller → Spring Proxy → Service 형태입니다.
첫 번째 호출 시 트랜잭션이 없으므로:
```text
Controller
    ↓
Spring Transaction Proxy
    ↓
insertNewCompany()
TX BEGIN
INSERT
TX COMMIT
```
메소드가 끝나면 Commit합니다.
두 번째 호출 시 이전 트랜잭션은 이미 종료되어 있으므로:
```text
insertDentpInfoHistory()
새 TX BEGIN
INSERT
TX COMMIT
```
이런 식입니다.
따라서 예를 들어 마지막 관계정보 INSERT에서 오류가 발생하면:

| 처리                | 결과          |
| ----------------- | ----------- |
| 기업정보 INSERT       | 이미 COMMIT   |
| 기업 History INSERT | 이미 COMMIT   |
| 회원 Master UPDATE  | 이미 COMMIT   |
| 회원-기업 관계 INSERT   | ROLLBACK    |
| 최종 결과             | **데이터 불일치** |
## 3. 가장 권장하는 개선 구조
### 3-1. 권장 Layer
```text
Controller
    ↓
Application/Transaction Service       ← Transaction Boundary
    ↓
┌───────────────────────────────┐
│ JoinSellerService             │
│ UserInfoService               │
│ CstmrDmstcDentpDAO            │
└───────────────────────────────┘
    ↓
DAO / MyBatis
    ↓
MariaDB
```
이때 Controller의 책임은:
```text
HTTP Parameter 처리
입력값 기본 가공
Transaction Service 호출
Response 생성
Session 갱신
```
Transaction Service의 책임은:
```text
하나의 비즈니스 Use Case
+
하나의 Transaction Boundary
```
가 됩니다.
## 4. Transaction XML은 어떻게 변경하는 것이 좋은가
현재 운영중인 기존 Service에 영향을 최소화하려면 **기존 설정을 건드리지 않고 Transaction 전용 Service package를 추가**하는 것을 권장합니다.
예:
```text
app.xxx.service
 ├─ JoinSellerService
 ├─ UserInfoService
 └─ transaction
      └─ JoinSellerTransactionService
```
XML:
```xml
<tx:advice id="txAdvice" transaction-manager="txManager">
    <tx:attributes>
        <tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
<aop:config>
    <!-- 기존 Service Transaction : 기존 운영 동작 유지 -->
    <aop:pointcut id="requiredTx"
        expression="execution(* app..service.*Service.*(..))"/>
    <aop:advisor
        advice-ref="txAdvice"
        pointcut-ref="requiredTx" />
    <!-- 복수 Service를 하나의 Transaction으로 묶는 상위 Service -->
    <aop:pointcut id="transactionServiceTx"
        expression="execution(public * app..service.transaction..*TransactionService.*(..))"/>
    <aop:advisor
        advice-ref="txAdvice"
        pointcut-ref="transactionServiceTx" />
</aop:config>
```
이 방법의 장점은 다음과 같습니다.

| 항목                | 평가       |
| ----------------- | -------- |
| 기존 Service 수정     | 없음       |
| 기존 Controller 수정  | 호출부분만 변경 |
| DAO 수정            | 없음       |
| 기존 Transaction 영향 | 거의 없음    |
| 신규 복합 업무 확장       | 쉬움       |
| Rollback 범위       | 명확       |
| 운영 적용 위험          | 낮음       |

특히 기존 pointcut:
```xml
app..service.*Service.*
```
와 신규:
```xml
app..service.transaction..*TransactionService.*
```
를 분리함으로써 **Transaction Boundary용 Service가 명확하게 식별**됩니다.
장기적으로는
```text
transaction
 ├─ JoinSellerTransactionService
 ├─ OrderTransactionService
 ├─ PaymentTransactionService
 ├─ ClaimTransactionService
 └─ MemberTransactionService
```
형태로 확장할 수 있습니다.
## 5. 신규 `JoinSellerTransactionService`
다음 정도가 현재 소스를 최소 변경하면서 적용하기 가장 좋습니다.
```java
package app.xxx.service.transaction;
import java.util.List;
import java.util.Map;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;
/**
 * 판매자 기업정보 등록/수정 Transaction 통합 Service.
 *
 * <p>
 * 기업정보, 기업이력, 회원정보, 회원-기업관계정보 및 첨부파일 DB 정보를
 * 하나의 Transaction 단위로 처리한다.
 * </p>
 *
 * <pre>
 * << 개정이력(Modification Information) >>
 *
 *   수정일          수정내용
 *  ----------      --------------------------------
 *  2026.09.02      최초 생성
 * </pre>
 *
 * @since 2026.09.02
 * @version 1.0
 */
@Service
public class JoinSellerTransactionService {
    private final JoinSellerService joinSellerService;
    private final UserInfoService userInfoService;
    private final CstmrDmstcDentpDAO cstmrDmstcDentpDAO;
    public JoinSellerTransactionService(
            JoinSellerService joinSellerService,
            UserInfoService userInfoService,
            CstmrDmstcDentpDAO cstmrDmstcDentpDAO) {
        this.joinSellerService = joinSellerService;
        this.userInfoService = userInfoService;
        this.cstmrDmstcDentpDAO = cstmrDmstcDentpDAO;
    }
    /**
     * 판매자 기업정보를 신규 등록하거나 수정한다.
     *
     * <p>
     * 본 메소드는 하나의 Transaction Boundary로 동작하며
     * 내부에서 호출하는 기존 REQUIRED Service는 동일 Transaction에 참여한다.
     * </p>
     *
     * @param vo 기업정보
     * @param mberSn 회원 일련번호
     * @param loginId 로그인 ID
     * @param files 첨부파일
     * @return 처리결과
     * @exception Exception 업무 처리 중 오류
     */
    public JoinSellerTransactionResult saveCompany(
            CstmrDentpInfoVO vo,
            String mberSn,
            String loginId,
            Map<String, MultipartFile> files) throws Exception {
        long dentpInfoSn = vo.getDentpInfoSn();
        String entpId = "";
        /*
         * 1. 신규 기업 등록
         */
        if (dentpInfoSn == 0L) {
            // 기업정보 등록
            dentpInfoSn = joinSellerService.insertNewCompany(vo);
            /*
             * 중요:
             * 이후 로직에서는 입력 VO의 기존 값이 아닌 실제 INSERT 결과로
             * 취득한 dentpInfoSn을 기준으로 처리한다.
             */
            vo.setDentpInfoSn(dentpInfoSn);
            // 생성된 기업 ID 조회
            entpId = joinSellerService.selectEntpId(dentpInfoSn);
            // 기업정보 History 생성
            CstmrDentpInfoVO historyVO = new CstmrDentpInfoVO();
            historyVO.setRegtrId(loginId);
            historyVO.setDentpInfoSn(dentpInfoSn);
            joinSellerService.insertDentpInfoHistory(historyVO);
            // 회원 Master 기업 ID 변경
            CstmrDmstcMberMstVO mberVO = new CstmrDmstcMberMstVO();
            mberVO.setDmstcMberSn(Long.parseLong(mberSn));
            mberVO.setEntpId(entpId);
            userInfoService.updateEntpMst(mberVO);
            // 회원-기업 관계정보 등록
            CstmrDmstcMberEntpRelVO relVO =
                    new CstmrDmstcMberEntpRelVO();
            relVO.setDmstcMberSn(Long.parseLong(mberSn));
            relVO.setDentpInfoSn(dentpInfoSn);
            relVO.setAfflcoMberSttsCd(
                    CommonConstants.AFFLCO_MBER_STTS_CD_10);
            relVO.setResgnYn("N");
            relVO.setDelYn("N");
            relVO.setUseYn("Y");
            relVO.setRegtrId(loginId);
            relVO.setUpdusrId(loginId);
            userInfoService.insertCstmrDmstcMberEntpRel(relVO);
        } else {
            /*
             * 2. 기존 기업정보 수정
             */
            ObjectMapper mapper = new ObjectMapper()
                    .configure(
                            DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES,
                            false);
            SellerDentpInfoVO sellerDentpInfoVO =
                    mapper.convertValue(vo, SellerDentpInfoVO.class);
            sellerDentpInfoVO.setRegtrId(loginId);
            sellerDentpInfoVO.setUpdusrId(loginId);
            cstmrDmstcDentpDAO.updateCompanyInfomation(
                    sellerDentpInfoVO);
            /*
             * 기존 Controller에서는 수정 시 setEntpId가 ""인 상태로
             * Session에 저장될 가능성이 있으므로 실제 기업 ID를 다시 취득한다.
             */
            entpId = joinSellerService.selectEntpId(dentpInfoSn);
        }
        /*
         * 3. 첨부파일 정보 처리
         */
        if (files != null && !files.isEmpty()) {
            List<CstmrAtchImgListVO> atchList =
                    joinSellerService.parseFileInf(files, true);
            if (atchList != null && !atchList.isEmpty()) {
                CstmrAtchImgListVO atchVO = atchList.get(0);
                /*
                 * 기존 코드의 vo.getDentpInfoSn() 대신
                 * 실제 처리된 dentpInfoSn을 사용한다.
                 */
                atchVO.setDentpInfoSn(dentpInfoSn);
                atchVO.setUpdusrId(loginId);
                atchVO.setRegtrId(loginId);
                joinSellerService.updateFileInfs(atchVO);
            }
        }
        return new JoinSellerTransactionResult(
                dentpInfoSn,
                entpId);
    }
}
```
## 6. Result 객체
Controller와 Transaction Service 사이에 `Map`을 사용하기보다는 작은 Result 객체를 두는 것을 권장합니다.
```java
/**
 * 판매자 기업정보 Transaction 처리 결과.
 *
 * @since 2026.09.02
 * @version 1.0
 */
public class JoinSellerTransactionResult {
    private final long dentpInfoSn;
    private final String entpId;
    public JoinSellerTransactionResult(
            long dentpInfoSn,
            String entpId) {
        this.dentpInfoSn = dentpInfoSn;
        this.entpId = entpId;
    }
    public long getDentpInfoSn() {
        return dentpInfoSn;
    }
    public String getEntpId() {
        return entpId;
    }
}
```
## 7. 수정된 Controller
Controller에서 DB 관련 Service 호출을 모두 제거합니다.
```java
@RequestMapping("/test/insertNewCompany.do")
@ResponseBody
public Map<String, Object> insertNewCompany(
        MultipartHttpServletRequest multiRequest,
        ViewBaseVO view,
        CstmrDentpInfoVO vo) throws Exception {
    CryptoUtil cryptoUtil = new CryptoUtil();
    ModelMap map = new ModelMap();
    Map<String, MultipartFile> files = multiRequest.getFileMap();
    /*
     * 기업정보 기본값 설정
     */
    vo.setEntpId(CommonConstants.MM_SLLR_PREFIX_STR);
    vo.setZip(vo.getCmZip());
    vo.setAdres(vo.getCmAdres());
    vo.setDtlAdres(vo.getCmDtlAdres());
    vo.setEngAdres(vo.getCmEngAdres());
    vo.setEngDtlAdres(vo.getCmEngDtlAdres());
    /*
     * 개인정보 암호화
     */
    if (vo.getRpsntTelno() != null
            && !"".equals(vo.getRpsntTelno())) {
        vo.setRpsntTelno(
                cryptoUtil.encrypt(
                        vo.getRpsntTelno(),
                        "SEED"));
    }
    vo.setRpsntEmail(
            cryptoUtil.encrypt(
                    vo.getRpsntEmail(),
                    "SEED"));
    vo.setRegtrId(view.getLoginId());
    vo.setUpdusrId(view.getLoginId());
    /*
     * ============================================================
     * DB Transaction 처리
     * ============================================================
     *
     * 기존에는 Controller에서 여러 Service를 순차적으로 호출하여
     * 각각 별도의 Transaction으로 Commit될 수 있었다.
     *
     * 변경 후에는 Transaction Service 단 한 번만 호출하고
     * 내부 DB 처리를 하나의 Transaction으로 처리한다.
     */
    JoinSellerTransactionResult result =
            joinSellerTransactionService.saveCompany(
                    vo,
                    view.getMberSn(),
                    view.getLoginId(),
                    files);
    long dentpInfoSn = result.getDentpInfoSn();
    String entpId = result.getEntpId();
    map.addAttribute(
            "dentpInfoSn",
            dentpInfoSn);
    map.addAttribute(
            "atchSn",
            "");
    /*
     * Transaction Service가 정상적으로 Return된 이후에만
     * Session 정보를 변경한다.
     */
    SessionVO sessionVO = sessionService.getSession();
    sessionVO.setIsDmstcEntpMst(true);
    sessionVO.setEntpId(entpId);
    sessionVO.setDentpSn(
            String.valueOf(dentpInfoSn));
    view.setIsMaster(true);
    view.setEntpId(entpId);
    view.setDentpSn(
            String.valueOf(dentpInfoSn));
    return map;
}
```
Controller의 변경 범위는 사실상 아래 부분입니다.
#### 7-1-1. 기존
```java
joinSellerService.insertNewCompany(...);
joinSellerService.selectEntpId(...);
joinSellerService.insertDentpInfoHistory(...);
userInfoService.updateEntpMst(...);
userInfoService.insertCstmrDmstcMberEntpRel(...);
cstmrDmstcDentpDAO.updateCompanyInfomation(...);
joinSellerService.updateFileInfs(...);
```
#### 7-1-2. 변경
```java
JoinSellerTransactionResult result =
        joinSellerTransactionService.saveCompany(
                vo,
                view.getMberSn(),
                view.getLoginId(),
                files);
```
이것이 운영소스 영향도를 가장 낮추는 핵심입니다.
## 8. 변경 후 실제 Transaction 동작
`JoinSellerTransactionService.saveCompany()`가 Proxy를 통과하면:
```text
Controller
    │
    ▼
Transaction Proxy
    │
    ├── BEGIN TX-100
    │
    ▼
JoinSellerTransactionService.saveCompany()
    │
    ├─ joinSellerService.insertNewCompany()
    │      REQUIRED
    │      → TX-100 참여
    │
    ├─ joinSellerService.selectEntpId()
    │      REQUIRED
    │      → TX-100 참여
    │
    ├─ joinSellerService.insertDentpInfoHistory()
    │      REQUIRED
    │      → TX-100 참여
    │
    ├─ userInfoService.updateEntpMst()
    │      REQUIRED
    │      → TX-100 참여
    │
    ├─ userInfoService.insert...Rel()
    │      REQUIRED
    │      → TX-100 참여
    │
    └─ joinSellerService.updateFileInfs()
           REQUIRED
           → TX-100 참여
           │
           ▼
       정상 종료
           │
           ▼
       COMMIT TX-100
```
중간에서:
```java
userInfoService.insertCstmrDmstcMberEntpRel(...)
```
가 Exception을 발생시키면:
```text
insertNewCompany             INSERT
insertDentpInfoHistory       INSERT
updateEntpMst                UPDATE
insertCstmr...Rel            Exception
                              ↓
                         ROLLBACK
                              ↓
기업정보                     취소
History                      취소
회원 Master                  취소
관계정보                     취소
첨부 DB정보                   취소
```
가 됩니다.
## 9. 반드시 확인해야 하는 6가지
### 9-1. 모든 DAO가 같은 `txManager`를 사용해야 함
이것이 가장 중요합니다.
다음 구조라면 정상입니다.
```text
txManager
   │
DataSource-A
   │
├─ JoinSellerDAO
├─ UserInfoDAO
└─ CstmrDmstcDentpDAO
```
그러나 다음처럼 DataSource가 다르면:
```text
txManager-A → DataSource-A → JoinSellerDAO
txManager-B → DataSource-B → UserInfoDAO
```
`REQUIRED`만으로는 두 DB를 하나의 물리 Transaction으로 만들 수 없습니다.
즉:
> **Transaction Service를 추가한다고 해서 서로 다른 TransactionManager까지 자동으로 원자화되는 것은 아닙니다.**
다중 DB라면 JTA/XA 또는 현재 사용하는 복수 DB 트랜잭션 전략을 별도로 검토해야 합니다.
### 9-2. 내부 Service에 `REQUIRES_NEW`가 있으면 안 됨
다음 설정이 숨어 있다면:
```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
```
또는:
```xml
<tx:method
    name="insertXXX"
    propagation="REQUIRES_NEW"/>
```
해당 메소드는 외부 Transaction에서 분리됩니다.
Spring에서 `REQUIRES_NEW`는 기존 Transaction에 참여하지 않고 독립적인 물리 Transaction을 생성하기 때문에 먼저 Commit될 수 있습니다. 
이번 업무범위에서는 모두:
```text
REQUIRED
```
이어야 합니다.
### 9-3. Exception을 잡고 먹어버리면 Rollback 안 됨
아래와 같은 Service가 있으면 안 됩니다.
```java
try {
    dao.insert(...);
} catch (Exception e) {
    LOGGER.error("등록 오류", e);
}
```
상위 Transaction Service 입장에서는:
```text
정상 종료
```
가 되므로 Commit할 수 있습니다.
반드시:
```java
try {
    dao.insert(...);
} catch (Exception e) {
    LOGGER.error("등록 오류", e);
    throw e;
}
```
형태가 되어야 합니다.
현재:
```xml
rollback-for="Exception"
```
이므로 checked Exception까지 Rollback 대상으로 지정되어 있습니다. Spring `<tx:method>`은 기본적으로 RuntimeException에 대해 Rollback하고, `rollback-for`로 checked Exception까지 확대할 수 있습니다. 
### 9-4. Self Invocation에 의존하면 안 됨
다음 구조는 피해야 합니다.
```java
public void methodA() {
    this.methodB();
}
@Transactional
public void methodB() {
}
```
Spring 기본 Transaction AOP는 Proxy 방식이므로 **외부에서 Proxy를 거쳐 들어온 호출만 Transaction Interceptor가 가로챕니다.** 동일 객체 내부 `this.method()` 호출은 별도의 Transaction advice를 적용받지 않습니다. 
따라서 이번 구조처럼:
```text
Controller
 ↓
JoinSellerTransactionService Proxy
 ↓
JoinSellerTransactionService
 ↓
다른 Service Proxy
```
형태가 안전합니다.
### 9-5. 파일 시스템은 DB Transaction 대상이 아님
여기는 특히 주의해야 합니다.
```java
joinSellerService.parseFileInf(files, true);
```
이 메소드가 실제 물리 파일을:
```text
/storage/xxx/image.jpg
```
와 같이 저장한다면 DB Transaction Rollback으로는 파일이 삭제되지 않습니다.
예:
```text
파일 저장 성공
↓
DB INSERT
↓
후속 DB UPDATE 실패
↓
DB ROLLBACK
↓
물리 파일은 남음
```
따라서 `parseFileInf()`가 실제 파일 저장까지 수행한다면 장기적으로는:
```text
1. 임시 파일 저장
2. DB Transaction
3. DB Commit 성공
4. 최종 파일 위치 확정
```
또는 실패 시 compensation:
```java
try {
    ...
} catch (Exception e) {
    fileService.deleteUploadedFile(...);
    throw e;
}
```
방식이 필요합니다.
다만 이것까지 한 번에 변경하면 운영 영향도가 커지므로 **이번 1차 Transaction 개선에서는 DB 원자성을 우선 해결하고 파일 보상처리는 2차 개선으로 분리하는 것**을 권장합니다.
### 9-6. Session은 Commit 성공 후 변경
현재 코드에서 Session 변경:
```java
SessionVO v = sessionService.getSession();
v.setIsDmstcEntpMst(true);
v.setEntpId(setEntpId);
v.setDentpSn(String.valueOf(dentpInfoSn));
```
은 새 Transaction Service **호출 성공 후 Controller에서 그대로 수행하는 것이 적절합니다.**
즉:
```java
result = transactionService.saveCompany(...);
// 여기까지 왔다는 것은 DB Commit 성공
session 변경
```
구조가 좋습니다.
Session을 Transaction Service 내부에서 먼저 변경하면 DB가 Rollback되었는데 Session만 변경되는 상태를 만들 수 있습니다.
## 10. 현재 소스에서 Transaction 이외에 발견되는 문제
이번 수정 시 같이 손보는 것이 좋습니다.
### 10-1. 문제 1. 첨부파일 기업번호
기존:
```java
dentpInfoSn = joinSellerService.insertNewCompany(vo);
...
atchVO.setDentpInfoSn(vo.getDentpInfoSn());
```
`insertNewCompany()`가 `vo` 내부 값까지 변경한다는 보장이 없다면:
```text
dentpInfoSn = 100123
vo.getDentpInfoSn() = 0
```
상태가 될 수 있습니다.
따라서:
```java
atchVO.setDentpInfoSn(dentpInfoSn);
```
으로 변경해야 합니다.
이번 예제에는 수정 반영했습니다.
### 10-2. 문제 2. 수정 경로의 `setEntpId`
현재:
```java
String setEntpId = "";
if (dentpInfoSn == 0) {
    setEntpId = joinSellerService.selectEntpId(dentpInfoSn);
} else {
    ...
}
...
v.setEntpId(setEntpId);
```
수정 경로에서는 `setEntpId`가 계속 `""`일 가능성이 있습니다.
따라서 수정 후에도:
```java
entpId = joinSellerService.selectEntpId(dentpInfoSn);
```
를 통해 실제 값을 가져오는 것이 안전합니다.
### 10-3. 문제 3. Controller에서 DAO 직접 호출
현재:
```java
cstmrDmstcDentpDAO.updateCompanyInfomation(sellerDentpInfoVO);
```
Controller → DAO 직접 호출은 Application Architecture 관점에서 제거해야 합니다.
```text
Controller
 ↓
Service
 ↓
DAO
```
로 통일하는 것이 맞습니다.
이번 예제에서는 최소 변경을 위해 Transaction Service가 DAO를 직접 사용했지만, 2차 리팩터링에서는:
```java
joinSellerService.updateCompanyInfomation(...)
```
을 추가하여:
```text
Controller
 ↓
Transaction Service
 ↓
Business Service
 ↓
DAO
```
구조로 변경하는 것이 더 좋습니다.
## 11. 지금 기존 Transaction 설정 전체를 바꾸지 않는 이유
예를 들어 일괄적으로:
```xml
<tx:method name="*" propagation="SUPPORTS"/>
```
또는 기존 Service Transaction을 제거하는 방법은 추천하지 않습니다.
기존에:
```text
Controller → 기존 Service
```
로 직접 호출하는 수백 개 업무가 있을 수 있기 때문입니다.
기존 Service `REQUIRED`를 제거하면 기존 업무가 Transaction 없이 수행될 위험이 생깁니다.
따라서 단계적으로:

| 단계  | 조치                                | 운영 위험 |
| --- | --------------------------------- | ----: |
| 1차  | 기존 REQUIRED 유지                    |    낮음 |
| 1차  | Transaction Service 신규 추가         |    낮음 |
| 1차  | 문제 Controller만 신규 Service 호출      |    낮음 |
| 2차  | Controller→DAO 직접호출 제거            |    중간 |
| 2차  | 복합 업무를 Transaction Service로 점진 이전 |    낮음 |
| 3차  | 조회 Service read-only/SUPPORTS 최적화 |    중간 |
| 3차  | 기존 광범위 `*Service.*` pointcut 축소   |    중간 |
이 Migration 방식이 운영 시스템에는 가장 안전합니다.
## 12. 적용 후 반드시 해봐야 할 Rollback 테스트
단순히 정상등록만 테스트하면 안 됩니다.
테스트 DB에서 마지막 INSERT 직전에 의도적으로:
```java
throw new RuntimeException("Transaction rollback test");
```
를 넣습니다.
예:
```java
userInfoService.updateEntpMst(mberVO);
/*
 * Transaction Rollback Test
 */
if (true) {
    throw new RuntimeException(
            "Transaction rollback test");
}
userInfoService.insertCstmrDmstcMberEntpRel(relVO);
```
호출 후 다음을 DB에서 확인합니다.
```sql
SELECT *
FROM 기업정보
WHERE DENTP_INFO_SN = ?;
SELECT *
FROM 기업이력
WHERE DENTP_INFO_SN = ?;
SELECT *
FROM 회원기업관계
WHERE DENTP_INFO_SN = ?;
```
**세 테이블 모두 데이터가 없어야 정상입니다.**
테스트 결과:
```text
기업정보       0건
기업History    0건
회원Master     변경 전 값
회원기업관계   0건
첨부DB정보     0건
```
이면 하나의 Transaction으로 정상 통합된 것입니다.
## 13. 최종 권장안
이번 문제에서는 Transaction 설정을 대규모로 변경하지 않는 것이 중요합니다.
```text
                       ┌─────────────────────────────┐
Controller ───────────▶│ JoinSellerTransactionService │
                       │          REQUIRED           │
                       └─────────────┬───────────────┘
                                     │
                  ┌──────────────────┼──────────────────┐
                  ▼                  ▼                  ▼
        JoinSellerService     UserInfoService      DAO
          REQUIRED              REQUIRED
                  │                  │                  │
                  └──────────────────┼──────────────────┘
                                     ▼
                          SAME PHYSICAL TX
                                     │
                        ┌────────────┴────────────┐
                        │                         │
                    SUCCESS                    ERROR
                        │                         │
                      COMMIT                 ROLLBACK
```
제가 적용 우선순위를 잡는다면 **① 신규 `JoinSellerTransactionService` 추가 → ② Controller의 복수 Service/DAO 호출을 신규 Service 한 번 호출로 교체 → ③ 기존 Service의 `REQUIRED`는 그대로 유지 → ④ `REQUIRES_NEW`, 다른 `txManager`, 예외 swallow 여부 점검 → ⑤ 강제 Exception을 넣어 실제 전체 Rollback 검증** 순서로 진행하겠습니다.
이 방법이 현재 운영중인 Spring 5.3 레거시 시스템에서 **변경량·회귀 위험·확장성·트랜잭션 일관성을 동시에 고려했을 때 가장 현실적인 방법**입니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
