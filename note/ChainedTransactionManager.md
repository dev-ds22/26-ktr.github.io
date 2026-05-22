
## ChainedTransactionManager에 txManager, txManagerMail 등록
```xml
<tx:advice id="txAdvice" transaction-manager="transactionManager">
    <tx:attributes>
        <tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>

<aop:config>
    <aop:pointcut id="requiredTx" expression="execution(* aa.app..service.*Service.*(..))"/>
    <aop:advisor advice-ref="txAdvice" pointcut-ref="requiredTx" />
</aop:config>
    
<aop:config>
    <aop:pointcut id="requiredApiTx" expression="execution(* aa.api..service.*Service.*(..))"/>
    <aop:advisor advice-ref="txAdvice" pointcut-ref="requiredApiTx" />
</aop:config>

<bean id="transactionManager" class="org.springframework.data.transaction.ChainedTransactionManager" > 

    <constructor-arg>
        <list>
            <ref bean="txManager"/>
            <ref bean="txManagerMail"/>
        </list>
    </constructor-arg>
</bean>

<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
    
<bean id="txManagerMail" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSourceMail"/>
</bean>
```


`ChainedTransactionManager`에 `txManager`, `txManagerMail`을 등록하면 **하나의 `@Transactional` 경계 안에서 두 개의 `PlatformTransactionManager`를 순차적으로 제어**할 수는 있습니다.  
하지만 이것은 **DB 관점의 완전한 단일 트랜잭션**이나 **XA/2PC 분산 트랜잭션**이 아닙니다.  
즉, 표현을 정확히 하면 다음과 같습니다.

> “두 TransactionManager를 하나의 Spring 트랜잭션 경계처럼 묶어 순차 commit/rollback 하게 만드는 best-effort 방식이다. 단, 한쪽 commit 성공 후 다른 쪽 commit 실패 시 부분 commit이 발생할 수 있다.”  
> Spring Data Commons 공식 API에서도 `ChainedTransactionManager`는 여러 delegate transaction manager의 생성, commit, rollback을 조율하지만, 부분 commit으로 인한 불일치 상태를 애플리케이션이 감당하거나 복구할 수 있을 때만 사용해야 하며, 그 외에는 사용을 권장하지 않는다고 설명합니다. 또한 Spring Data Commons 2.5부터 deprecated 상태입니다. ([Home](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/transaction/ChainedTransactionManager.html "ChainedTransactionManager (Spring Data Core 4.0.5 API)"))

### 1. `ChainedTransactionManager`의 역할

| 구분    | 설명                                                                                                                 |
| ----- | ------------------------------------------------------------------------------------------------------------------ |
| 소속    | Spring Framework core가 아니라 `spring-data-commons`의 `org.springframework.data.transaction.ChainedTransactionManager` |
| 목적    | 여러 `PlatformTransactionManager`를 하나의 manager처럼 묶어 실행                                                               |
| 방식    | 등록 순서대로 transaction 시작, 역순으로 commit/rollback                                                                       |
| 보장 수준 | best-effort 수준                                                                                                     |
| 한계    | XA/2PC가 아니므로 atomic commit 보장 불가                                                                                   |
| 상태    | Spring Data Commons 2.5부터 deprecated                                                                               |

- 공식 문서 기준 동작은 다음과 같습니다. 등록된 transaction manager들은 **지정한 순서대로 트랜잭션을 시작**하고, **commit/rollback은 역순으로 수행**됩니다. commit 중 특정 manager가 예외를 던지면 아직 commit되지 않은 나머지 manager는 rollback됩니다. ([Home](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/transaction/ChainedTransactionManager.html "ChainedTransactionManager (Spring Data Core 4.0.5 API)"))
### 2. `txManager`, `txManagerMail`을 list에 등록하면 같은 Transaction인가?

#### 정확한 답

**Spring AOP 관점에서는 하나의 트랜잭션 경계로 묶을 수 있습니다.**  
**하지만 DB/리소스 관점에서는 각각 별도의 로컬 트랜잭션입니다.**

| 질문                                      | 답               |
| --------------------------------------- | --------------- |
| `@Transactional` 하나로 둘 다 제어 가능한가?       | 가능              |
| 두 DB Connection이 동일한 물리 트랜잭션인가?         | 아님              |
| 둘 다 rollback 가능한가?                      | 일반적인 예외 발생 시 가능 |
| 둘 다 commit을 원자적으로 보장하는가?                | 보장하지 않음         |
| 한쪽 commit 성공 후 다른 쪽 commit 실패 가능성이 있는가? | 있음              |
| 금융/주문/정산처럼 강한 정합성이 필요한가?                | 권장하지 않음         |

- 예를 들어 다음처럼 등록했다고 가정
```xml
<bean id="chainedTxManager" class="org.springframework.data.transaction.ChainedTransactionManager">
    <constructor-arg>
        <list>
            <ref bean="txManager"/>
            <ref bean="txManagerMail"/>
        </list>
    </constructor-arg>
</bean>
```

이 경우 동작 순서는 개념적으로 다음과 같습니다.

```text
트랜잭션 시작:
1. txManager 시작
2. txManagerMail 시작
커밋:
3. txManagerMail commit
4. txManager commit
롤백:
5. txManagerMail rollback
6. txManager rollback
```

따라서 `new ChainedTransactionManager(txManager, txManagerMail)` 형태라면 `txManagerMail`이 먼저 commit되고, 그 다음 `txManager`가 commit됩니다.

### 3. 중요한 오해: list에 등록만 하면 자동 적용되는가?

아닙니다.  
`ChainedTransactionManager` bean을 만들어도, 실제 트랜잭션 AOP가 그 manager를 사용해야 적용됩니다.

#### XML AOP 기준 예시

```xml
<tx:advice id="txAdvice" transaction-manager="chainedTxManager">
    <tx:attributes>
        <tx:method name="insert*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="update*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="delete*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="*" propagation="SUPPORTS" read-only="true"/>
    </tx:attributes>
</tx:advice>
```

만약 기존 설정이 다음처럼 되어 있으면:

```xml
<tx:advice id="txAdvice" transaction-manager="txManager">
```

`txManagerMail`은 **같은 AOP 트랜잭션에 포함되지 않습니다.**  
즉, `txManagerMail`을 list에 등록한 `chainedTxManager`를 실제 advice 또는 `@Transactional`에서 사용해야 합니다.

#### Annotation 기준 예시

```java
@Transactional(transactionManager = "chainedTxManager", rollbackFor = Exception.class)
public void saveOrderAndMailHistory(...) {
    orderDao.insertOrder(...);      // txManager 대상
    mailDao.insertMailHistory(...); // txManagerMail 대상
}
```

### 4. 실패 시나리오별 동작

| 상황                                                | 결과                                                 |
| ------------------------------------------------- | -------------------------------------------------- |
| 비즈니스 로직 중 예외 발생                                   | 두 transaction manager 모두 rollback 시도               |
| `txManagerMail` commit 실패                         | 아직 `txManager`가 commit 전이면 `txManager` rollback 가능 |
| `txManagerMail` commit 성공 후 `txManager` commit 실패 | 메일 DB는 commit, 메인 DB는 실패 가능                        |
| 한쪽 commit 완료 후 다른 쪽 실패                            | 부분 commit 발생                                       |
| commit 결과가 섞임                                     | `HeuristicCompletionException` 발생 가능               |

- 공식 API는 첫 번째 transaction manager가 commit에 성공하고 이후 transaction manager가 commit에 실패하는 경우, 부분 commit 상태가 될 수 있으며 이때 `HeuristicCompletionException`을 던져 부분 commit을 알린다고 설명합니다. ([Home](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/transaction/ChainedTransactionManager.html "ChainedTransactionManager (Spring Data Core 4.0.5 API)"))
- `HeuristicCompletionException`은 transaction coordinator 측의 heuristic decision으로 인한 transaction failure를 나타내는 Spring transaction 예외이며, 상태값으로 `STATE_MIXED` 같은 결과를 가질 수 있습니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/HeuristicCompletionException.html "HeuristicCompletionException (Spring Framework 7.0.7 API)"))

### 5. 실무상 가장 큰 위험

#### 핵심 위험

```text
메인 DB insert 성공
메일 이력 DB insert 성공
메일 이력 DB commit 성공
메인 DB commit 실패
```

이 경우 결과는 다음처럼 될 수 있습니다.

| DB     | 결과                    |
| ------ | --------------------- |
| 메인 DB  | rollback 또는 commit 실패 |
| 메일 DB  | 이미 commit됨            |
| 시스템 상태 | 불일치                   |

- 이것이 `ChainedTransactionManager`의 가장 큰 문제입니다. Spring Data Commons 이슈에서도 이 방식은 분산 트랜잭션을 best-effort로 흉내 내는 구조이며, rollback 중 불일치나 예상하지 못한 동작을 만들 수 있다고 설명합니다. 특히 transaction synchronization 저장소가 ThreadLocal 구조라는 점과, 두 `AbstractPlatformTransactionManager` 사용 시 첫 번째 manager가 동기화를 처리해버려 두 번째 commit 실패 후 복구가 어렵다는 문제가 언급되어 있습니다. ([GitHub](https://github.com/spring-projects/spring-data-commons/issues/2232 "Deprecate ChainedTransactionManager [DATACMNS-1817] · Issue #2232 · spring-projects/spring-data-commons · GitHub"))

### 6. `txManager`, `txManagerMail` 구성 시 판단 기준

|케이스|사용 판단|
|---|---|
|메인 DB와 메일 이력 DB 둘 다 반드시 일치해야 함|비권장|
|한쪽 commit 실패 시 운영자가 보정 가능|제한적 사용 가능|
|메일 이력은 부가 데이터이고 일부 누락/중복 허용 가능|사용 가능하나 outbox 권장|
|주문/결제/정산/재고 차감|비권장|
|XA 지원 DB/드라이버/운영환경 있음|JTA/XA 검토|
|단순 메일 발송 성공 이력 저장|별도 트랜잭션 또는 outbox 권장|

### 7. 등록 순서 주의

공식 문서 기준으로 “transaction을 깨뜨릴 가능성이 가장 높은 manager를 list의 마지막에 두라”고 되어 있습니다. 이유는 commit이 역순으로 실행되기 때문입니다. ([Home](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/transaction/ChainedTransactionManager.html "ChainedTransactionManager (Spring Data Core 4.0.5 API)"))

```java
new ChainedTransactionManager(txManager, txManagerMail)
```

위 순서라면:

```text
시작: txManager → txManagerMail
커밋: txManagerMail → txManager
```

따라서 `txManagerMail`이 commit 실패 가능성이 높다면 마지막에 두는 것이 상대적으로 안전합니다. 그래야 `txManagerMail` commit 실패 시 아직 `txManager`가 commit되지 않았으므로 `txManager` rollback 가능성이 남습니다.  
반대로 메인 DB인 `txManager`가 가장 중요하고, commit 실패 가능성을 가장 먼저 확인하고 싶다면 순서를 재검토해야 합니다. 다만 어떤 순서를 사용해도 **완전한 원자성은 보장되지 않습니다.**

### 8. 메일 처리 업무에서의 권장 구조

`txManagerMail`이라는 이름상 메일 발송/메일 이력 DB로 보이므로, 실무에서는 `ChainedTransactionManager`보다 아래 구조가 더 안전합니다.

#### 권장 1: 메인 트랜잭션 + Outbox

```text
1. 메인 DB 트랜잭션 안에서 업무 데이터 저장
2. 같은 메인 DB에 mail_outbox 테이블 insert
3. commit 성공 후 별도 배치/스케줄러/비동기 worker가 메일 발송
4. 성공/실패 상태 업데이트
```

|장점|설명|
|---|---|
|정합성|메인 업무 데이터와 메일 요청 데이터가 같은 DB transaction에 묶임|
|복구|발송 실패 시 재시도 가능|
|운영|실패 메일 추적 가능|
|장애 대응|메일 서버 장애가 메인 업무 transaction을 깨지 않음|

#### 권장 2: 메인 DB commit 후 메일 이력 별도 저장

```java
@Transactional(transactionManager = "txManager", rollbackFor = Exception.class)
public void saveMainData(...) {
    mainDao.insert(...);
}
```

그 후:

```java
@Transactional(transactionManager = "txManagerMail", propagation = Propagation.REQUIRES_NEW)
public void saveMailHistory(...) {
    mailDao.insert(...);
}
```

이 구조는 메일 이력 실패가 메인 업무 rollback을 유발하지 않도록 분리할 때 적합합니다. 단, 메일 이력 누락 가능성은 운영적으로 보완해야 합니다.

### 9. JTA/XA와의 차이

강한 분산 트랜잭션이 필요하다면 `ChainedTransactionManager`가 아니라 `JtaTransactionManager`와 XA resource를 검토해야 합니다. Spring Framework 공식 Javadoc은 `JtaTransactionManager`가 여러 리소스에 걸친 distributed transaction 처리에 적합하다고 설명합니다. 반면 단일 JDBC DataSource에는 `DataSourceTransactionManager`면 충분하다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/jta/JtaTransactionManager.html "JtaTransactionManager (Spring Framework 7.0.7 API)"))

