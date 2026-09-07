# `updateSeller` Transaction 구조 개선안
현재 메서드는 단순한 회원정보 수정이 아니라 아래 4개 DB 작업이 **하나의 업무 트랜잭션**을 구성합니다.

|순서|처리|현재 호출|
|---:|---|---|
|1|회원 기본정보 수정|`updateCstmrDmstcMberMst()`|
|2|기업 소속 신청정보 등록|`insertCstmrDmstcMberBlngAplyMst()`|
|3|관심 산업정보 저장|`insertIntrstIndust()`|
|4|관심 상품정보 저장|`insertIntrstGoods()`|

따라서 이전 `JoinSellerApplicationService` 방식과 동일하게 **Controller → Application Transaction Service → 기존 개별 Service** 구조로 변경하는 것이 적합합니다.
## 1. 권장 구조
```text
SellerController
    │
    └── SellerUpdateApplicationService   ← Transaction Boundary
            │
            ├── userInfoService.updateCstmrDmstcMberMst()
            ├── userInfoService.insertCstmrDmstcMberBlngAplyMst()
            ├── userInfoService.insertIntrstIndust()
            └── userInfoService.insertIntrstGoods()
```
핵심은 아래입니다.
```text
Controller에는 Transaction을 두지 않는다.
        ↓
SellerUpdateApplicationService의 updateSeller() 하나를 Transaction 단위로 한다.
        ↓
하위 Service에서 Exception 발생 시 반드시 상위로 전달한다.
        ↓
4개 DB 작업 전체 Rollback
```
## 2. Controller 수정
Controller는 요청을 받고 Transaction Service를 호출하는 역할만 남기는 것을 권장합니다.
```java
@Resource(name = "sellerUpdateApplicationService")
private SellerUpdateApplicationService sellerUpdateApplicationService;

/**
 * 판매자 회원정보를 수정한다.
 *
 * @param view 로그인 사용자 정보
 * @param vo 판매자 수정 정보
 * @return 처리 결과
 * @exception Exception 처리 중 발생한 예외
 */
@RequestMapping(value = "/mm/mim/updateSeller.do", method = RequestMethod.POST)
@ResponseBody
public Map<String, Object> updateSeller(
        ViewBaseVO view,
        @RequestBody SellerDataVO vo) throws Exception {

    return sellerUpdateApplicationService.updateSeller(view, vo);
}
```
Controller에서 다음 코드들이 모두 제거됩니다.
```text
CryptoUtil 생성
비밀번호 검증
회원 VO 조립
회원정보 수정
기업소속정보 등록
관심산업 등록
관심상품 등록
결과 Map 조립
```
## 3. `SellerUpdateApplicationService.java`
프로젝트의 기존 `*Service` Transaction AOP를 그대로 적용할 수 있도록 클래스명을 `SellerUpdateApplicationService`로 하는 것을 권장합니다.
```java
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import javax.annotation.Resource;

import org.springframework.stereotype.Service;

/**
 * 판매자 회원정보 수정 Application Service.
 *
 * <p>
 * 판매자 회원정보 수정과 관련된 여러 Service 처리를 하나의
 * Transaction 단위로 관리한다.
 * </p>
 *
 * <pre>
 * << 개정이력(Modification Information) >>
 *
 *   수정일          수정자        수정내용
 *  ----------      --------      ---------------------------
 *  2026.09.07                    최초 생성
 * </pre>
 *
 * @author
 * @since 2026.09.07
 * @version 1.0
 * @see
 */
@Service("sellerUpdateApplicationService")
public class SellerUpdateApplicationService {

    private static final String YES = "Y";

    private static final String NO = "N";

    @Resource(name = "userInfoService")
    private UserInfoService userInfoService;

    /**
     * 판매자 회원정보를 수정한다.
     *
     * <p>
     * 회원정보 수정, 기업소속 신청정보 등록,
     * 관심산업정보 등록, 관심상품정보 등록을
     * 하나의 Transaction으로 처리한다.
     * </p>
     *
     * @param view 로그인 사용자 정보
     * @param vo 판매자 수정 정보
     * @return 처리 결과
     * @exception Exception 처리 중 발생한 예외
     */
    public Map<String, Object> updateSeller(
            ViewBaseVO view,
            SellerDataVO vo) throws Exception {

        /*
         * 1. 기본 요청정보 검증
         */
        validateRequest(view, vo);

        /*
         * 회원번호는 처리 중 반복 변환하지 않고 최초 1회만 변환한다.
         */
        Long dmstcMberSn = Long.valueOf(view.getMberSn());

        /*
         * CryptoUtil은 기존 Bean과의 영향 및 상태 공유를 방지하기 위해
         * 현재 구조에서는 Method 내부에서 생성한다.
         */
        CryptoUtil cryptoUtil = new CryptoUtil();

        /*
         * 2. 회원 기본정보 생성
         */
        CstmrDmstcMberMstVO mberMstVO =
                createMemberMstVO(
                        view,
                        vo,
                        dmstcMberSn,
                        cryptoUtil);

        /*
         * 비밀번호가 사용 불가능한 경우에는
         * DB 변경작업을 시작하기 전에 즉시 종료한다.
         */
        if (mberMstVO == null) {

            Map<String, Object> result =
                    new HashMap<String, Object>();

            result.put(
                    CommonConstants.COMMON_RESULT_CODE,
                    9999);

            result.put(
                    CommonConstants.COMMON_RESULT_MESSAGE,
                    "사용 불가한 비밀번호 입니다. 다른 비밀번호를 입력해 주세요.");

            return result;
        }

        /*
         * ==========================================================
         * 여기부터 실제 DB 변경 작업
         *
         * 아래 모든 처리는 반드시 하나의 Transaction으로 수행되어야 한다.
         * ==========================================================
         */

        /*
         * 3. 회원 기본정보 수정
         */
        Map<String, Object> mberResult =
                userInfoService.updateCstmrDmstcMberMst(
                        mberMstVO);

        /*
         * 4. 기업 소속 신청정보 등록
         */
        insertCompanyApplication(
                view,
                vo,
                mberMstVO,
                dmstcMberSn);

        /*
         * 5. 회원 관심산업정보 등록
         */
        List<CstmrDmstcMberIntrstIndustListVO> industList =
                vo.getIndustList();

        Map<String, Object> industResult =
                userInfoService.insertIntrstIndust(
                        view,
                        industList);

        /*
         * 6. 회원 관심상품정보 등록
         */
        List<CstmrDmstcMberIntrstGoodsListVO> goodsList =
                vo.getGoodsList();

        Map<String, Object> goodsResult =
                userInfoService.insertIntrstGoods(
                        view,
                        goodsList);

        /*
         * 7. Controller 반환결과 생성
         */
        return createResult(
                mberResult,
                industResult,
                goodsResult);
    }

    /**
     * 회원 기본정보 VO를 생성한다.
     *
     * @param view 로그인 사용자 정보
     * @param vo 판매자 수정 정보
     * @param dmstcMberSn 회원 일련번호
     * @param cryptoUtil 암호화 Utility
     * @return 회원 기본정보 VO.
     *         비밀번호 사용이 불가능한 경우 null
     * @exception Exception 암호화 처리 중 발생한 예외
     */
    private CstmrDmstcMberMstVO createMemberMstVO(
            ViewBaseVO view,
            SellerDataVO vo,
            Long dmstcMberSn,
            CryptoUtil cryptoUtil) throws Exception {

        CstmrDmstcMberMstVO mberMstVO =
                new CstmrDmstcMberMstVO();

        /*
         * 정지기업 여부
         */
        String mstYn = NO;

        if (CommonConstants.DENTP_STTS_CD_STOP.equals(
                vo.getEntpSttsCd())) {

            mstYn = YES;
        }

        mberMstVO.setDmstcMberSn(dmstcMberSn);

        /*
         * 비밀번호 변경 요청이 존재하는 경우에만
         * 비밀번호 안전도 검증 및 Hash 처리를 수행한다.
         */
        if (vo.getSellerMemberPwd() != null
                && !"".equals(vo.getSellerMemberPwd())) {

            PwdSaftyLevelUtil pwdLevelUtil =
                    new PwdSaftyLevelUtil();

            String pwdLevel =
                    pwdLevelUtil.getPwdSaftyLevel(
                            vo.getSellerMemberPwd());

            if ("not".equals(pwdLevel)) {
                return null;
            }

            mberMstVO.setMberPwd(
                    cryptoUtil.saltHash512(
                            vo.getSellerMemberPwd()));
        }

        /*
         * 회원 기본정보 설정
         */
        mberMstVO.setEngCstmrName(
                vo.getEngCstmrName());

        mberMstVO.setMobno(
                cryptoUtil.encrypt(
                        vo.getMobno(),
                        "SEED"));

        mberMstVO.setEmail(
                cryptoUtil.encrypt(
                        vo.getEmail(),
                        "SEED"));

        mberMstVO.setTelno(
                cryptoUtil.encrypt(
                        vo.getTelno(),
                        "SEED"));

        mberMstVO.setFaxno(
                vo.getFaxno());

        mberMstVO.setZip(
                vo.getZip());

        mberMstVO.setAdres(
                vo.getAdres());

        mberMstVO.setDtlAdres(
                vo.getDtlAdres());

        mberMstVO.setEngAdres(
                vo.getEngAdres());

        mberMstVO.setEngDtlAdres(
                vo.getEngDtlAdres());

        mberMstVO.setOneIntrstNatCd(
                vo.getOneIntrstNatCd());

        mberMstVO.setTwoIntrstNatCd(
                vo.getTwoIntrstNatCd());

        mberMstVO.setThreeIntrstNatCd(
                vo.getThreeIntrstNatCd());

        mberMstVO.setUpdusrId(
                view.getLoginId());

        /*
         * 2024.01.09
         * 정지기업일 때 프로세스 처리를 위한 추가정보
         */
        mberMstVO.setLoginId(
                view.getLoginId());

        mberMstVO.setEntpMstYn(
                mstYn);

        mberMstVO.setDentpInfoSn(
                vo.getDentpInfoSn());

        mberMstVO.setDeptNm(
                vo.getDeptName());

        mberMstVO.setGradeNm(
                vo.getOfcpsName());

        return mberMstVO;
    }

    /**
     * 기업 소속 신청정보를 등록한다.
     *
     * @param view 로그인 사용자 정보
     * @param vo 판매자 수정 정보
     * @param mberMstVO 회원 기본정보
     * @param dmstcMberSn 회원 일련번호
     * @exception Exception 등록 처리 중 발생한 예외
     */
    private void insertCompanyApplication(
            ViewBaseVO view,
            SellerDataVO vo,
            CstmrDmstcMberMstVO mberMstVO,
            Long dmstcMberSn) throws Exception {

        String aplySttsCd = null;

        /*
         * 기업 대표회원이거나 정지기업 처리 대상인 경우
         */
        if (YES.equals(mberMstVO.getEntpMstYn())
                || YES.equals(vo.getEntpMstYn())) {

            aplySttsCd =
                    CommonConstants.DMSTC_DENTP_BLNG_APLY_MST_30;

        /*
         * 일반 기업소속 신청
         */
        } else if (NO.equals(vo.getEntpMstYn())) {

            aplySttsCd =
                    CommonConstants.DMSTC_DENTP_BLNG_APLY_MST_10;
        }

        /*
         * 소속 신청 처리 대상이 아닌 경우
         */
        if (aplySttsCd == null) {
            return;
        }

        CstmrDmstcMberBlngAplyMstVO companyVO =
                new CstmrDmstcMberBlngAplyMstVO();

        companyVO.setAplySttsCd(
                aplySttsCd);

        companyVO.setDmstcMberSn(
                dmstcMberSn);

        companyVO.setDentpInfoSn(
                vo.getDentpInfoSn());

        companyVO.setDeptName(
                vo.getDeptName());

        companyVO.setOfcpsName(
                vo.getOfcpsName());

        companyVO.setRegtrId(
                view.getLoginId());

        companyVO.setUpdusrId(
                view.getLoginId());

        userInfoService.insertCstmrDmstcMberBlngAplyMst(
                companyVO);
    }

    /**
     * Controller 반환 결과를 생성한다.
     *
     * @param mberResult 회원정보 수정 결과
     * @param industResult 관심산업 수정 결과
     * @param goodsResult 관심상품 수정 결과
     * @return 처리 결과
     */
    private Map<String, Object> createResult(
            Map<String, Object> mberResult,
            Map<String, Object> industResult,
            Map<String, Object> goodsResult) {

        Map<String, Object> result =
                new HashMap<String, Object>();

        result.put(
                "mberResult",
                mberResult);

        result.put(
                "industResult",
                industResult);

        result.put(
                "goodsResult",
                goodsResult);

        return result;
    }

    /**
     * 요청 필수정보를 검증한다.
     *
     * @param view 로그인 사용자 정보
     * @param vo 판매자 수정 정보
     */
    private void validateRequest(
            ViewBaseVO view,
            SellerDataVO vo) {

        if (view == null) {
            throw new IllegalArgumentException(
                    "ViewBaseVO가 존재하지 않습니다.");
        }

        if (vo == null) {
            throw new IllegalArgumentException(
                    "SellerDataVO가 존재하지 않습니다.");
        }

        if (view.getMberSn() == null
                || "".equals(view.getMberSn())) {

            throw new IllegalArgumentException(
                    "회원 일련번호가 존재하지 않습니다.");
        }

        if (view.getLoginId() == null
                || "".equals(view.getLoginId())) {

            throw new IllegalArgumentException(
                    "로그인 사용자 ID가 존재하지 않습니다.");
        }
    }
}
```
## 4. Transaction 설정에서 가장 중요한 부분
26_KTR에서 기존에 사용하던 전역 Transaction 정책이 다음과 같은 형태라면:
```xml
<tx:advice id="txAdvice"
           transaction-manager="transactionManager">
    <tx:attributes>
        <tx:method name="*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
```
그리고 pointcut이 대략 다음처럼 `*Service`를 대상으로 한다면:
```xml
execution(* com.xxx..service..*Service.*(..))
```
이번 클래스명을:
```java
SellerUpdateApplicationService
```
로 지정한 이유가 여기에 있습니다.
`SellerUpdateApplicationService`는 **`Service`로 끝나기 때문에 기존 AOP Transaction 대상에 자연스럽게 포함**됩니다.
따라서 별도로 다음을 중복 적용할 필요가 없습니다.
```java
@Transactional
```
반대로 현재 XML pointcut이 이 신규 클래스를 포함하지 않는 구조라면 반드시 pointcut을 확인해야 합니다.
### Annotation Transaction을 사용하는 프로젝트라면
XML AOP 대신 Annotation 기반이라면 다음과 같이 적용합니다.
```java
@Transactional(
    propagation = Propagation.REQUIRED,
    rollbackFor = Exception.class
)
public Map<String, Object> updateSeller(
        ViewBaseVO view,
        SellerDataVO vo) throws Exception {
```
**XML Transaction과 Annotation Transaction을 이유 없이 중복 적용하는 것은 권장하지 않습니다.**
## 5. 하위 `userInfoService`에서 반드시 확인해야 할 사항
Transaction Service를 추가하는 것만으로 100% 원자성이 보장되는 것은 아닙니다.
다음 사항이 가장 중요합니다.