|구분|ChainedTransactionManager|JTA/XA|
|---|---|---|
|방식|여러 로컬 트랜잭션 순차 제어|전역 트랜잭션|
|2PC|없음|있음|
|부분 commit 가능성|있음|낮음, coordinator가 조율|
|구성 난이도|상대적으로 낮음|높음|
|운영 복잡도|중간|높음|
|Spring 권장성|deprecated, 비권장|강한 정합성 필요 시 검토|

### 10. 실무 적용 시 체크리스트

|항목|확인 내용|
|---|---|
|적용 여부|`tx:advice` 또는 `@Transactional`이 `chainedTxManager`를 실제로 참조하는가|
|대상 DAO|각 DAO가 올바른 `DataSource` / `SqlSessionFactory` / `JdbcTemplate`을 사용하는가|
|예외 정책|checked exception도 rollback해야 하면 `rollback-for="Exception"` 또는 `rollbackFor = Exception.class` 설정|
|순서|실패 가능성이 높은 manager를 list 마지막에 둘지 검토|
|장애 테스트|두 번째 commit 실패, DB connection 끊김, timeout, deadlock 테스트|
|보정 방안|부분 commit 발생 시 수동/자동 보정 절차 준비|
|모니터링|`HeuristicCompletionException` 발생 로그 별도 알림|
|대안 검토|Outbox, 재처리 테이블, 보상 트랜잭션, JTA/XA 검토|

### 11. 최종 판단

현재 `txManager`가 메인 DB이고 `txManagerMail`이 메일 관련 DB라면, 실무 권장 순서는 다음입니다.

|업무 성격|권장 방식|
|---|---|
|메일 이력이 반드시 메인 업무와 100% 일치해야 함|JTA/XA 또는 같은 DB로 통합|
|메일 발송/이력은 부가 기능|`ChainedTransactionManager`보다 Outbox 권장|
|단기간에 최소 변경으로 두 DB rollback을 맞추고 싶음|`ChainedTransactionManager` 제한적 사용 가능|
|운영 장애 시 수동 보정이 어렵다|사용 비권장|

### 요약

`ChainedTransactionManager`에 `txManager`, `txManagerMail`을 등록하면 두 manager가 하나의 Spring transaction boundary 안에서 같이 동작하게 만들 수는 있습니다. 그러나 두 DB가 하나의 물리적 transaction으로 합쳐지는 것은 아니며, commit 단계에서 부분 commit이 발생할 수 있습니다. 특히 주문, 결제, 정산, 재고처럼 정합성이 중요한 업무에는 부적합합니다. 메일 처리라면 `ChainedTransactionManager`보다 **메인 DB transaction + outbox 테이블 + 비동기 발송/재시도 구조**가 실무적으로 더 안전합니다.

## 현재 Spring Transaction 설정 분석

#### 1. 전체 결론

현재 설정은 `aa.app..service.*Service.*(..)`와 `aa.api..service.*Service.*(..)`에 해당하는 모든 Service 메소드에 대해 `ChainedTransactionManager`를 적용하고, 내부적으로 `txManager`, `txManagerMail` 두 개의 `DataSourceTransactionManager`를 함께 시작시키는 구조입니다.  
핵심 문제는 다음입니다.

| 구분       | 판단                                                       |
| -------- | -------------------------------------------------------- |
| 트랜잭션 범위  | 모든 Service 메소드가 `REQUIRED`                               |
| 트랜잭션 매니저 | `ChainedTransactionManager`                              |
| 대상 DB    | `dataSource`, `dataSourceMail` 2개                        |
| 원자성      | 완전한 단일 트랜잭션 아님                                           |
| 실무 위험    | 부분 commit, 불필요한 Mail DB Connection 점유, 조회 트랜잭션 과다, 장애 전파 |
| 개선 방향    | 기본 트랜잭션은 Main DB 중심으로 분리하고, Mail DB는 필요한 메소드에만 별도 적용     |

- `ChainedTransactionManager`는 여러 `PlatformTransactionManager`를 순차적으로 제어하지만, XA/2PC가 아니므로 두 DB의 commit을 완전하게 원자적으로 보장하지 않습니다. Spring Data Commons 공식 Javadoc에서도 이 클래스는 deprecated이며, 부분 commit을 애플리케이션이 허용하거나 복구할 수 있는 경우에만 사용해야 한다고 설명합니다. ([Home](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/transaction/ChainedTransactionManager.html?utm_source=chatgpt.com "ChainedTransactionManager (Spring Data Core 4.0.5 API)"))

---

### 2. 현재 설정 구조

```xml
<tx:advice id="txAdvice" transaction-manager="transactionManager">
    <tx:attributes>
        <tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
```

이 설정은 pointcut에 걸리는 모든 Service 메소드에 대해 다음 정책을 적용합니다.

| 항목                  | 현재 설정                       |
| ------------------- | --------------------------- |
| 대상 메소드              | `name="*"` 전체               |
| 전파 속성               | `REQUIRED`                  |
| rollback 기준         | `Exception` 포함              |
| transaction-manager | `transactionManager`        |
| 실제 manager          | `ChainedTransactionManager` |

- 즉, 조회/저장/수정/삭제/메일/외부 API 호출 여부와 관계없이 모두 동일한 트랜잭션 정책을 탑니다.

---

### 3. 현재 `transactionManager`의 실제 의미

```xml
<bean id="transactionManager" class="org.springframework.data.transaction.ChainedTransactionManager">
    <constructor-arg>
        <list>
            <ref bean="txManager"/>
            <ref bean="txManagerMail"/>
        </list>
    </constructor-arg>
</bean>
```

이 설정은 다음 두 트랜잭션 매니저를 하나의 manager처럼 묶습니다.

```xml
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
<bean id="txManagerMail" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSourceMail"/>
</bean>
```

`DataSourceTransactionManager`는 단일 JDBC `DataSource`에 대한 트랜잭션 매니저이며, 현재 스레드에 해당 `DataSource`의 JDBC Connection을 바인딩합니다. 따라서 현재 구조에서는 Main DB와 Mail DB 각각에 대해 별도 JDBC Connection 기반 로컬 트랜잭션이 생성됩니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/DataSourceTransactionManager.html?utm_source=chatgpt.com "Class DataSourceTransactionManager"))

#### 중요한 해석

```text
transactionManager = ChainedTransactionManager(txManager, txManagerMail)
```

이것은 다음을 의미합니다.

```text
Spring AOP 트랜잭션 경계: 하나처럼 보임
물리 DB 트랜잭션: Main DB, Mail DB 각각 별도
commit 보장: 완전한 atomic commit 아님
```

---

### 4. 실행 순서

현재 list 순서:

```text
1. txManager
2. txManagerMail
```

일반적인 동작 순서는 다음과 같습니다.

```text
트랜잭션 시작:
1. txManager 시작
2. txManagerMail 시작
커밋:
3. txManagerMail commit
4. txManager commit
롤백:
5. txManagerMail rollback
6. txManager rollback
```

## `ChainedTransactionManager`는 delegate transaction manager를 등록 순서대로 시작하고, commit/rollback은 역순으로 수행합니다. commit 중 실패가 발생하면 아직 commit되지 않은 transaction manager는 rollback됩니다. ([Home](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/transaction/ChainedTransactionManager.html?utm_source=chatgpt.com "ChainedTransactionManager (Spring Data Core 4.0.5 API)"))

### 5. 가장 큰 실무 문제

#### 5-1. 모든 Service 호출 시 Main DB + Mail DB 트랜잭션이 같이 시작될 가능성

현재 모든 Service 메소드가 `transactionManager`, 즉 `ChainedTransactionManager`를 사용합니다.  
따라서 다음과 같은 단순 Main DB 조회 Service도:

```java
public List<Product> selectProductList() {
    return productDao.selectProductList();
}
```

트랜잭션 관점에서는 다음처럼 처리될 수 있습니다.

```text
1. Main DB transaction 시작
2. Mail DB transaction 시작
3. Main DB 조회
4. Mail DB는 실제 업무에 사용하지 않아도 transaction manager 대상이 됨
5. commit 시 Main DB, Mail DB 모두 commit 처리
```

이 구조는 실무에서 좋지 않습니다.

| 문제            | 설명                                                     |
| ------------- | ------------------------------------------------------ |
| Connection 낭비 | Mail DB를 사용하지 않는 업무도 Mail DB Connection을 점유할 수 있음      |
| Pool 고갈       | 트래픽 증가 시 `dataSourceMail` pool까지 같이 소모                 |
| 장애 전파         | Mail DB 장애가 Main 업무 전체 장애로 전파                          |
| 성능 저하         | 모든 Service 메소드에 두 DB 트랜잭션 처리 비용 발생                     |
| 분석 어려움        | 실제 Mail DB를 사용하지 않았는데 Mail DB connection log가 발생할 수 있음 |

- 특히 현재처럼 `dataSourceMail`이 메일 이력/메일 발송 관련 DB라면, 일반 업무 Service에서 Mail DB 트랜잭션을 항상 시작하는 것은 과한 설계입니다.

---

#### 5-2. 두 DB가 완전한 하나의 트랜잭션으로 묶이지 않음

현재 설정을 보면 “두 transaction manager가 하나로 묶인다”고 오해하기 쉽습니다.  
하지만 실제로는 다음과 같습니다.

```text
Main DB transaction ≠ Mail DB transaction
ChainedTransactionManager = 두 로컬 트랜잭션의 순차 제어
```

예상 가능한 장애 시나리오:

```text
1. Main DB insert 성공
2. Mail DB insert 성공
3. Mail DB commit 성공
4. Main DB commit 실패
5. 결과: Mail DB만 반영되고 Main DB는 실패
```

결과:

| DB      | 상태                    |
| ------- | --------------------- |
| Main DB | rollback 또는 commit 실패 |
| Mail DB | 이미 commit 완료          |
| 전체 상태   | 데이터 불일치               |

- 이 경우 Spring은 `HeuristicCompletionException`을 발생시킬 수 있지만, 이미 commit된 DB를 자동으로 되돌릴 수는 없습니다. 공식 Javadoc도 `ChainedTransactionManager`는 부분 commit을 감당할 수 있는 경우에만 사용해야 한다고 설명합니다. ([Home](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/transaction/ChainedTransactionManager.html?utm_source=chatgpt.com "ChainedTransactionManager (Spring Data Core 4.0.5 API)"))

---

#### 5-3. Mail DB 장애가 전체 서비스 장애로 번질 수 있음

현재 구조에서는 Main 업무만 수행하는 Service라도 transaction manager가 `txManagerMail`까지 같이 제어합니다.  
예를 들어:

```java
public void updateMemberInfo(Member member) {
    memberDao.update(member); // Main DB만 사용
}
```

Mail DB가 장애 상태이면 다음 문제가 발생할 수 있습니다.

```text
Main DB는 정상
Mail DB connection 획득 실패
ChainedTransactionManager 시작 실패
전체 Service 실패
```

즉, 메일 DB가 부가 시스템인데도 Main 업무 전체를 실패시킬 수 있습니다.  
실무에서는 이 구조가 특히 위험합니다.

|상황|영향|
|---|---|
|Mail DB connection pool 고갈|Main 업무 Service 실패 가능|
|Mail DB 장애|App/API Service 전체 장애 가능|
|Mail DB timeout|Main DB transaction 지연|
|Mail DB deadlock|관련 없는 업무까지 영향 가능|

---

#### 5-4. 모든 메소드 `rollback-for="Exception"`의 영향

현재 설정:

```xml
<tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
```

이 설정은 checked exception까지 rollback 대상으로 포함합니다.  
장점은 명확합니다.

|장점|설명|
|---|---|
|rollback 범위 명확|`IOException`, `SQLException`, 사용자 정의 checked exception도 rollback|
|실무 실수 감소|checked exception 발생 시 의도치 않은 commit 방지|
|하지만 모든 메소드에 일괄 적용하면 단점도 있습니다.||
|문제|설명|
|---|---|
|부가 기능 실패로 전체 rollback|메일, 로그, 알림 실패가 본 업무 rollback 가능|
|예외 처리 정책 불명확|어떤 예외는 무시해야 하는데 모두 rollback 대상|
|외부 연계 오류 전파|API 호출 실패가 DB 저장까지 rollback|
|조회 메소드도 rollback 정책 적용|실익은 적고 비용만 증가|

---

#### 5-5. `REQUIRED` 내부 rollback-only 문제

`REQUIRED`는 기존 트랜잭션이 있으면 같은 물리 트랜잭션에 참여합니다.  
예:

```java
public void order() {
    try {
        couponService.useCoupon(); // REQUIRED
    } catch (Exception e) {
        // 예외를 잡고 계속 진행
    }
    orderDao.insertOrder();
}
```

## 내부 `couponService.useCoupon()`에서 RuntimeException 또는 rollback 대상 Exception이 발생하면 트랜잭션이 rollback-only로 표시될 수 있습니다. 외부에서 예외를 catch해도 최종 commit 시점에 `UnexpectedRollbackException`이 발생할 수 있습니다. Spring 공식 문서도 내부 트랜잭션 scope가 rollback-only를 설정하면 외부 commit 시점에 `UnexpectedRollbackException`이 발생하는 것이 정상 동작이라고 설명합니다. ([Home](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html?utm_source=chatgpt.com "Transaction Propagation :: Spring Framework"))  
현재 구조에서는 이 문제가 Main DB뿐 아니라 Mail DB까지 같이 엮일 수 있습니다.

#### 5-6. 조회 메소드까지 모두 쓰기 트랜잭션화

현재 `name="*"`이므로 다음 메소드도 모두 `REQUIRED`입니다.

```text
select*
get*
find*
count*
list*
search*
```

문제:

| 문제               | 설명                                        |
| ---------------- | ----------------------------------------- |
| Connection 점유 증가 | 단순 조회도 transaction scope 동안 Connection 유지 |
| Lock/일관성 비용      | DB/드라이버/격리수준에 따라 불필요한 비용 발생               |
| Pool 부하          | 조회 트래픽이 많으면 connection active 증가          |
| readOnly 최적화 불가  | `read-only="true"`가 적용되지 않음               |

- Spring 기준으로 `PROPAGATION_REQUIRED`는 기존 트랜잭션이 있으면 참여하고, 없으면 새 트랜잭션을 만듭니다. 따라서 단순 조회도 명시적으로 제외하지 않으면 트랜잭션 대상이 됩니다. ([Home](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html?utm_source=chatgpt.com "Transaction Propagation :: Spring Framework"))

---

#### 5-7. Pointcut 누락 가능성

현재 pointcut:

```xml
execution(* aa.app..service.*Service.*(..))
execution(* aa.api..service.*Service.*(..))
```

이 표현식은 보통 다음 형태를 대상으로 합니다.

```text
aa.app.xxx.service.OrderService.method()
aa.api.xxx.service.MemberService.method()
```

하지만 다음 구조는 누락될 가능성이 있습니다.

```text
aa.app.xxx.service.impl.OrderServiceImpl.method()
aa.api.xxx.service.impl.MemberServiceImpl.method()
```

이유는 `service.*Service`가 `service` 패키지 바로 아래의 `*Service` 타입을 의미하기 때문입니다. `service` 하위의 `impl` 패키지까지 포함하려면 보통 다음처럼 작성해야 합니다.

```xml
execution(* aa.app..service..*Service.*(..))
execution(* aa.app..service..*ServiceImpl.*(..))
```

또는 클래스 명명 규칙에 따라 다음처럼 넓게 잡습니다.

```xml
execution(* aa.app..service..*(..))
```

## 다만 너무 넓게 잡으면 불필요한 bean까지 트랜잭션 대상이 될 수 있으므로 주의해야 합니다.

#### 5-8. `*Service`만 매칭하므로 `*ServiceImpl` 누락 가능

현재 클래스명이 다음과 같다면:

```text
OrderService
MemberService
MailService
```

매칭 가능성이 높습니다.  
하지만 실무에서 흔한 구조가 다음이면:

```text
OrderService interface
OrderServiceImpl class
```

실제 Spring bean이 `OrderServiceImpl`이고 pointcut이 구현체명을 기준으로 평가되면 `*Service`에 걸리지 않을 수 있습니다.  
안전하게 하려면 프로젝트 구조에 맞춰 다음 중 하나를 명확히 선택해야 합니다.

|구조|권장 pointcut|
|---|---|
|interface가 `*Service`, 구현체가 `*ServiceImpl`|interface proxy 기준 적용 여부 확인 필요|
|구현체가 `*ServiceImpl`|`*ServiceImpl.*(..)` 포함|
|service 하위 패키지 포함|`service..` 사용|
|app/api 공통 적용|하나의 pointcut으로 통합 가능|

---

#### 5-9. XML `aop:config`가 두 번 분리되어 있음

현재:

```xml
<aop:config>
    <aop:pointcut id="requiredTx" .../>
    <aop:advisor .../>
</aop:config>
<aop:config>
    <aop:pointcut id="requiredApiTx" .../>
    <aop:advisor .../>
</aop:config>
```

동작 자체가 반드시 문제는 아니지만, 관리 측면에서는 하나로 합치는 편이 명확합니다.  
개선 예:

```xml
<aop:config>
    <aop:pointcut id="serviceTx"
        expression="execution(* aa.app..service..*Service.*(..)) or execution(* aa.api..service..*Service.*(..))"/>
    <aop:advisor advice-ref="txAdvice" pointcut-ref="serviceTx"/>
</aop:config>
```

## 단, 이 개선은 구조 정리일 뿐이고, 핵심 문제인 `ChainedTransactionManager` 일괄 적용 문제는 별도로 해결해야 합니다.

### 6. 현재 설정에서 발생 가능한 대표 오류

#### 6-1. Mail DB 장애로 Main 업무 실패

```text
증상:
- 회원 조회, 주문 저장, 상품 조회 등 Main 업무 실패
- 원인은 Mail DB connection timeout
가능 로그:
- Cannot get JDBC Connection
- Timeout waiting for idle object
- Could not open JDBC Connection for transaction
```

원인:

```text
모든 Service가 ChainedTransactionManager를 사용하므로 Mail DB도 transaction 시작 대상
```

---

#### 6-2. 부분 commit

```text
증상:
- Main DB에는 데이터 없음
- Mail DB에는 이력 데이터 존재
- 또는 반대 상태 발생
가능 로그:
- HeuristicCompletionException
- TransactionSystemException
- CommitException 계열
```

원인:

```text
ChainedTransactionManager는 2PC가 아니므로 한쪽 commit 후 다른 쪽 commit 실패 가능
```

---

#### 6-3. 조회 API에서 connection pool 고갈

```text
증상:
- 조회 트래픽 증가 시 Main/Mail DB pool active 증가
- Timeout waiting for idle object 발생
```

원인:

```text
select*, get*, find*도 REQUIRED 대상
Mail DB를 사용하지 않아도 txManagerMail이 참여
```

---

#### 6-4. 내부 예외 catch 후 최종 rollback

```text
증상:
- Service 내부에서 예외를 catch했는데 메소드 종료 시 UnexpectedRollbackException
```

원인:

```text
내부 REQUIRED 메소드가 rollback-only 설정
외부 메소드가 commit하려 했지만 이미 rollback-only 상태
```

---

#### 6-5. self-invocation으로 트랜잭션 미적용

```java
public void outer() {
    inner(); // 같은 클래스 내부 호출
}
public void inner() {
    dao.insert();
}
```

## Spring AOP proxy 기반 트랜잭션은 일반적으로 proxy를 통해 외부에서 호출될 때 적용됩니다. 같은 클래스 내부 호출은 proxy를 거치지 않으므로 트랜잭션이 기대와 다르게 동작할 수 있습니다.

### 7. 개선 방향

#### 개선안 1. Main DB 기본 트랜잭션과 Mail DB 트랜잭션을 분리

가장 권장하는 방향입니다.

```xml
<tx:advice id="mainTxAdvice" transaction-manager="txManager">
    <tx:attributes>
        <tx:method name="select*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="get*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="find*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="count*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="list*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="search*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="insert*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="update*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="delete*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="save*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="process*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
```

Mail 전용 Service는 별도 advice로 분리합니다.

```xml
<tx:advice id="mailTxAdvice" transaction-manager="txManagerMail">
    <tx:attributes>
        <tx:method name="insert*" propagation="REQUIRES_NEW" rollback-for="Exception"/>
        <tx:method name="save*" propagation="REQUIRES_NEW" rollback-for="Exception"/>
        <tx:method name="update*" propagation="REQUIRES_NEW" rollback-for="Exception"/>
        <tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
```

pointcut도 분리합니다.

```xml
<aop:config>
    <aop:pointcut id="mainServiceTx"
        expression="execution(* aa.app..service..*Service.*(..)) or execution(* aa.api..service..*Service.*(..))"/>
    <aop:advisor advice-ref="mainTxAdvice" pointcut-ref="mainServiceTx"/>
    <aop:pointcut id="mailServiceTx"
        expression="execution(* aa..mail..service..*Service.*(..))"/>
    <aop:advisor advice-ref="mailTxAdvice" pointcut-ref="mailServiceTx"/>
</aop:config>
```

## 단, 위 pointcut의 `aa..mail..service..*Service.*(..)`는 예시입니다. 실제 패키지가 `aa.app.mail.service`, `aa.api.mail.service`, `aa.mail.service` 중 무엇인지에 맞춰 조정해야 합니다.

#### 개선안 2. `ChainedTransactionManager` 기본 적용 제거

현재 가장 먼저 해야 할 개선입니다.

```xml
<tx:advice id="txAdvice" transaction-manager="transactionManager">
```

이 부분을 기본 업무 Service에서는 다음처럼 변경하는 것이 안전합니다.

```xml
<tx:advice id="txAdvice" transaction-manager="txManager">
```

그리고 `txManagerMail`은 Mail Service에만 별도로 적용합니다.  
이렇게 하면:

|개선 효과|설명|
|---|---|
|Main 업무 안정성|Mail DB 장애가 Main 업무 전체로 전파되는 위험 감소|
|Connection 절약|Main 업무에서 Mail DB Connection 미사용|
|장애 분석 쉬움|어떤 Service가 어떤 DB transaction을 쓰는지 명확|
|부분 commit 위험 감소|불필요한 다중 DB transaction 제거|

---

#### 개선안 3. 메일 발송/메일 이력은 Outbox 구조 검토

메일 관련 업무라면 실무적으로 가장 안전한 구조는 다음입니다.

```text
1. Main DB transaction 안에서 본 업무 저장
2. 같은 Main DB에 mail_outbox 테이블 insert
3. transaction commit
4. 별도 scheduler/worker가 mail_outbox를 읽어 메일 발송
5. 성공/실패/재시도 횟수 업데이트
```

장점:

| 구분    | 효과                                           |
| ----- | -------------------------------------------- |
| 정합성   | 본 업무와 메일 요청 기록이 같은 Main DB transaction에 포함   |
| 장애 격리 | 메일 서버/Mail DB 장애가 본 업무 transaction을 직접 깨지 않음 |
| 재처리   | 실패 메일 재시도 가능                                 |
| 운영    | 발송 상태 추적 가능                                  |
| 복구    | 누락/중복 대응이 쉬움                                 |

- 현재 `txManagerMail`이 별도 Mail DB 저장용이라면, Main 업무와 강하게 묶기보다 Outbox/비동기 구조가 더 안전합니다.

---

#### 개선안 4. 조회 메소드 분리

현재는 모든 메소드가 `REQUIRED`입니다.  
개선:

```xml
<tx:method name="select*" propagation="SUPPORTS" read-only="true"/>
<tx:method name="get*" propagation="SUPPORTS" read-only="true"/>
<tx:method name="find*" propagation="SUPPORTS" read-only="true"/>
<tx:method name="count*" propagation="SUPPORTS" read-only="true"/>
<tx:method name="list*" propagation="SUPPORTS" read-only="true"/>
<tx:method name="search*" propagation="SUPPORTS" read-only="true"/>
```

단, 조회 중에도 반드시 반복 읽기 일관성이나 lazy loading 처리가 필요한 경우는 `REQUIRED read-only=true`를 선택할 수 있습니다.

```xml
<tx:method name="select*" propagation="REQUIRED" read-only="true"/>
```

실무 기준으로는 다음처럼 판단합니다.

|조회 유형|권장|
|---|---|
|단순 MyBatis/JDBC 조회|`SUPPORTS + readOnly`|
|여러 조회를 하나의 일관된 snapshot으로 봐야 함|`REQUIRED + readOnly` 검토|
|JPA lazy loading 필요|`REQUIRED + readOnly` 검토|
|대량 조회/긴 처리|트랜잭션 최소화|

---

#### 개선안 5. pointcut 보정

현재:

```xml
execution(* aa.app..service.*Service.*(..))
```

개선 후보:

```xml
execution(* aa.app..service..*Service.*(..))
```

`service..`로 변경하면 `service` 하위 패키지까지 포함됩니다.  
구현체가 `*ServiceImpl`이면 다음도 포함해야 합니다.

```xml
execution(* aa.app..service..*ServiceImpl.*(..))
```

권장 예:

```xml
<aop:pointcut id="mainServiceTx"
    expression="
        execution(* aa.app..service..*Service.*(..)) or
        execution(* aa.app..service..*ServiceImpl.*(..)) or
        execution(* aa.api..service..*Service.*(..)) or
        execution(* aa.api..service..*ServiceImpl.*(..))
    "/>
```

## 다만 XML 속성 안에서는 줄바꿈/공백 문제를 피하기 위해 한 줄로 관리하는 편이 안전합니다.

### 8. 권장 최종 구조 예시

#### 8-1. Main 업무용

```xml
<tx:advice id="mainTxAdvice" transaction-manager="txManager">
    <tx:attributes>
        <tx:method name="select*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="get*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="find*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="count*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="list*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="search*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="insert*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="update*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="delete*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="save*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="process*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
```

#### 8-2. Mail DB 전용

```xml
<tx:advice id="mailTxAdvice" transaction-manager="txManagerMail">
    <tx:attributes>
        <tx:method name="select*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="get*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="find*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="insert*" propagation="REQUIRES_NEW" rollback-for="Exception"/>
        <tx:method name="update*" propagation="REQUIRES_NEW" rollback-for="Exception"/>
        <tx:method name="save*" propagation="REQUIRES_NEW" rollback-for="Exception"/>
        <tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
```

#### 8-3. AOP 설정 통합

```xml
<aop:config>
    <aop:pointcut id="mainServiceTx"
        expression="execution(* aa.app..service..*Service.*(..)) or execution(* aa.app..service..*ServiceImpl.*(..)) or execution(* aa.api..service..*Service.*(..)) or execution(* aa.api..service..*ServiceImpl.*(..))"/>
    <aop:advisor advice-ref="mainTxAdvice" pointcut-ref="mainServiceTx"/>
    <aop:pointcut id="mailServiceTx"
        expression="execution(* aa..mail..service..*Service.*(..)) or execution(* aa..mail..service..*ServiceImpl.*(..))"/>
    <aop:advisor advice-ref="mailTxAdvice" pointcut-ref="mailServiceTx"/>
</aop:config>
```

## 주의: `mailServiceTx` pointcut은 실제 패키지 구조 확인 후 조정해야 합니다. 또한 Mail Service가 Main Service pointcut에도 동시에 걸리면 advice가 중복 적용될 수 있으므로 패키지 분리 또는 exclusion이 필요합니다.

### 9. 최소 변경 개선안

현재 운영 중이고 큰 구조 변경이 어렵다면, 최소 변경은 다음 순서가 안전합니다.

#### 1단계: 기본 transaction manager를 Main DB로 변경

```xml
<tx:advice id="txAdvice" transaction-manager="txManager">
```

기존:

```xml
<tx:advice id="txAdvice" transaction-manager="transactionManager">
```

이 변경만으로도 모든 Service가 불필요하게 Mail DB 트랜잭션을 시작하는 문제를 크게 줄일 수 있습니다.

#### 2단계: Mail Service 전용 advice 추가

```xml
<tx:advice id="mailTxAdvice" transaction-manager="txManagerMail">
    <tx:attributes>
        <tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
```

#### 3단계: Mail Service pointcut만 별도 지정

```xml
<aop:pointcut id="mailServiceTx"
    expression="execution(* aa..mail..service..*Service.*(..)) or execution(* aa..mail..service..*ServiceImpl.*(..))"/>
<aop:advisor advice-ref="mailTxAdvice" pointcut-ref="mailServiceTx"/>
```

#### 4단계: 조회 메소드 분리

```xml
<tx:method name="select*" propagation="SUPPORTS" read-only="true"/>
<tx:method name="get*" propagation="SUPPORTS" read-only="true"/>
<tx:method name="find*" propagation="SUPPORTS" read-only="true"/>
<tx:method name="count*" propagation="SUPPORTS" read-only="true"/>
```

---

### 10. 운영 적용 전 검증 항목

| 항목                    | 확인 방법                                     |
| --------------------- | ----------------------------------------- |
| pointcut 매칭           | Service 메소드 호출 시 transaction 로그 확인        |
| Main DB만 사용하는 Service | Mail DB connection이 발생하지 않는지 확인           |
| Mail Service          | `dataSourceMail` transaction이 정상 적용되는지 확인 |
| 조회 Service            | 불필요한 transaction/connection 점유 감소 확인      |
| 예외 rollback           | checked exception 발생 시 rollback 여부 확인     |
| 내부 catch              | `UnexpectedRollbackException` 발생 케이스 테스트  |
| 부분 commit             | Chained 사용 제거 또는 제한 여부 확인                 |
| pool 상태               | DBCP2 active/idle connection 변화 확인        |

- 테스트 시에는 다음 로그 레벨을 임시로 올려 확인하는 것이 좋습니다.
```properties
logging.level.org.springframework.transaction=DEBUG
logging.level.org.springframework.jdbc.datasource.DataSourceTransactionManager=DEBUG
```

## log4j/log4j2 XML 환경이면 동일 패키지를 DEBUG로 조정하면 됩니다.

### 11. 최종 권장 판단

현재 설정은 “모든 Service 메소드를 두 DB 트랜잭션에 강제로 묶는 구조”에 가깝습니다. 이 구조는 단순해 보이지만 실무에서는 위험합니다.  
가장 중요한 개선 방향은 다음입니다.

```text
1. ChainedTransactionManager를 기본 transactionManager로 사용하지 않는다.
2. Main 업무 Service는 txManager만 사용한다.
3. Mail DB는 Mail Service에만 txManagerMail을 별도 적용한다.
4. 메일 발송/메일 이력은 가능하면 Outbox + 비동기 재처리 구조로 분리한다.
5. 조회 메소드는 SUPPORTS/readOnly로 분리한다.
6. pointcut은 service 하위 패키지와 *ServiceImpl 매칭 여부를 반드시 확인한다.
```

현재 구조에서 `ChainedTransactionManager`를 계속 사용해야 한다면, 최소한 전체 Service에 일괄 적용하지 말고 “정말 두 DB를 동시에 갱신해야 하고 부분 commit을 운영적으로 복구할 수 있는 일부 Service”에만 제한적으로 적용하는 것이 안전합니다.

## Deadlock, wait connection 발생

### 현재 Transaction 설정으로 Deadlock 발생 가능성

#### 1. 결론

현재 설정만으로 **Deadlock이 반드시 발생한다**고 단정할 수는 없습니다. Deadlock은 보통 DB에서 서로 다른 트랜잭션이 **락을 잡는 순서가 꼬일 때** 발생합니다. 다만 현재 설정은 Deadlock 또는 Deadlock처럼 보이는 `lock wait timeout`, `connection pool timeout` 가능성을 **높이는 구조**입니다.  
핵심 원인은 다음입니다.

```text
모든 Service 메소드 = REQUIRED
모든 Service 트랜잭션 = ChainedTransactionManager
결과 = Main DB + Mail DB 트랜잭션이 항상 같이 시작될 수 있음
```

## Spring의 `DataSourceTransactionManager`는 지정된 `DataSource`에서 가져온 JDBC Connection을 현재 스레드에 바인딩합니다. 따라서 현재 구조에서는 하나의 Service 호출에서 `dataSource`, `dataSourceMail` 각각의 Connection이 스레드에 묶일 수 있습니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/DataSourceTransactionManager.html?utm_source=chatgpt.com "Class DataSourceTransactionManager"))  
`ChainedTransactionManager`는 여러 transaction manager를 순차적으로 조율하지만, XA/2PC가 아니고 deprecated 상태입니다. 부분 commit이나 복구 불가능한 상태를 애플리케이션이 감당할 수 있을 때만 제한적으로 사용해야 합니다. ([Home](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/transaction/ChainedTransactionManager.html?utm_source=chatgpt.com "ChainedTransactionManager (Spring Data Core 4.0.5 API)"))

### 2. 현재 설정에서 Deadlock 가능성을 높이는 지점

#### 2-1. 모든 Service가 두 DB 트랜잭션을 동시에 시작

현재 설정:

```xml
<tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
```

그리고 transaction manager:

```xml
<bean id="transactionManager" class="org.springframework.data.transaction.ChainedTransactionManager">
    <constructor-arg>
        <list>
            <ref bean="txManager"/>
            <ref bean="txManagerMail"/>
        </list>
    </constructor-arg>
</bean>
```

이 구조에서는 Main DB만 사용하는 Service도 `txManagerMail` 트랜잭션까지 같이 시작될 수 있습니다.  
예:

```java
public void updateMember() {
    memberDao.updateMember(); // Main DB만 사용
}
```

실제 트랜잭션 관점:

```text
1. Main DB Connection 획득
2. Mail DB Connection 획득
3. Main DB update
4. Mail DB commit 처리
5. Main DB commit 처리
```

문제는 Mail DB를 쓰지 않는 업무까지 Mail DB Connection과 트랜잭션 lifecycle에 관여할 수 있다는 점입니다.

|영향|설명|
|---|---|
|Lock 유지 시간 증가|Service 메소드 전체가 트랜잭션 범위가 됨|
|Connection 점유 증가|Main DB + Mail DB Connection을 동시에 점유|
|장애 전파|Mail DB 지연이 Main 업무 지연으로 전파|
|Lock wait 증가|불필요하게 긴 트랜잭션으로 다른 트랜잭션 대기 가능|
|Deadlock 가능성 증가|여러 Service가 서로 다른 순서로 DB row를 갱신하면 충돌 가능|

---

### 3. 발생 가능한 Deadlock 시나리오

#### 3-1. Main DB 내부 Deadlock

가장 일반적인 Deadlock입니다.

```text
Thread A:
1. 주문 테이블 update
2. 회원 테이블 update
Thread B:
3. 회원 테이블 update
4. 주문 테이블 update
```

결과:

```text
Thread A는 회원 row lock 대기
Thread B는 주문 row lock 대기
서로 상대방 lock 해제를 기다림
Deadlock 발생
```

## 이 경우 원인은 `ChainedTransactionManager` 자체라기보다 **SQL 실행 순서 불일치**입니다. 다만 모든 Service가 `REQUIRED`로 길게 묶이면 lock 보유 시간이 길어져 deadlock 확률이 올라갑니다.

#### 3-2. Main DB와 Mail DB를 함께 쓰는 교차 Deadlock 또는 Lock Wait

현재 구조에서 특히 주의할 부분입니다.

```text
Thread A:
1. Main DB 주문 update
2. Mail DB 발송이력 update
Thread B:
3. Mail DB 발송이력 update
4. Main DB 주문 update
```

발생 가능 흐름:

```text
Thread A: Main DB row lock 보유
Thread B: Mail DB row lock 보유
Thread A: Mail DB row lock 대기
Thread B: Main DB row lock 대기
```