|검증 항목|정상|문제|
|---|---|---|
|하위 Service propagation|`REQUIRED`|`REQUIRES_NEW`|
|TransactionManager|동일 TransactionManager|서로 다른 TransactionManager|
|DAO Exception|상위로 throw|catch 후 정상 return|
|SQL 오류 처리|Exception 발생|오류 Map만 반환|
|Transaction Service|Spring Bean Proxy 경유|`new SellerUpdateApplicationService()`|
|호출 방향|Controller → ApplicationService|동일 Bean 내부 self invocation|

특히 현재 코드의 하위 메서드들이 `Map<String,Object>`를 반환하는 구조라 다음을 반드시 확인해야 합니다.
```java
try {
    dao.update(...);
} catch (Exception e) {
    result.put("resultCode", 9999);
    return result;
}
```
이런 코드가 존재한다면 **현재 Transaction 구조를 개선해도 Rollback되지 않을 수 있습니다.**
잘못된 형태:
```java
try {
    dao.update(...);
} catch (Exception e) {
    LOGGER.error("처리 실패", e);

    result.put(
            CommonConstants.COMMON_RESULT_CODE,
            9999);

    return result;
}
```
Transaction 관점에서 위 코드는 메서드가 **정상 종료한 것**입니다.
따라서 다음처럼 예외를 상위로 전달해야 합니다.
```java
try {
    dao.update(...);
} catch (Exception e) {
    LOGGER.error("처리 실패", e);
    throw e;
}
```
또는 단순히:
```java
dao.update(...);
```
로 두고 공통 Exception Handler에서 처리하는 것이 더 안전합니다.
## 6. 기존 `ModelMap.addAttribute()` 코드도 수정이 필요
기존:
```java
map.addAttribute(mberResult);
map.addAttribute(industResult);
map.addAttribute(goodsResult);
```
이 방식은 권장하지 않습니다.
`addAttribute(Object)`는 명시적인 key를 지정하지 않고 객체의 convention 이름을 생성합니다. 세 객체가 모두 `Map` 계열이면 이름 충돌 또는 예상하지 못한 JSON 구조가 만들어질 수 있습니다.
최소한 다음처럼 명시해야 합니다.
```java
map.put("mberResult", mberResult);
map.put("industResult", industResult);
map.put("goodsResult", goodsResult);
```
그래서 개선 코드에서는 MVC 객체인 `ModelMap` 자체를 Application Service에서 사용하지 않고:
```java
Map<String, Object>
```
만 사용했습니다.
## 7. 현재 구조와 개선 구조 비교
|항목|현재|개선|
|---|---|---|
|Controller 역할|비즈니스 처리까지 수행|HTTP 요청/응답|
|Transaction Boundary|불명확|`SellerUpdateApplicationService.updateSeller()`|
|회원정보 수정|개별 Service|통합 Transaction|
|기업소속 신청|개별 Service|통합 Transaction|
|관심산업|개별 Service|통합 Transaction|
|관심상품|개별 Service|통합 Transaction|
|중간 실패|부분 Commit 가능|전체 Rollback|
|비밀번호 검증|Controller|Application Service|
|암호화|Controller|Application Service|
|중복 `Long.parseLong()`|존재|1회|
|Null-safe `"Y"` 비교|미흡|`YES.equals()`|
|`ModelMap` 의존|Controller 비즈니스 로직에 혼재|제거|
|테스트 용이성|낮음|Application Service 단위 테스트 가능|
## 8. 실제 Rollback 시나리오
예를 들어:
```text
① 회원정보 UPDATE                  성공
② 기업소속 신청 INSERT             성공
③ 관심산업 DELETE/INSERT           성공
④ 관심상품 INSERT                  실패
                                  ↓
                              SQLException
                                  ↓
                      SellerUpdateApplicationService
                                  ↓
                         Transaction Rollback
```
최종 DB:
```text
① 회원정보 UPDATE          → ROLLBACK
② 기업소속 신청 INSERT     → ROLLBACK
③ 관심산업 변경            → ROLLBACK
④ 관심상품 INSERT          → ROLLBACK
```
이것이 이번 수정의 핵심 목적입니다.