## 이 경우 두 DB가 같은 MariaDB 인스턴스 안에 있으면 InnoDB가 deadlock으로 감지할 수 있습니다. 반대로 Main DB와 Mail DB가 서로 다른 DB 서버라면 각 DB는 전체 대기 관계를 알 수 없으므로 deadlock으로 감지되지 않고 `lock wait timeout` 형태로 나타날 수 있습니다.  
MySQL/InnoDB 공식 문서 기준으로 InnoDB deadlock은 전체 트랜잭션을 rollback하며, lock wait timeout은 기본적으로 대기하던 statement만 rollback합니다. 따라서 lock wait timeout은 애플리케이션에서 전체 트랜잭션 rollback 정책을 명확히 가져가야 합니다. ([MySQL Developer Zone](https://dev.mysql.com/doc/en/innodb-error-handling.html?utm_source=chatgpt.com "17.20.5 InnoDB Error Handling"))

#### 3-3. 조회 메소드까지 REQUIRED로 묶여 Lock 유지 시간이 증가

현재는 `select*`, `get*`, `find*`, `count*`도 모두 `REQUIRED`입니다.

```java
public List<Order> selectOrderList() {
    List<Order> list = orderDao.selectOrderList();
    externalApi.call(); // 느린 외부 호출
    return list;
}
```

문제:

```text
DB 조회는 끝났지만 Service 메소드가 끝날 때까지 트랜잭션 유지
Connection 반환 지연
일부 DB 격리수준/쿼리 유형에서는 lock 또는 read view 유지
```

단순 `SELECT`는 보통 row lock을 직접 잡지 않지만, 다음 경우에는 문제가 커질 수 있습니다.

|경우|위험|
|---|---|
|`SELECT ... FOR UPDATE`|row lock 보유|
|`UPDATE 전 선행 SELECT`|이후 update까지 트랜잭션 유지|
|대량 조회 후 후처리|Connection 장시간 점유|
|외부 API 호출 포함|DB 작업과 무관하게 트랜잭션 장기화|
|`REPEATABLE READ`|read view 장기 유지로 purge 지연 가능|

---

#### 3-4. Connection Pool 고갈이 Deadlock처럼 보일 수 있음

현재 설정은 하나의 Service 호출이 Main DB와 Mail DB Connection을 모두 잡을 수 있습니다.  
예를 들어 요청 50개가 동시에 들어오면:

```text
Main DB Connection 50개 필요
Mail DB Connection 50개 필요
```

Mail DB를 실제로 사용하지 않는 요청도 Mail DB Connection을 잡을 수 있으면, 다음 오류가 발생할 수 있습니다.

```text
Cannot get JDBC Connection
Timeout waiting for idle object
Could not open JDBC Connection for transaction
```

## 이것은 DB deadlock은 아니지만, 운영 현장에서는 “서버가 멈춘 것처럼 보이는 현상”으로 나타날 수 있습니다.

### 4. 현재 설정에서 Deadlock 위험도를 높이는 원인 정리

| 원인                             |   위험도 | 설명                                                       |
| ------------------------------ | ----: | -------------------------------------------------------- |
| 모든 메소드 `REQUIRED`              |    높음 | 조회/외부연계까지 트랜잭션 범위에 포함                                    |
| 전역 `ChainedTransactionManager` |    높음 | 모든 Service가 Main DB + Mail DB를 같이 점유                     |
| 두 DB 갱신 순서 불일치                 | 매우 높음 | Main→Mail, Mail→Main 순서가 섞이면 교차 대기 가능                    |
| 긴 Service 메소드                  |    높음 | lock/connection 보유 시간 증가                                 |
| 외부 API/메일 발송 포함                |    높음 | DB lock을 잡은 상태로 외부 지연에 영향                                |
| 인덱스 부족                         |    높음 | update/delete 시 불필요한 범위 lock 증가                          |
| 대량 update/delete               |    높음 | lock 범위 확대                                               |
| 내부 예외 catch                    |    중간 | deadlock보다는 rollback-only/UnexpectedRollbackException 유발 |

- Spring의 `REQUIRED`는 기존 트랜잭션이 있으면 참여하고, 내부 트랜잭션이 rollback-only를 표시하면 외부 commit 시점에 `UnexpectedRollbackException`이 발생할 수 있습니다. 이 문제는 deadlock은 아니지만, 현재처럼 모든 Service가 같은 트랜잭션 경계에 묶이면 장애 원인 분석을 어렵게 만듭니다. ([Home](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html?utm_source=chatgpt.com "Transaction Propagation :: Spring Framework"))

---

### 5. 회피 방안

#### 5-1. 가장 우선: `ChainedTransactionManager`를 전역 기본값에서 제거

현재 구조:

```xml
<tx:advice id="txAdvice" transaction-manager="transactionManager">
```

개선:

```xml
<tx:advice id="mainTxAdvice" transaction-manager="txManager">
```

즉, 기본 업무 Service는 Main DB transaction manager만 사용하게 해야 합니다.

```xml
<tx:advice id="mainTxAdvice" transaction-manager="txManager">
    <tx:attributes>
        <tx:method name="select*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="get*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="find*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="count*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="list*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="search*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="insert*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="update*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="delete*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="save*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="process*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
```

효과:

```text
Main 업무에서 Mail DB Connection 점유 제거
Mail DB 장애가 Main 업무로 전파되는 위험 감소
불필요한 두 DB 동시 트랜잭션 제거
Deadlock/lock wait 발생 범위 축소
```

---

#### 5-2. Mail DB는 Mail Service에만 별도 적용

```xml
<tx:advice id="mailTxAdvice" transaction-manager="txManagerMail">
    <tx:attributes>
        <tx:method name="select*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="get*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="find*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="insert*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="update*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="save*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
```

AOP 예:

```xml
<aop:config>
    <aop:pointcut id="mainServiceTx"
        expression="execution(* aa.app..service..*Service.*(..)) or execution(* aa.api..service..*Service.*(..))"/>
    <aop:advisor advice-ref="mainTxAdvice" pointcut-ref="mainServiceTx"/>
    <aop:pointcut id="mailServiceTx"
        expression="execution(* aa..mail..service..*Service.*(..))"/>
    <aop:advisor advice-ref="mailTxAdvice" pointcut-ref="mailServiceTx"/>
</aop:config>
```

주의:

```text
Mail Service가 mainServiceTx와 mailServiceTx에 동시에 걸리지 않도록 패키지/pointcut을 분리해야 함
```

---

#### 5-3. 두 DB를 반드시 함께 갱신해야 한다면 갱신 순서 고정

두 DB를 동시에 갱신하는 업무를 완전히 제거할 수 없다면, 모든 코드에서 순서를 고정해야 합니다.  
권장:

```text
항상 Main DB → Mail DB 순서
또는 항상 Mail DB → Main DB 순서
```

금지:

```text
Service A: Main DB → Mail DB
Service B: Mail DB → Main DB
```

실무 기준으로는 보통 다음 순서가 낫습니다.

```text
1. Main DB 업무 데이터 처리
2. Main DB outbox 저장
3. commit
4. 별도 worker가 Mail DB 또는 메일 서버 처리
```

## 즉, 가능하면 두 DB를 한 트랜잭션처럼 묶지 않는 방향이 더 안전합니다.

#### 5-4. 외부 API, 메일 발송, 파일 처리는 트랜잭션 밖으로 분리

위험한 구조:

```java
@Transactional
public void processOrder() {
    orderDao.updateOrder();
    mailSender.send();       // 외부 지연 가능
    externalApi.call();      // 외부 지연 가능
    mailHistoryDao.insert();
}
```

개선 구조:

```text
1. Main DB transaction 시작
2. 주문 상태 변경
3. mail_outbox 저장
4. Main DB commit
5. 별도 scheduler/worker에서 메일 발송
6. 발송 결과 저장
```

## 이 방식은 DB lock 보유 시간을 줄이고, 외부 시스템 장애가 DB 트랜잭션을 붙잡는 문제를 줄입니다.

#### 5-5. 조회 메소드는 `SUPPORTS + read-only`로 분리

현재:

```xml
<tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
```

개선:

```xml
<tx:method name="select*" propagation="SUPPORTS" read-only="true"/>
<tx:method name="get*" propagation="SUPPORTS" read-only="true"/>
<tx:method name="find*" propagation="SUPPORTS" read-only="true"/>
<tx:method name="count*" propagation="SUPPORTS" read-only="true"/>
<tx:method name="list*" propagation="SUPPORTS" read-only="true"/>
<tx:method name="search*" propagation="SUPPORTS" read-only="true"/>
```

단, 다음 조회는 예외적으로 `REQUIRED + read-only`를 검토할 수 있습니다.

|조회 유형|권장|
|---|---|
|단순 목록 조회|`SUPPORTS + readOnly`|
|여러 조회를 같은 snapshot으로 봐야 함|`REQUIRED + readOnly`|
|`SELECT FOR UPDATE` 사용|명시적 변경성 업무로 분류|
|JPA lazy loading 필요|`REQUIRED + readOnly` 검토|
|MyBatis/JDBC 단순 조회|대체로 `SUPPORTS + readOnly`|

---

#### 5-6. SQL update 순서 표준화

Deadlock 회피의 핵심은 Spring 설정보다 SQL 실행 순서입니다.  
예를 들어 주문 처리에서 항상 다음 순서를 지키게 합니다.

```text
1. 회원
2. 주문
3. 주문상세
4. 재고
5. 쿠폰
6. 포인트
7. 이력
```

다른 Service도 이 순서를 어기지 않아야 합니다.  
금지 예:

```text
A Service: 주문 → 재고
B Service: 재고 → 주문
```

권장 예:

```text
A Service: 주문 → 재고
B Service: 주문 → 재고
```

---

#### 5-7. 인덱스와 where 조건 점검

Deadlock과 lock wait는 인덱스 부족 때문에도 자주 발생합니다.  
주의 SQL:

```sql
UPDATE order_item
SET status = 'CANCEL'
WHERE order_no = ?
```

`order_no` 인덱스가 없으면 불필요하게 많은 row를 스캔하고 lock 범위가 커질 수 있습니다.  
점검 대상:

```text
UPDATE/DELETE 대상 where 컬럼
JOIN UPDATE 조건 컬럼
상태값 batch update 조건 컬럼
중복 확인 후 insert하는 unique key
외래키 컬럼 인덱스
```

---

#### 5-8. Deadlock은 재시도 정책 필요

InnoDB deadlock은 정상적인 동시성 상황에서도 발생할 수 있으므로, 중요한 짧은 트랜잭션은 제한적 retry를 둘 수 있습니다. MySQL 공식 문서도 deadlock이 발생하면 전체 트랜잭션이 rollback되며 transaction을 retry해야 한다고 설명합니다. ([MySQL Developer Zone](https://dev.mysql.com/doc/en/innodb-error-handling.html?utm_source=chatgpt.com "17.20.5 InnoDB Error Handling"))  
예시 정책:

```text
대상: 주문 상태 변경, 재고 차감, 포인트 적립 등 짧은 트랜잭션
횟수: 2~3회
간격: 100~500ms backoff
주의: 외부 API 호출, 메일 발송 포함 트랜잭션에는 retry 금지
```

---

### 6. 운영에서 확인할 로그/쿼리

#### 6-1. DB에서 Deadlock 확인

MariaDB/MySQL:

```sql
SHOW ENGINE INNODB STATUS\G
```

확인 항목:

```text
LATEST DETECTED DEADLOCK
대기 중인 transaction
lock 잡은 index
대상 table
대상 query
```

#### 6-2. Lock 대기 확인

```sql
SHOW FULL PROCESSLIST;
```

상태 예:

```text
Waiting for row lock
Locked
Updating
Sending data
```

#### 6-3. Spring transaction 로그

```properties
logging.level.org.springframework.transaction=DEBUG
logging.level.org.springframework.jdbc.datasource.DataSourceTransactionManager=DEBUG
```

확인할 내용:

```text
어떤 Service에서 transaction 시작되는지
Main DB만 쓰는 Service에서 Mail DB transaction도 시작되는지
commit/rollback 순서
예외 발생 후 rollback-only 여부
```

#### 6-4. DBCP2 Pool 상태

확인 항목:

```text
numActive
numIdle
maxTotal
maxWaitMillis
borrow wait time
```

## Main DB 업무인데 Mail DB pool active가 같이 증가하면 현재 설정 문제가 실제로 영향을 주고 있는 것입니다.

### 7. 개선 우선순위

|순서|개선|효과|
|--:|---|---|
|1|전역 `ChainedTransactionManager` 제거|가장 큼|
|2|Main/Mail transaction advice 분리|장애 범위 축소|
|3|조회 메소드 `SUPPORTS/readOnly` 분리|Connection/lock 시간 감소|
|4|외부 API/메일 발송 트랜잭션 밖으로 분리|장기 트랜잭션 방지|
|5|Main→Mail 등 갱신 순서 표준화|Deadlock 회피|
|6|UPDATE/DELETE 인덱스 점검|Lock 범위 축소|
|7|짧은 트랜잭션 retry 적용|운영 안정성 증가|

---

### 8. 최종 판단

현재 설정은 DB Deadlock을 직접 만드는 설정이라기보다, **Deadlock이 발생하기 쉬운 환경을 만드는 설정**에 가깝습니다. 특히 모든 Service에 `ChainedTransactionManager`를 적용해 Main DB와 Mail DB 트랜잭션을 항상 같이 시작하는 점이 가장 위험합니다.  
가장 안전한 회피 방향은 다음입니다.

```text
1. 기본 Service transaction-manager를 ChainedTransactionManager가 아닌 txManager로 변경
2. Mail DB 작업은 txManagerMail 전용 Service로 분리
3. 조회 메소드는 SUPPORTS/readOnly 적용
4. Main DB와 Mail DB를 동시에 갱신하는 흐름 제거
5. 불가피하면 모든 Service의 DB 접근 순서를 동일하게 강제
6. 외부 API/메일 발송은 DB transaction 밖으로 분리
7. Deadlock 발생 가능 업무에는 제한적 retry 적용
```

즉, 현재 설정에서 Deadlock 가능성을 낮추는 핵심은 **트랜잭션 범위를 줄이고, 두 DB를 전역으로 묶지 않으며, SQL 갱신 순서를 표준화하는 것**입니다.

## DBCP 에서 발생 가능한 문제

### 결론

DBCP에서 문제 발생 가능성이 **높습니다.**  
특히 현재처럼 `ChainedTransactionManager(txManager, txManagerMail)`를 **전체 Service 기본 트랜잭션 매니저**로 쓰고 있다면, Main DB 업무만 처리하는 요청도 Mail DB pool의 `maxTotal=10` 제한에 막혀 전체 Service가 지연·실패할 수 있습니다.  
핵심은 다음입니다.

```text
동일 DB 서버 / 다른 스키마 여부와 무관하게
dataSource와 dataSourceMail이 별도 DBCP2 pool이면
각 pool의 maxTotal은 독립적으로 적용된다.
```

## Apache Commons DBCP2의 `maxTotal`은 해당 pool에서 동시에 할당 가능한 active connection의 최대 개수입니다. 즉 Main DB pool이 100이어도 Mail DB pool이 10이면, `dataSourceMail`을 사용하는 트랜잭션은 최대 10개까지만 동시에 connection을 빌릴 수 있습니다. ([commons.apache.org](https://commons.apache.org/dbcp/configuration.html?utm_source=chatgpt.com "BasicDataSource Configuration – Apache Commons DBCP"))

### 1. 현재 구성의 의미

현재 구조를 단순화하면 다음과 같습니다.

```xml
ChainedTransactionManager
 ├─ txManager     → dataSource      → Main DB schema, maxTotal=100
 └─ txManagerMail → dataSourceMail  → Mail DB schema, maxTotal=10
```

`DataSourceTransactionManager`는 지정된 `DataSource`에서 JDBC Connection을 가져와 현재 스레드에 바인딩합니다. 따라서 `txManager`와 `txManagerMail`은 각각 다른 pool에서 connection을 얻습니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/DataSourceTransactionManager.html?utm_source=chatgpt.com "Class DataSourceTransactionManager"))  
현재 Service 트랜잭션이 `ChainedTransactionManager`를 사용하면, 하나의 Service 호출에서 다음 흐름이 발생할 수 있습니다.

```text
1. Main DB connection 획득
2. Mail DB connection 획득
3. Service 로직 수행
4. Mail DB commit
5. Main DB commit
```

## 즉, **Main DB만 사용하는 업무라도 Mail DB connection 획득 단계에서 막힐 수 있습니다.**

### 2. 장애 발생 가능성

#### 2-1. Mail DB pool 10개가 전체 서비스 병목이 됨

동시 요청이 50개이고, 모든 요청이 `ChainedTransactionManager`를 탄다고 가정하면 다음과 같습니다.

| 구분                                                       |  설정 | 필요한 수 | 결과     |
| -------------------------------------------------------- | --: | ----: | ------ |
| Main DB pool                                             | 100 |    50 | 여유 있음  |
| Mail DB pool                                             |  10 |    50 | 40개 대기 |
| 이 경우 Main DB는 여유가 있어도 Mail DB pool 10개 때문에 전체 요청이 지연됩니다. |     |       |        |

대표 증상:
```text
Cannot get JDBC Connection
Could not open JDBC Connection for transaction
Timeout waiting for idle object
NoSuchElementException: Timeout waiting for idle object
```

## DB deadlock이 아니라 **connection pool wait**인데, 현장에서는 WAS가 멈춘 것처럼 보일 수 있습니다.

#### 2-2. Main DB connection까지 같이 묶여 낭비됨

문제는 Mail DB connection 대기만이 아닙니다.  
`ChainedTransactionManager`의 트랜잭션 시작 순서가 다음과 같다면:

```text
txManager 시작 → txManagerMail 시작
```

일부 요청은 Main DB connection을 먼저 잡은 뒤 Mail DB connection을 기다릴 수 있습니다.

```text
Thread-1 ~ Thread-10  : Main DB connection + Mail DB connection 보유
Thread-11 ~ Thread-50 : Main DB connection 보유 후 Mail DB connection 대기 가능
```

이렇게 되면 Mail DB pool 부족이 Main DB pool까지 잠식합니다.

|문제|설명|
|---|---|
|Main DB connection 점유|실제 업무 실행 전부터 Main DB connection을 잡고 대기|
|Main DB pool 낭비|Mail DB 대기 때문에 Main DB connection 반환 지연|
|장애 전파|Mail DB pool 부족이 Main DB 업무 장애로 확대|
|처리량 감소|Main DB maxTotal=100의 장점이 사라짐|

---

#### 2-3. 동일 DB 서버라서 DB 전체 connection 한도에도 영향

Main DB와 Mail DB가 같은 DB 서버에 있으면, 물리적으로는 같은 MariaDB/MySQL 서버의 `max_connections`를 공유합니다.  
예를 들어:

```text
Main pool maxTotal=100
Mail pool maxTotal=10
WAS 2대
```

최대 DB connection 가능 수는 대략:

```text
(100 + 10) × 2 = 220
```

여기에 운영자 접속, 배치, 다른 애플리케이션, 모니터링 connection이 추가됩니다.  
따라서 DB 서버의 `max_connections`가 250이라면 여유가 작습니다.

```text
DB max_connections 250
애플리케이션 최대 220
기타 접속 20~30
→ 피크 시 Too many connections 가능
```

## 동일 서버의 다른 스키마라고 해도 DB connection 수는 서버 단위로 소비된다고 보는 것이 안전합니다.

#### 2-4. Deadlock보다 Lock Wait/Pool Timeout 가능성이 더 큼

이 구성에서 더 현실적인 장애는 DB deadlock보다는 다음입니다.

| 유형                    |   가능성 | 설명                               |
| --------------------- | ----: | -------------------------------- |
| Mail DB pool timeout  |    높음 | `maxTotal=10`으로 전체 Service가 대기   |
| Main DB connection 낭비 |    높음 | Mail DB 대기 중 Main connection 보유  |
| DB max_connections 부족 | 중간~높음 | 같은 DB 서버 connection 총량 공유        |
| DB row-level deadlock |    중간 | SQL 갱신 순서 문제와 결합 시 발생            |
| 부분 commit             |    중간 | ChainedTransactionManager 특성상 가능 |

- `ChainedTransactionManager`는 여러 transaction manager를 순차 제어하지만 XA/2PC가 아니며, Spring Data Commons에서 deprecated 상태입니다. 공식 Javadoc도 부분 commit을 애플리케이션이 감당할 수 있을 때만 사용해야 한다고 설명합니다. ([Home](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/transaction/ChainedTransactionManager.html?utm_source=chatgpt.com "ChainedTransactionManager (Spring Data Core 4.0.5 API)"))

---

### 3. 가장 위험한 시나리오

#### 시나리오 A: Main 업무 폭주로 Mail pool 고갈

```text
1. 상품 조회/회원 조회/주문 저장 요청 증가
2. 모든 Service가 ChainedTransactionManager 진입
3. 실제 Mail DB를 쓰지 않아도 txManagerMail connection 필요
4. Mail pool maxTotal=10 도달
5. 11번째 요청부터 대기
6. 대기 중 Main DB connection도 잡고 있으면 Main pool까지 소모
7. 전체 서비스 지연/장애
```

#### 시나리오 B: Mail DB 지연으로 Main 업무 장애

```text
1. Mail DB schema에 lock wait 또는 slow query 발생
2. txManagerMail transaction 시작/commit 지연
3. Main 업무 Service도 같이 지연
4. Main DB는 정상인데 전체 업무 장애처럼 보임
```

#### 시나리오 C: 다중 WAS에서 DB connection 총량 초과

```text
WAS 3대
Main maxTotal=100
Mail maxTotal=10
최대 애플리케이션 connection = 330
DB max_connections = 300
→ 피크 시 DB 접속 실패 가능
```

---

### 4. 문제 회피 1순위: 전역 `ChainedTransactionManager` 제거

현재 가장 중요한 개선은 이것입니다.

#### 현재 위험 구조

```xml
<tx:advice id="txAdvice" transaction-manager="transactionManager">
```

`transactionManager`가 `ChainedTransactionManager`이면 모든 Service가 Main/Mail DB pool을 같이 사용합니다.

#### 개선 구조

```xml
<tx:advice id="mainTxAdvice" transaction-manager="txManager">
    <tx:attributes>
        <tx:method name="select*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="get*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="find*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="count*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="list*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="search*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="insert*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="update*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="delete*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="save*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="process*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
```

Mail DB는 별도 Service에만 적용합니다.

```xml
<tx:advice id="mailTxAdvice" transaction-manager="txManagerMail">
    <tx:attributes>
        <tx:method name="select*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="get*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="find*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="insert*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="update*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="save*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
```

효과:

```text
Main 업무는 Main DB pool만 사용
Mail 업무는 Mail DB pool만 사용
Mail DB pool 10개가 전체 서비스 병목이 되는 문제 제거
```

---

### 5. 문제 회피 2순위: Mail 처리 구조 분리

메일 발송/메일 이력은 Main 업무 트랜잭션과 강하게 묶지 않는 것이 안전합니다.

#### 권장 구조: Outbox

```text
1. Main DB transaction 시작
2. 본 업무 데이터 저장
3. Main DB에 mail_outbox insert
4. Main DB commit
5. 별도 scheduler/worker가 mail_outbox 조회
6. 메일 발송 또는 Mail DB 이력 저장
7. 성공/실패/재시도 횟수 업데이트
```

장점:

|구분|효과|
|---|---|
|장애 격리|Mail DB 장애가 Main 업무 실패로 전파되지 않음|
|재처리|메일 실패 재시도 가능|
|Connection 절약|일반 업무 요청에서 Mail DB connection 미사용|
|정합성|본 업무와 메일 요청 기록은 같은 Main DB transaction에 포함|
|운영성|실패 건 조회/재처리 가능|

---

### 6. 문제 회피 3순위: 부득이하게 Chained를 유지할 경우

구조 변경이 어렵다면 최소한 아래 조치를 권장합니다.

#### 6-1. Chained 적용 범위 축소

전체 Service가 아니라 정말 두 DB를 동시에 갱신해야 하는 일부 Service에만 적용합니다.

```xml
<tx:advice id="chainedTxAdvice" transaction-manager="transactionManager">
    <tx:attributes>
        <tx:method name="saveWithMail*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="processWithMail*" propagation="REQUIRED" rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
```

#### 6-2. Mail pool을 무작정 100으로 올리지 않기

Mail `maxTotal=10` 때문에 장애가 난다고 해서 단순히 100으로 올리면 DB 서버 전체 connection 고갈 위험이 커집니다.  
검토 순서:

```text
1. 실제 Mail DB 동시 처리 필요량 측정
2. WAS 대수 반영
3. DB max_connections 확인
4. Main/Mail 합산 connection 상한 계산
5. 여유 connection 확보
6. 그 후 maxTotal 조정
```

계산 예:

```text
DB max_connections = 300
WAS = 2대
Main maxTotal = 100
Mail maxTotal = 10
애플리케이션 최대 = 220
운영/배치/모니터링 예비 = 40
총 예상 = 260
여유 = 40
```

이 경우 Mail을 50으로 올리면:

```text
(100 + 50) × 2 = 300
예비 connection 없음
→ 위험
```

#### 6-3. 트랜잭션 시작 순서 재검토

현재 순서가:

```text
txManager → txManagerMail
```

이면 Mail pool이 부족할 때 Main DB connection을 먼저 잡고 Mail connection을 기다릴 수 있습니다.  
단순 병목 관점에서는 실패 가능성이 높은 Mail을 먼저 잡도록 순서를 바꾸면 Main DB connection 낭비를 줄일 수 있습니다.

```xml
<list>
    <ref bean="txManagerMail"/>
    <ref bean="txManager"/>
</list>
```

그러나 주의해야 합니다.

```text
ChainedTransactionManager는 commit이 역순으로 수행된다.
순서 변경은 commit 순서와 부분 commit 위험도도 바꾼다.
```

## 따라서 순서 변경은 임시 완화책일 뿐이고, 근본 해결은 전역 Chained 제거입니다.

### 7. DBCP2 설정 측면 개선안

#### 7-1. Main/Mail pool을 업무 특성에 맞게 분리

Mail DB가 부가 기능이면 `maxTotal=10` 자체는 틀린 값이 아닐 수 있습니다. 문제는 **모든 Service가 Mail pool을 타는 구조**입니다.

|항목|권장 판단|
|---|---|
|Mail DB가 부가 기능|작은 pool 유지 가능|
|모든 Service가 Mail pool 사용|구조 문제|
|Mail 발송/이력 처리량 많음|별도 worker pool과 Mail pool 조정|
|Main 업무와 강한 정합성 필요|같은 DB outbox 또는 JTA/XA 검토|

#### 7-2. `maxWaitMillis`를 무한 또는 과도하게 길게 두지 않기

`maxWaitMillis`가 너무 길면 장애가 빠르게 드러나지 않고 WAS thread가 오래 묶입니다.  
권장:

```text
대기 시간을 제한하고, 실패를 빠르게 감지
업무 특성에 맞게 3~10초 등으로 조정 검토
```

단, 정확한 값은 현재 응답시간 SLA와 DB 부하에 맞춰 정해야 합니다.

#### 7-3. pool 모니터링 필수

운영에서 반드시 봐야 할 값:

```text
dataSource.numActive
dataSource.numIdle
dataSourceMail.numActive
dataSourceMail.numIdle
borrow wait time
maxWaitMillis timeout count
DB Threads_connected
DB max_connections
```

## Main 업무 트래픽 증가 시 `dataSourceMail.numActive`가 같이 증가하면 현재 구조 문제가 실제로 발생 중인 것입니다.

### 8. 점검 SQL/명령

#### DB 서버 connection 현황

```sql
SHOW VARIABLES LIKE 'max_connections';
SHOW STATUS LIKE 'Threads_connected';
SHOW FULL PROCESSLIST;
```

#### 스키마별 접속 구분

같은 DB 서버에 붙어도 `db` 컬럼으로 현재 기본 스키마를 일부 확인할 수 있습니다.

```sql
SHOW FULL PROCESSLIST;
```

다만 JDBC URL에 스키마가 지정되어 있어도 쿼리에서 `other_schema.table`처럼 접근하면 processlist의 `db`만으로 완전한 업무 구분은 어렵습니다.

#### InnoDB lock 확인

```sql
SHOW ENGINE INNODB STATUS\G
```

확인:

```text
LATEST DETECTED DEADLOCK
TRANSACTIONS
lock wait
```

---

### 9. 실무 권장 최종안

#### 최우선 권장

```text
1. 기본 Service transaction-manager를 ChainedTransactionManager에서 txManager로 변경
2. Mail DB 관련 Service만 txManagerMail 사용
3. Main 업무 중 메일 필요 시 Main DB outbox에 기록
4. 별도 scheduler/worker가 Mail DB 처리
5. Mail pool maxTotal=10은 worker 동시성에 맞춰 별도 산정
```

#### 최소 변경안

```text
1. 전역 Chained 적용 제거
2. mainTxAdvice, mailTxAdvice 분리
3. 조회 메소드는 SUPPORTS/readOnly 적용
4. Mail 관련 Service pointcut이 Main pointcut과 중복되지 않게 조정
5. DBCP2 JMX/log로 Main/Mail pool active 추적
```

#### 불가피하게 현 구조 유지 시

```text
1. Mail maxTotal=10으로 전체 요청이 막히는지 모니터링
2. Chained 적용 pointcut을 일부 Service로 축소
3. DB max_connections 기준으로 Main+Mail+WAS대수 합산 재산정
4. maxWaitMillis를 제한해 장애를 빠르게 감지
5. Mail pool 증설은 최후 수단으로만 적용
```

---

### 최종 판단

Main DB와 Mail DB가 같은 서버의 다른 스키마라 하더라도, `dataSource`와 `dataSourceMail`이 별도 DBCP2 pool이면 `Main maxTotal=100`, `Mail maxTotal=10`의 차이는 실제 병목이 될 수 있습니다. 특히 현재처럼 모든 Service가 `ChainedTransactionManager`를 타면 Mail DB pool 10개가 전체 애플리케이션의 동시 처리량을 제한할 수 있습니다.  
따라서 문제의 핵심은 `Mail maxTotal=10` 자체가 아니라, **Mail DB가 필요 없는 Service까지 Mail pool을 사용하게 만드는 전역 ChainedTransactionManager 적용 구조**입니다.  
가장 안전한 회피안은 **기본 트랜잭션을 `txManager`로 되돌리고, Mail DB는 Mail 전용 Service/Outbox/비동기 worker로 분리**하는 것입니다.

## 참고 1

### 현재 설정 기준 TransactionManager 매핑 정리

#### 1. Bean ID 기준 매핑

|구분|Bean ID|Class|대상 DataSource|역할|
|---|---|---|---|---|
|Main DB용 TM|`txManager`|`org.springframework.jdbc.datasource.DataSourceTransactionManager`|`dataSource`|Main DB 단일 트랜잭션 관리|
|Mail DB용 TM|`txManagerMail`|`org.springframework.jdbc.datasource.DataSourceTransactionManager`|`dataSourceMail`|Mail DB 단일 트랜잭션 관리|
|통합 TM|`transactionManager`|`org.springframework.data.transaction.ChainedTransactionManager`|`txManager` + `txManagerMail`|Main DB + Mail DB 트랜잭션을 순차 제어|
|AOP Advice|`txAdvice`|`tx:advice`|`transactionManager` 사용|Service 메소드에 ChainedTransactionManager 적용|

#### 2. 설정상 직접 연결 관계

```xml
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
```

```text
txManager → dataSource → Main DB
```

```xml
<bean id="txManagerMail" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSourceMail"/>
</bean>
```

```text
txManagerMail → dataSourceMail → Mail DB
```

```xml
<bean id="transactionManager" class="org.springframework.data.transaction.ChainedTransactionManager">
    <constructor-arg>
        <list>
            <ref bean="txManager"/>
            <ref bean="txManagerMail"/>
        </list>
    </constructor-arg>
</bean>
```

```text
transactionManager
 ├─ txManager     → dataSource
 └─ txManagerMail → dataSourceMail
```

#### 3. 실제 Service에 적용되는 TransactionManager

첨부 설정에서는 `txAdvice`가 다음처럼 되어 있습니다.

```xml
<tx:advice id="txAdvice" transaction-manager="transactionManager">
```

따라서 아래 pointcut에 걸리는 Service 메소드들은 **`txManager` 단독** 또는 **`txManagerMail` 단독**이 아니라, `transactionManager`를 사용합니다.

```xml
execution(* aa.app..service.*Service.*(..))
execution(* aa.api..service.*Service.*(..))
```

즉, 현재 Service 적용 구조는 다음입니다.

```text
aa.app..service.*Service.*(..)
aa.api..service.*Service.*(..)
        ↓
txAdvice
        ↓
transactionManager = ChainedTransactionManager
        ↓
txManager + txManagerMail
        ↓
dataSource + dataSourceMail
```

#### 4. 중요한 해석

|질문|답|
|---|---|
|`dataSource`에 적용되는 직접 TM은?|`txManager`|
|`dataSourceMail`에 적용되는 직접 TM은?|`txManagerMail`|
|Service AOP에서 실제 사용하는 TM은?|`transactionManager`|
|`transactionManager`의 실체는?|`ChainedTransactionManager`|
|현재 Service가 Main DB만 써도 Mail DB TM이 관여할 수 있는가?|가능|
|`txManager`가 Service에 직접 적용되는가?|현재 첨부 설정만 보면 아님|
|`txManagerMail`이 Service에 직접 적용되는가?|현재 첨부 설정만 보면 아님|

#### 5. 실행 흐름

현재 list 순서가 다음과 같습니다.

```xml
<list>
    <ref bean="txManager"/>
    <ref bean="txManagerMail"/>
</list>
```

개념상 흐름은 다음과 같습니다.

```text
트랜잭션 시작:
1. txManager 시작        → dataSource Connection 처리
2. txManagerMail 시작    → dataSourceMail Connection 처리
커밋:
3. txManagerMail commit
4. txManager commit
롤백:
5. txManagerMail rollback
6. txManager rollback
```

#### 6. 최종 정리

현재 설정에서 `dataSource` 자체는 `txManager`가 관리하고, `dataSourceMail` 자체는 `txManagerMail`이 관리합니다.  
하지만 Service AOP 설정의 `transaction-manager`가 `transactionManager`로 되어 있기 때문에, 실제 Service 메소드에는 `txManager` 단독이 아니라 **`ChainedTransactionManager(transactionManager)`가 적용**됩니다.  
따라서 현재 구조의 핵심은 다음입니다.

```text
dataSource      → txManager
dataSourceMail  → txManagerMail
Service AOP     → transactionManager
transactionManager → txManager + txManagerMail
```

즉, **개별 DataSource 기준으로는 각각 별도 TransactionManager가 있지만, Service 트랜잭션 적용 기준으로는 두 TransactionManager를 묶은 ChainedTransactionManager가 사용되고 있는 설정**입니다.

## 별도의 TransactionManager 생성 고려 시

- ChainedTransactionManager는 그대로 유지, 별도로 Main DB용 TransactionManager 를 생성
- Mail DB와 관계없는 Transaction은 신규 TransactionManager를 이용하게 할 경우
### 결론

네. **문제는 상당히 감소합니다.**  
현재의 `ChainedTransactionManager`를 완전히 제거하지 않더라도, **Mail DB와 관계없는 Service는 Main DB 전용 `txManager`를 사용하도록 분리**하면 다음 문제는 크게 줄어듭니다.

| 구분              |  개선 여부 | 설명                                                 |
| --------------- | -----: | -------------------------------------------------- |
| Mail DB pool 병목 |  크게 감소 | Main 업무가 `dataSourceMail` connection을 더 이상 요구하지 않음 |
| Main 업무 장애 전파   |     감소 | Mail DB 장애가 Main DB 업무 전체로 번질 가능성 감소               |
| 불필요한 2개 DB 트랜잭션 |     감소 | Main-only 업무는 Main DB transaction만 사용              |
| 부분 commit 위험    |  일부 감소 | Chained 적용 범위가 줄어드는 만큼 감소                          |
| Chained 자체 위험   | 그대로 존재 | Chained를 쓰는 업무에는 여전히 부분 commit 가능                  |
|                 |        |                                                    |
- Spring의 `DataSourceTransactionManager`는 단일 `DataSource`용 transaction manager이며, 해당 `DataSource`의 JDBC Connection을 현재 스레드에 바인딩합니다. 따라서 Main DB 전용 transaction manager를 사용하면 Main DB connection만 트랜잭션 대상이 됩니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/DataSourceTransactionManager.html?utm_source=chatgpt.com "Class DataSourceTransactionManager"))

---

### 1. 현재 구조의 문제

현재 구조는 다음과 같습니다.

```xml
<tx:advice id="txAdvice" transaction-manager="transactionManager">
```

여기서 `transactionManager`가 다음 bean입니다.

```xml
<bean id="transactionManager" class="org.springframework.data.transaction.ChainedTransactionManager">
    <constructor-arg>
        <list>
            <ref bean="txManager"/>
            <ref bean="txManagerMail"/>
        </list>
    </constructor-arg>
</bean>
```

즉, 모든 Service가 아래 흐름을 탈 수 있습니다.

```text
Service 호출
→ txManager 시작
→ txManagerMail 시작
→ Main DB Connection 사용
→ Mail DB Connection 사용 또는 점유 가능
→ Mail DB commit
→ Main DB commit
```

## 이 구조에서는 Main DB만 사용하는 업무도 Mail DB pool 상태에 영향을 받을 수 있습니다.

### 2. 개선 구조

질문하신 개선 방향은 다음 구조입니다.

```text
ChainedTransactionManager 유지
 ├─ Main + Mail 동시 처리 필요 업무에만 사용
Main DB 전용 TransactionManager 추가/사용
 └─ Mail DB와 무관한 일반 업무에 사용
```

사실 현재 설정에도 이미 Main DB 전용 manager가 있습니다.