# `updateSeller` 하위 Service까지 포함한 Transaction 재검토
제공한 실제 하위 메서드까지 확인하면, 이전 `JoinSellerApplicationService.saveCompany()`에서 적용했던 **Controller → Application Service(Transaction Boundary) → 기존 Business Service → DAO** 구조를 그대로 적용하는 것이 적절합니다. 다만 단순히 상위 Transaction Service만 추가해서는 부족하고, 아래 사항은 같이 수정해야 합니다. 
## 1. 현재 코드의 핵심 문제
|우선순위|문제|현재 영향|조치|
|---:|---|---|---|
|**1**|`updateCstmrDmstcMberMst()`가 `cnt == 0`이어도 Exception을 발생시키지 않음|회원정보 수정 실패 후 나머지 INSERT가 실행되어 Commit 가능|실패를 Exception으로 전환|
|**2**|`bkAlrm()`이 전체 업무 완료 전에 호출됨|관심상품 저장 실패로 DB Rollback돼도 알림은 이미 발송될 수 있음|DB 전체 성공 후 처리, 외부발송이면 AFTER_COMMIT|
|**3**|관심산업/상품 `DELETE → INSERT`가 각각 독립 호출|현재 Transaction 경계에 따라 부분 Commit 가능|상위 Service의 하나의 `REQUIRED`로 통합|
|**4**|`insertIntrstGoods()`가 `"industMsg"` 반환|관심산업 결과와 key 충돌|`goodsMsg`로 수정|
|**5**|`vo.getEntpMstYn().equals("Y")`|null이면 NPE|`"Y".equals(...)`|
|**6**|`String.valueOf(long)`의 empty 검사|primitive `long`은 빈 문자열이 될 수 없어 실질적으로 의미 없음|유효범위 검사 또는 제거|
|**7**|관심 목록 null 처리 불명확|null이면 NPE Rollback|`null`과 empty의 업무 의미를 명확히 구분|
|**8**|Controller `ModelMap.addAttribute(Map)` 반복|동일 convention key 충돌 가능|명시적인 결과 Map 사용|
## 2. Transaction 관점에서 가장 중요한 문제
현재:
```java
cnt = cstmrDmstcMberMstDAO.updateCstmrDmstcMberMst(vo);
if(cnt > 0) {
    msg = "S";
} else {
    msg = "E";
}
result.put("msg", msg);
return result;
```
`cnt == 0`이어도 메서드가 **정상 return**합니다.
Spring Transaction은 이를 실패로 판단하지 않습니다.
따라서 현재 구조에 단순히 Application Service를 추가하면:
```text
회원정보 UPDATE → cnt = 0
        ↓
"msg" = "E" 정상 return
        ↓
기업소속 INSERT
        ↓
관심산업 DELETE/INSERT
        ↓
관심상품 DELETE/INSERT
        ↓
Application Service 정상 종료
        ↓
COMMIT
```
가 가능합니다.
기존 코드가 이미 `cnt > 0`을 성공 기준으로 사용하고 있으므로 동일 계약을 유지한다면 실패 시 반드시 예외를 발생시켜야 합니다.
```java
if (cnt <= 0) {
    throw new IllegalStateException("회원정보 수정에 실패하였습니다.");
}
```
> 단, 해당 DAO의 UPDATE count가 프로젝트 DB/JDBC 설정상 "변경된 행"인지 "매칭된 행"인지 한번 확인하는 것이 좋습니다. 현재 기존 코드 자체가 `cnt > 0`을 성공 조건으로 사용하고 있으므로 이번 개선에서는 기존 의미를 그대로 유지하는 것이 변경 영향도가 가장 작습니다.
# 3. 최종 권장 구조
```text
SellerController
        │
        ▼
SellerUpdateApplicationService.updateSeller()
        │
        │  ← REQUIRED Transaction 시작
        │
        ├─ userInfoService.updateCstmrDmstcMberMst()
        │
        ├─ userInfoService.insertCstmrDmstcMberBlngAplyMst()
        │
        ├─ userInfoService.insertIntrstIndust()
        │
        └─ userInfoService.insertIntrstGoods()
        │
        ▼
모두 성공
        │
        ▼
COMMIT
```
중간 하나라도 Exception이면:
```text
                     Exception
                         │
                         ▼
회원정보 UPDATE       ─────┐
기업소속 INSERT       ─────┤
관심산업 DELETE/INSERT─────┤ → ROLLBACK
관심상품 DELETE/INSERT─────┘
```
# 4. Controller 최종 형태
Controller에는 Transaction 관련 비즈니스 로직을 남기지 않습니다.
```java
@Resource(name = "sellerUpdateApplicationService")
private SellerUpdateApplicationService sellerUpdateApplicationService;
/**
 * 판매자 회원정보를 수정한다.
 *
 * @param view 로그인 사용자 정보
 * @param vo 판매자 수정 정보
 * @return 처리 결과
 * @exception Exception 처리 중 발생한 예외
 */
@RequestMapping(
        value = "/mm/mim/updateSeller.do",
        method = RequestMethod.POST)
@ResponseBody
public Map<String, Object> updateSeller(
        ViewBaseVO view,
        @RequestBody SellerDataVO vo) throws Exception {
    return sellerUpdateApplicationService.updateSeller(view, vo);
}
```
# 5. `SellerUpdateApplicationService`
현재 프로젝트의 `JoinSellerApplicationService`와 동일하게 Application Service가 Transaction Boundary가 되도록 구성합니다.
```java
/**
 * 판매자 회원정보 수정 Application Service.
 *
 * <pre>
 * << 개정이력(Modification Information) >>
 *
 *   수정일          수정자        수정내용
 *  ----------      --------      ---------------------------
 *  2026.09.07                    최초 생성
 * </pre>
 *
 * @author
 * @since 2026.09.07
 * @version 1.0
 * @see
 */
@Service("sellerUpdateApplicationService")
public class SellerUpdateApplicationService {
    private static final String YES = "Y";
    private static final String NO = "N";
    private static final String SUCCESS = "S";
    @Resource(name = "userInfoService")
    private UserInfoService userInfoService;
    /**
     * 판매자 회원정보 수정 전체 업무를 하나의 Transaction으로 처리한다.
     *
     * <p>
     * 처리 순서
     * 1. 요청정보 검증
     * 2. 회원 기본정보 수정
     * 3. 기업 소속 신청정보 등록
     * 4. 관심산업정보 재등록
     * 5. 관심상품정보 재등록
     * </p>
     *
     * @param view 로그인 사용자 정보
     * @param vo 판매자 수정 정보
     * @return 처리 결과
     * @exception Exception 처리 중 발생한 예외
     */
    public Map<String, Object> updateSeller(
            ViewBaseVO view,
            SellerDataVO vo) throws Exception {
        validateRequest(view, vo);
        CryptoUtil cryptoUtil = new CryptoUtil();
        /*
         * DB 변경 이전에 회원 수정 데이터를 생성한다.
         */
        CstmrDmstcMberMstVO mberMstVO =
                createMemberMstVO(view, vo, cryptoUtil);
        /*
         * 비밀번호 사용 가능 여부 확인.
         *
         * 비밀번호 검증 실패는 DB 오류가 아니므로
         * DB 변경작업 전에 정상 응답으로 종료한다.
         */
        if (mberMstVO == null) {
            return createPasswordErrorResult();
        }
        /*
         * ====================================================
         * 여기부터 하나의 Transaction으로 처리한다.
         * ====================================================
         */
        Map<String, Object> mberResult =
                userInfoService.updateCstmrDmstcMberMst(mberMstVO);
        /*
         * 회원정보 수정 실패가 Map 형태로 전달되는 기존 구현에 대한
         * 이중 안전장치이다.
         *
         * 하위 Service에서도 Exception을 발생시키도록 수정하는 것이
         * 기본 원칙이다.
         */
        if (!SUCCESS.equals(mberResult.get("msg"))) {
            throw new IllegalStateException(
                    "판매자 회원정보 수정에 실패하였습니다.");
        }
        /*
         * 기업 소속 신청정보 등록
         */
        insertCompanyApplication(
                view,
                vo,
                mberMstVO);
        /*
         * 관심산업 정보 저장
         */
        Map<String, Object> industResult =
                userInfoService.insertIntrstIndust(
                        view,
                        vo.getIndustList());
        /*
         * 관심상품 정보 저장
         */
        Map<String, Object> goodsResult =
                userInfoService.insertIntrstGoods(
                        view,
                        vo.getGoodsList());
        /*
         * 기존 Ajax 응답 영향도를 최소화하기 위하여
         * 결과 Map을 하나의 Map으로 병합한다.
         */
        Map<String, Object> result =
                new HashMap<String, Object>();
        result.putAll(mberResult);
        result.putAll(industResult);
        result.putAll(goodsResult);
        return result;
    }
    /**
     * 회원 수정용 VO를 생성한다.
     *
     * @param view 로그인 사용자 정보
     * @param vo 판매자 수정 정보
     * @param cryptoUtil 암호화 Utility
     * @return 회원 수정정보
     * @exception Exception 암호화 처리 중 발생한 예외
     */
    private CstmrDmstcMberMstVO createMemberMstVO(
            ViewBaseVO view,
            SellerDataVO vo,
            CryptoUtil cryptoUtil) throws Exception {
        CstmrDmstcMberMstVO mberMstVO =
                new CstmrDmstcMberMstVO();
        String mstYn = NO;
        if (CommonConstants.DENTP_STTS_CD_STOP.equals(
                vo.getEntpSttsCd())) {
            mstYn = YES;
        }
        mberMstVO.setDmstcMberSn(
                Long.valueOf(view.getMberSn()));
        /*
         * 비밀번호가 입력된 경우에만 안전도 및 Hash 처리
         */
        if (vo.getSellerMemberPwd() != null
                && !"".equals(vo.getSellerMemberPwd())) {
            PwdSaftyLevelUtil pwdLevelUtil =
                    new PwdSaftyLevelUtil();
            String pwdLevel =
                    pwdLevelUtil.getPwdSaftyLevel(
                            vo.getSellerMemberPwd());
            if ("not".equals(pwdLevel)) {
                return null;
            }
            mberMstVO.setMberPwd(
                    cryptoUtil.saltHash512(
                            vo.getSellerMemberPwd()));
        }
        mberMstVO.setEngCstmrName(
                vo.getEngCstmrName());
        mberMstVO.setMobno(
                cryptoUtil.encrypt(
                        vo.getMobno(),
                        "SEED"));
        mberMstVO.setEmail(
                cryptoUtil.encrypt(
                        vo.getEmail(),
                        "SEED"));
        mberMstVO.setTelno(
                cryptoUtil.encrypt(
                        vo.getTelno(),
                        "SEED"));
        mberMstVO.setFaxno(
                vo.getFaxno());
        mberMstVO.setZip(
                vo.getZip());
        mberMstVO.setAdres(
                vo.getAdres());
        mberMstVO.setDtlAdres(
                vo.getDtlAdres());
        mberMstVO.setEngAdres(
                vo.getEngAdres());
        mberMstVO.setEngDtlAdres(
                vo.getEngDtlAdres());
        mberMstVO.setOneIntrstNatCd(
                vo.getOneIntrstNatCd());
        mberMstVO.setTwoIntrstNatCd(
                vo.getTwoIntrstNatCd());
        mberMstVO.setThreeIntrstNatCd(
                vo.getThreeIntrstNatCd());
        mberMstVO.setUpdusrId(
                view.getLoginId());
        /*
         * 2024.01.09
         * 정지기업일 때 프로세스 처리를 위한 추가
         */
        mberMstVO.setLoginId(
                view.getLoginId());
        mberMstVO.setEntpMstYn(
                mstYn);
        mberMstVO.setDentpInfoSn(
                vo.getDentpInfoSn());
        mberMstVO.setDeptNm(
                vo.getDeptName());
        mberMstVO.setGradeNm(
                vo.getOfcpsName());
        return mberMstVO;
    }
    /**
     * 기업 소속 신청정보를 등록한다.
     *
     * @param view 로그인 사용자 정보
     * @param vo 판매자 수정 정보
     * @param mberMstVO 회원 수정정보
     * @exception Exception 등록 처리 중 발생한 예외
     */
    private void insertCompanyApplication(
            ViewBaseVO view,
            SellerDataVO vo,
            CstmrDmstcMberMstVO mberMstVO) throws Exception {
        String aplySttsCd = null;
        if (YES.equals(mberMstVO.getEntpMstYn())
                || YES.equals(vo.getEntpMstYn())) {
            aplySttsCd =
                    CommonConstants.DMSTC_DENTP_BLNG_APLY_MST_30;
        } else if (NO.equals(vo.getEntpMstYn())) {
            aplySttsCd =
                    CommonConstants.DMSTC_DENTP_BLNG_APLY_MST_10;
        }
        /*
         * 기업소속 신청 대상이 아닐 경우 처리하지 않는다.
         */
        if (aplySttsCd == null) {
            return;
        }
        CstmrDmstcMberBlngAplyMstVO companyVO =
                new CstmrDmstcMberBlngAplyMstVO();
        companyVO.setAplySttsCd(
                aplySttsCd);
        companyVO.setDmstcMberSn(
                Long.valueOf(view.getMberSn()));
        companyVO.setDentpInfoSn(
                vo.getDentpInfoSn());
        companyVO.setDeptName(
                vo.getDeptName());
        companyVO.setOfcpsName(
                vo.getOfcpsName());
        companyVO.setRegtrId(
                view.getLoginId());
        companyVO.setUpdusrId(
                view.getLoginId());
        userInfoService.insertCstmrDmstcMberBlngAplyMst(
                companyVO);
    }
    /**
     * 요청정보를 검증한다.
     *
     * @param view 로그인 사용자 정보
     * @param vo 판매자 수정 정보
     */
    private void validateRequest(
            ViewBaseVO view,
            SellerDataVO vo) {
        if (view == null) {
            throw new IllegalArgumentException(
                    "로그인 사용자 정보가 존재하지 않습니다.");
        }
        if (vo == null) {
            throw new IllegalArgumentException(
                    "판매자 수정정보가 존재하지 않습니다.");
        }
        if (view.getMberSn() == null
                || "".equals(view.getMberSn())) {
            throw new IllegalArgumentException(
                    "회원 일련번호가 존재하지 않습니다.");
        }
        if (view.getLoginId() == null
                || "".equals(view.getLoginId())) {
            throw new IllegalArgumentException(
                    "로그인 사용자 ID가 존재하지 않습니다.");
        }
        /*
         * null과 empty는 서로 다른 의미로 관리한다.
         *
         * empty List : 모든 관심항목 삭제
         * null       : 요청 데이터 오류
         */
        if (vo.getIndustList() == null) {
            throw new IllegalArgumentException(
                    "관심산업 정보가 존재하지 않습니다.");
        }
        if (vo.getGoodsList() == null) {
            throw new IllegalArgumentException(
                    "관심상품 정보가 존재하지 않습니다.");
        }
    }
    /**
     * 비밀번호 검증 오류 결과를 생성한다.
     *
     * @return 오류 결과
     */
    private Map<String, Object> createPasswordErrorResult() {
        Map<String, Object> result =
                new HashMap<String, Object>();
        result.put(
                CommonConstants.COMMON_RESULT_CODE,
                9999);
        result.put(
                CommonConstants.COMMON_RESULT_MESSAGE,
                "사용 불가한 비밀번호 입니다. 다른 비밀번호를 입력해 주세요.");
        return result;
    }
}
```
# 6. `updateCstmrDmstcMberMst()` 반드시 수정
이 메서드는 이번 Transaction 개선에서 가장 중요합니다.
## 수정안
```java
public Map<String, Object> updateCstmrDmstcMberMst(
        CstmrDmstcMberMstVO vo) throws Exception {
    int cnt = 0;
    int checkCnt = 0;
    Map<String, Object> result =
            new HashMap<String, Object>();
    CstmrDentpInfoVO cstmrDentpInfoVO =
            new CstmrDentpInfoVO();
    /*
     * 기업 대표회원 처리
     */
    if ("Y".equals(vo.getEntpMstYn())) {
        long dentpInfoSn =
                vo.getDentpInfoSn();
        long dmstcMberSn =
                vo.getDmstcMberSn();
        String loginId =
                vo.getLoginId();
        checkCnt =
                joinSellerDAO.checkSellerCompany(
                        dentpInfoSn);
        if (checkCnt == 0) {
            /*
             * primitive long을 String으로 변환하여
             * Empty 여부를 확인하는 기존 로직은 제거한다.
             *
             * dentpInfoSn == 0이 업무적으로 유효하지 않다면
             * 별도의 값 검증을 추가하는 것이 맞다.
             */
            cstmrDentpInfoVO.setEntpSttsCd(
                    CommonConstants.DENTP_STTS_CD_NORMAL);
            cstmrDentpInfoVO.setDentpInfoSn(
                    dentpInfoSn);
            joinSellerDAO.updateCompanyState(
                    cstmrDentpInfoVO);
            employeeDAO.updateEntpMst(
                    dmstcMberSn,
                    loginId);
            vo.setDmstcMberGbnCd(
                    CommonConstants.DMSTC_GBN_CD_ENTP_BLNG);
        } else {
            /*
             * 이미 기업 대표회원이 존재할 경우
             */
            vo.setEntpMstYn("N");
        }
        /*
         * 기업-회원 관계정보 등록
         */
        CstmrDmstcMberEntpRelVO cstmrDmstcMberEntpRelVO =
                new CstmrDmstcMberEntpRelVO();
        cstmrDmstcMberEntpRelVO.setDmstcMberSn(
                dmstcMberSn);
        cstmrDmstcMberEntpRelVO.setDentpInfoSn(
                dentpInfoSn);
        cstmrDmstcMberEntpRelVO.setAfflcoMberSttsCd(
                CommonConstants.AFFLCO_MBER_STTS_CD_10);
        cstmrDmstcMberEntpRelVO.setDeptName(
                vo.getDeptNm());
        cstmrDmstcMberEntpRelVO.setOfcpsName(
                vo.getGradeNm());
        cstmrDmstcMberEntpRelVO.setResgnYn(
                "N");
        cstmrDmstcMberEntpRelVO.setDelYn(
                "N");
        cstmrDmstcMberEntpRelVO.setUseYn(
                "Y");
        cstmrDmstcMberEntpRelVO.setRegtrId(
                loginId);
        cstmrDmstcMberEntpRelVO.setUpdusrId(
                loginId);
        cstmrDmstcMberEntpRelDAO
                .insertCstmrDmstcMberEntpRel(
                        cstmrDmstcMberEntpRelVO);
        /*
         * 기업 ID 설정
         */
        String setEntpId =
                joinSellerDAO.selectEntpId(
                        dentpInfoSn);
        vo.setEntpId(
                setEntpId);
    }
    /*
     * 회원정보 수정
     */
    cnt =
            cstmrDmstcMberMstDAO
                    .updateCstmrDmstcMberMst(vo);
    /*
     * 중요:
     *
     * 실패를 Map의 "E"로 반환하면 상위 Transaction이
     * 실패 사실을 알 수 없기 때문에 Rollback되지 않는다.
     */
    if (cnt <= 0) {
        throw new IllegalStateException(
                "회원정보 수정에 실패하였습니다."
                + " dmstcMberSn="
                + vo.getDmstcMberSn());
    }
    result.put(
            "msg",
            "S");
    return result;
}
```
### 기존 `bkAlrm()`은 여기서 제거
기존:
```java
if(cnt > 0) {
    msg = "S";
    commAlrmService.bkAlrm(
            null,
            null,
            CommonConstants.MM_MAIL_SLLR_MM07,
            vo.getLoginId(),
            "S");
}
```
이 위치에서는:
```text
회원 UPDATE 성공
    ↓
알림 발송
    ↓
관심산업 성공
    ↓
관심상품 실패
    ↓
DB 전체 ROLLBACK
```
이 되어 **DB에는 수정 내용이 없는데 성공 알림은 이미 사용자에게 전달되는 문제**가 생길 수 있습니다.
따라서 `bkAlrm()`의 내부 구현을 별도로 확인해야 합니다.