```xml
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
```

## 따라서 “새로 생성”보다는 **기존 `txManager`를 Main 업무 기본 transaction manager로 사용**하는 것이 더 적절합니다.

### 3. 적용 예시

#### 3-1. Chained는 이름을 명확히 변경

기존 `transactionManager`라는 이름은 기본 transaction manager처럼 보이므로 혼동을 줄이는 것이 좋습니다.

```xml
<bean id="chainedTxManager" class="org.springframework.data.transaction.ChainedTransactionManager">
    <constructor-arg>
        <list>
            <ref bean="txManager"/>
            <ref bean="txManagerMail"/>
        </list>
    </constructor-arg>
</bean>
```

#### 3-2. Main DB 전용 advice

```xml
<tx:advice id="mainTxAdvice" transaction-manager="txManager">
    <tx:attributes>
        <tx:method name="select*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="get*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="find*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="count*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="list*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="search*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="insert*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="update*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="delete*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="save*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="process*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
```

#### 3-3. Chained 전용 advice

```xml
<tx:advice id="chainedTxAdvice" transaction-manager="chainedTxManager">
    <tx:attributes>
        <tx:method name="saveWithMail*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="processWithMail*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
```

## 다만 `name="*"`를 Chained 전용 advice에 넣으면 pointcut이 넓을 때 다시 모든 메소드가 Chained를 탈 수 있으므로, Chained advice는 가능하면 메소드명을 제한하는 것이 안전합니다.