|`bkAlrm()` 구현|처리 방법|
|---|---|
|동일 DB의 알림 테이블 INSERT만 수행|같은 Transaction에 포함 가능|
|메일/SMS/외부 API 직접 전송|반드시 Commit 이후 실행 권장|
|DB 등록 + 외부 전송 혼합|DB 등록과 실제 전송 분리 권장|
외부 메시지 발송이라면 Spring의 `AFTER_COMMIT` 방식이 가장 안전합니다.
# 7. 기업소속 신청 Service
이 메서드는 현재 구조상 큰 문제는 없습니다.
```java
public long insertCstmrDmstcMberBlngAplyMst(
        CstmrDmstcMberBlngAplyMstVO vo) throws Exception {
    return cstmrDmstcMberBlngAplyMstDAO
            .insertCstmrDmstcMberBlngAplyMst(vo);
}
```
DAO에서 SQL Exception이 발생하면 상위로 그대로 전달되므로:
```text
insert 실패
    ↓
Exception
    ↓
Application Service
    ↓
전체 Rollback
```
이 가능합니다.
불필요하게:
```java
try {
    ...
} catch(Exception e) {
    return 0;
}
```
같은 처리는 추가하면 안 됩니다.
# 8. `insertIntrstIndust()` 수정
현재 `DELETE → INSERT` 방식은 Transaction만 제대로 묶이면 문제가 없습니다.
```java
public Map<String, Object> insertIntrstIndust(
        ViewBaseVO view,
        List<CstmrDmstcMberIntrstIndustListVO> industListVO)
        throws Exception {
    Map<String, Object> result =
            new HashMap<String, Object>();
    long dmstcMberSn =
            Long.parseLong(view.getMberSn());
    /*
     * 기존 관심산업정보 삭제
     */
    cstmrDmstcMberMstDAO.deleteIntrstIndust(
            dmstcMberSn);
    /*
     * 신규 관심산업정보 등록
     */
    for (CstmrDmstcMberIntrstIndustListVO vo
            : industListVO) {
        /*
         * CstmrDmstcMberIntrstIndustListVO에 회원번호가
         * 존재한다면 Client 값을 신뢰하지 않고
         * 아래처럼 서버 세션의 회원번호로 설정하는 것을 권장한다.
         *
         * vo.setDmstcMberSn(dmstcMberSn);
         */
        vo.setRegtrId(
                view.getLoginId());
        vo.setUpdusrId(
                view.getLoginId());
        cstmrDmstcMberMstDAO.insertIntrstIndust(
                vo);
    }
    result.put(
            "industMsg",
            "S");
    return result;
}
```
중간 INSERT에서 오류가 발생하면:
```text
DELETE
INSERT #1
INSERT #2
INSERT #3 → SQLException
              ↓
           ROLLBACK
```
따라서 기존 관심산업도 복원됩니다.
# 9. `insertIntrstGoods()` 수정
여기에는 실제 오류가 하나 있습니다.
현재:
```java
result.put("industMsg", "S");
```
관심상품인데 `industMsg`입니다.
다음으로 수정하는 것이 맞습니다.
```java
public Map<String, Object> insertIntrstGoods(
        ViewBaseVO view,
        List<CstmrDmstcMberIntrstGoodsListVO> goodsListVO)
        throws Exception {
    Map<String, Object> result =
            new HashMap<String, Object>();
    long dmstcMberSn =
            Long.parseLong(view.getMberSn());
    /*
     * 기존 관심상품정보 삭제
     */
    cstmrDmstcMberMstDAO.deleteIntrstGoods(
            dmstcMberSn);
    /*
     * 신규 관심상품정보 등록
     */
    for (CstmrDmstcMberIntrstGoodsListVO vo
            : goodsListVO) {
        /*
         * Client에서 전달받은 회원번호를 사용하지 않고
         * 로그인 Session의 회원번호를 사용한다.
         */
        vo.setDmstcMberSn(
                dmstcMberSn);
        vo.setRegtrId(
                view.getLoginId());
        vo.setUpdusrId(
                view.getLoginId());
        cstmrDmstcMberMstDAO.insertIntrstGoods(
                vo);
    }
    result.put(
            "goodsMsg",
            "S");
    return result;
}
```
단, 기존 JSP/AJAX에서 실제로:
```javascript
result.industMsg
```
만 사용하고 있는지는 확인해야 합니다. `goodsMsg`를 새로 추가하는 것이 논리적으로 맞지만 기존 화면과의 호환성은 별도 확인 대상입니다.
# 10. `checkSellerCompany()`에는 Transaction 외의 동시성 문제가 있음
다음 로직도 실무 관점에서는 주의해야 합니다.
```java
checkCnt = joinSellerDAO.checkSellerCompany(dentpInfoSn);
if (checkCnt == 0) {
    ...
    employeeDAO.updateEntpMst(...);
}
```
두 요청이 동시에 들어오면:
```text
Transaction A                     Transaction B
------------------------------------------------
checkSellerCompany = 0
                                  checkSellerCompany = 0
updateEntpMst()
                                  updateEntpMst()
INSERT relation
                                  INSERT relation
```
가 가능할 수 있습니다.
즉:
> **Transaction을 하나로 묶는 것과 동시성 제어는 별개의 문제입니다.**
기업별 대표 회원이 반드시 하나만 존재해야 하는 업무라면 최종 방어선은 DB에 있어야 합니다.
권장 우선순위:
```text
1. 가능한 경우 DB UNIQUE 제약조건
2. 필요 시 SELECT ... FOR UPDATE
3. Application 레벨 check만으로 유일성 보장하지 않음
```
다만 이 부분은 이번 Transaction 리팩터링과 별개의 수정이므로 현재 변경 범위를 최소화하려면 별도 개선 대상으로 두어도 됩니다.
# 11. 기존 Transaction AOP에서 확인할 부분
이전 `JoinSellerApplicationService`와 동일하게 기존 XML Transaction 설정이 예를 들어:
```xml
<tx:advice id="txAdvice"
           transaction-manager="transactionManager">
    <tx:attributes>
        <tx:method name="*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
```
이고:
```xml
<aop:pointcut
    id="servicePointcut"
    expression="execution(* app..service..*Service.*(..))"/>
```
와 같은 구조라면:
```java
SellerUpdateApplicationService
```
는 해당 AOP 대상에 포함되게 해야 합니다.
최종적으로 실제 실행 시 반드시:
```text
SellerUpdateApplicationService.updateSeller()
    REQUIRED TX 시작
        │
        ├─ userInfoService.update...
        ├─ userInfoService.insert...
        ├─ userInfoService.insertIntrstIndust...
        └─ userInfoService.insertIntrstGoods...
    COMMIT
```
가 되어야 합니다.
하위 Service의 Transaction이 `REQUIRED`라면 동일 Transaction에 참여하므로 문제없습니다.
반대로 아래가 있으면 반드시 점검해야 합니다.
```java
PROPAGATION_REQUIRES_NEW
```
특히 관심산업/관심상품 Service가 `REQUIRES_NEW`이면 이번 목적에 맞지 않습니다.
# 12. 권장 최종 Transaction 성공/실패 조건
Application Service에서는 **성공을 return 값으로 판단하기보다, 실패 시 Exception이라는 계약으로 통일하는 것**이 가장 안정적입니다.
현재:
```text
성공 → Map "S"
실패 → Map "E"
```
장기적으로는:
```text
성공 → 정상 return
실패 → Exception
```
으로 만드는 것이 좋습니다.
즉 이상적인 내부 Service는:
```java
public void updateCstmrDmstcMberMst(...) throws Exception {
    int cnt = dao.update(...);
    if (cnt <= 0) {
        throw new IllegalStateException(...);
    }
}
```
입니다.
`Map<String,Object>`는 DB Service가 아니라 **최종 Controller 응답 단계에서만 생성**하는 것이 Application Architecture상 더 적절합니다.
# 13. 이번 수정의 최종 판정