### 4. AOP pointcut 분리 예시

#### 권장 구조

```xml
<aop:config>
    <aop:pointcut id="mainServiceTx"
        expression="execution(* aa.app..service..*Service.*(..)) or execution(* aa.api..service..*Service.*(..))"/>
    <aop:advisor advice-ref="mainTxAdvice" pointcut-ref="mainServiceTx"/>
    <aop:pointcut id="chainedServiceTx"
        expression="execution(* aa.app..service..*MailLinkedService.*(..)) or execution(* aa.api..service..*MailLinkedService.*(..))"/>
    <aop:advisor advice-ref="chainedTxAdvice" pointcut-ref="chainedServiceTx"/>
</aop:config>
```

## 하지만 위 구조에는 중요한 문제가 있습니다. `*MailLinkedService`가 `mainServiceTx`에도 같이 걸릴 수 있습니다. 그러면 **동일 메소드에 Main advice와 Chained advice가 중복 적용**될 수 있습니다.

### 5. 가장 중요한 주의점: Pointcut 중복 금지

아래처럼 잡으면 위험합니다.

```xml
<aop:pointcut id="mainServiceTx"
    expression="execution(* aa.app..service..*Service.*(..))"/>
<aop:pointcut id="chainedServiceTx"
    expression="execution(* aa.app..service..*MailLinkedService.*(..))"/>
```

`MailLinkedService`도 `*Service`에 포함되면 두 advice가 모두 적용될 수 있습니다.

#### 안전한 분리 방법

패키지를 나누는 방식이 가장 명확합니다.

```text
aa.app.main.service      → txManager
aa.app.mailtx.service    → chainedTxManager
aa.api.main.service      → txManager
aa.api.mailtx.service    → chainedTxManager
```

예시:

```xml
<aop:config>
    <aop:pointcut id="mainServiceTx"
        expression="execution(* aa.app.main..service..*Service.*(..)) or execution(* aa.api.main..service..*Service.*(..))"/>
    <aop:advisor advice-ref="mainTxAdvice" pointcut-ref="mainServiceTx"/>
    <aop:pointcut id="chainedServiceTx"
        expression="execution(* aa.app.mailtx..service..*Service.*(..)) or execution(* aa.api.mailtx..service..*Service.*(..))"/>
    <aop:advisor advice-ref="chainedTxAdvice" pointcut-ref="chainedServiceTx"/>
</aop:config>
```

패키지 분리가 어렵다면 클래스명 규칙으로 분리하되, main pointcut에서 chained 대상을 제외하는 방식이 필요합니다.

```xml
<aop:pointcut id="mainServiceTx"
    expression="(execution(* aa.app..service..*Service.*(..)) or execution(* aa.api..service..*Service.*(..))) and !execution(* aa..service..*MailLinkedService.*(..))"/>
```

---

### 6. 개선 후 효과

#### 기존

```text
일반 Service 호출
→ Main DB pool 사용
→ Mail DB pool 사용 가능
→ Mail maxTotal=10 병목 가능
```

#### 개선 후

```text
일반 Service 호출
→ Main DB pool만 사용
→ Mail DB pool 미사용
```

|항목|기존|개선 후|
|---|---|---|
|Main-only 업무|Chained 사용|txManager만 사용|
|Mail DB pool 사용|거의 모든 Service에서 가능|Mail 연계 업무에서만 사용|
|Mail DB 장애 영향|전체 업무로 전파 가능|Chained 대상 업무로 제한|
|Connection 사용량|요청당 최대 2개 pool 사용|일반 요청은 1개 pool만 사용|
|장애 분석|어려움|어느 manager를 쓰는지 명확|

---

### 7. 남는 문제

이 방식은 문제를 줄이지만, `ChainedTransactionManager`의 구조적 한계는 남습니다.  
`ChainedTransactionManager`는 여러 transaction manager를 순차적으로 제어하지만, 부분 commit으로 인한 불일치 상태를 애플리케이션이 감당하거나 복구할 수 있을 때만 사용해야 하며, 그 외에는 권장되지 않습니다. 현재 공식 Javadoc에서도 deprecated로 표시되어 있습니다. ([Home](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/transaction/ChainedTransactionManager.html?utm_source=chatgpt.com "ChainedTransactionManager (Spring Data Core 4.0.5 API)"))  
즉, 아래 위험은 Chained 대상 업무에 계속 남습니다.

```text
Main DB commit 성공
Mail DB commit 실패
또는
Mail DB commit 성공
Main DB commit 실패
→ 부분 commit 가능
```

---

### 8. 호출 관계에서 주의할 점

#### 8-1. Main transaction 안에서 Chained Service를 호출하는 경우

```text
MainService(txManager)
 → MailLinkedService(chainedTxManager)
```

이 경우 Main DB transaction이 이미 있는 상태에서 Chained가 다시 `txManager`와 `txManagerMail`을 제어하려고 하므로 구조가 복잡해집니다.  
Spring의 `PROPAGATION_REQUIRED`는 기존 트랜잭션이 있으면 참여하고, 없으면 새 트랜잭션을 생성합니다. 동일한 물리 transaction에 매핑될 수 있으며, 내부 scope가 rollback-only를 설정하면 외부 commit에도 영향을 줄 수 있습니다. ([Home](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html?utm_source=chatgpt.com "Transaction Propagation :: Spring Framework"))  
권장하지 않는 호출:

```text
Main 전용 Service → Chained Service 직접 호출
```

권장 구조:

```text
상위 Facade Service에서 처음부터 필요한 transaction manager를 명확히 선택
```

예:

```text
Main only 업무    → MainFacadeService(txManager)
Main + Mail 업무 → MailLinkedFacadeService(chainedTxManager)
```

---

#### 8-2. Chained transaction 안에서 Main Service를 호출하는 경우

```text
MailLinkedFacadeService(chainedTxManager)
 → MainService(txManager)
```

이 경우 Main DB 쪽은 이미 thread-bound connection이 있을 가능성이 높습니다. MainService가 `REQUIRED`이면 기존 Main DB transaction에 참여하는 형태가 됩니다. 다만 Mail DB transaction도 이미 열려 있으므로 전체 업무는 여전히 Chained 범위 안에 있습니다.  
즉, 이 경우는 다음처럼 봐야 합니다.

```text
MainService가 txManager를 사용하더라도
상위가 Chained이면 이미 Mail DB transaction 범위 안이다.
```

---

#### 8-3. Main Service에서 Mail Service를 별도 txManagerMail로 호출하는 경우

```text
MainService(txManager)
 → MailService(txManagerMail)
```

이 경우 Chained가 아니므로 두 트랜잭션은 서로 독립입니다. MailService가 먼저 commit되고, 이후 MainService가 실패하면 다음 상황이 생길 수 있습니다.

```text
Mail DB 이력 commit 완료
Main DB rollback
→ 업무 데이터는 없는데 메일 이력만 남음
```

## 따라서 Mail 처리는 가능하면 직접 호출하지 말고 Outbox 또는 after-commit 이벤트 구조가 더 안전합니다.

### 9. 실무 권장안

#### 가장 안전한 구조

```text
1. 기본 Service는 txManager 사용
2. ChainedTransactionManager는 일부 Facade Service에만 제한
3. Mail DB와 무관한 Service는 절대 Chained pointcut에 포함하지 않음
4. Chained 대상 Service는 별도 패키지 또는 명명 규칙으로 분리
5. Main Service → Chained Service 호출 구조는 피함
6. Mail 발송/이력은 가능하면 Outbox/비동기 처리로 분리
```

#### 권장 패키지 예시

```text
aa.app.order.service          → txManager
aa.app.member.service         → txManager
aa.app.mailtx.service         → chainedTxManager
aa.app.mail.service           → txManagerMail 또는 비동기 worker
```

---

### 10. 운영 검증 방법

아래 로그를 임시로 활성화한 뒤, Main-only Service 호출 시 `txManagerMail` 로그가 발생하지 않는지 확인해야 합니다.

```properties
logging.level.org.springframework.transaction=DEBUG
logging.level.org.springframework.jdbc.datasource.DataSourceTransactionManager=DEBUG
```

확인 기준:

|테스트|기대 결과|
|---|---|
|Main-only 조회|`txManagerMail` transaction 시작 없음|
|Main-only 저장|`dataSource` connection만 사용|
|Mail 연계 저장|`txManager`, `txManagerMail` 모두 사용|
|Mail DB 장애|Main-only 업무는 정상 수행|
|Mail pool maxTotal 초과|Chained 대상 업무만 영향|

---

### 최종 판단

현재의 `ChainedTransactionManager`를 유지하더라도, **Mail DB와 관계없는 트랜잭션을 Main DB 전용 `txManager`로 분리하면 문제는 확실히 감소합니다.** 특히 Mail DB pool 10개로 인해 전체 Service가 막히는 문제, Mail DB 장애가 Main 업무로 전파되는 문제, 불필요한 2개 DB connection 점유 문제를 크게 줄일 수 있습니다.  
다만 이것은 **ChainedTransactionManager의 위험을 제거하는 것이 아니라 적용 범위를 줄이는 개선**입니다. 따라서 Chained는 “정말 Main DB와 Mail DB를 동시에 갱신해야 하고, 부분 commit 발생 시 복구 절차가 있는 일부 업무”에만 제한적으로 사용해야 합니다.