|항목|판정|
|---|---|
|Application Service 신설|**필수**|
|Controller Transaction 제거|**권장**|
|4개 작업 단일 `REQUIRED`|**필수**|
|`cnt == 0` Exception 처리|**필수**|
|하위 Service Exception swallow 금지|**필수**|
|관심산업 DELETE/INSERT 구조|Transaction 내에서는 사용 가능|
|관심상품 DELETE/INSERT 구조|Transaction 내에서는 사용 가능|
|`goodsMsg` 오타|**수정 권장**|
|`bkAlrm` 현 위치|**재검토 필수**|
|`checkSellerCompany` 동시성|추가 개선 대상|
|`ModelMap.addAttribute(Map)`|명시적 Map 반환으로 변경 권장|

## 결론
이번 메서드는 이전 `JoinSellerApplicationService`와 **동일한 Application Service Transaction Facade 방식으로 변경하는 것이 정확합니다.**
다만 이번 소스에서는 특히 다음 두 가지를 빠뜨리면 안 됩니다.
```java
// ① 실패를 반드시 Exception으로 전달
if (cnt <= 0) {
    throw new IllegalStateException(
            "회원정보 수정에 실패하였습니다.");
}
```
그리고:
```text
② commAlrmService.bkAlrm()
   → 전체 DB 처리 완료 전 호출하지 않음
   → 실제 외부 알림이면 AFTER_COMMIT으로 분리
```
이 두 부분까지 처리해야 `updateSeller`를 **회원정보·기업관계·소속신청·관심산업·관심상품 전체가 Commit되거나 전체가 Rollback되는 실무 수준의 단일 Transaction**으로 볼 수 있습니다.