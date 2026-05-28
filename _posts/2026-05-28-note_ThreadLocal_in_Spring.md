---
layout: single
title: "ThreadLocal_in_Spring"
excerpt: "ThreadLocal_in_Spring"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-05-28"
last_modified_at: "2026-05-28 14:25:44 +0900"
---
## 1. Spring 5.3 버전에서의 ThreadLocal 동작 방식

Spring 5.3이 사용되던 시기는 주로 **플랫폼 스레드(Platform Thread, OS 스레드와 1:1 매핑)** 기반의 'Thread-per-Request'(요청당 스레드 하나 할당) 모델을 기본으로 사용했습니다. 이 환경에서 `ThreadLocal`은 멀티스레드 간의 데이터 간섭 없이 현재 스레드만의 고유한 컨텍스트를 유지하는 핵심 기술이었습니다.

### 1-1. 🛠️ 동작 메커니즘

- **Thread 내부의 비밀 저장소:** 자바의 모든 `Thread` 객체는 내부에 `ThreadLocalMap`이라는 커스텀 Map을 하나씩 가지고 있습니다.

- **Key와 Value의 관계:** `ThreadLocal` 변수를 선언하고 `set(value)`을 호출하면, `ThreadLocal` 인스턴스 자체가 **Key**가 되고, 저장하려는 데이터가 **Value**가 되어 현재 실행 중인 스레드의 `ThreadLocalMap`에 저장됩니다.

- **값의 격리:** 다른 스레드가 동일한 `ThreadLocal` 변수를 참조해 `get()`을 호출하더라도, 각자 자신의 `Thread` 객체 내부에 있는 `ThreadLocalMap`을 조회하므로 서로의 데이터에 절대 침범할 수 없습니다.

### 1-2. 🔄 Spring 5.3에서의 구체적인 처리 흐름

1. **요청 진입:** HTTP 요청이 들어오면 톰캣 등의 스레드 풀에서 스레드를 하나 배정합니다.

2. **컨텍스트 바인딩:** Spring의 필터나 인터셉터(`RequestContextFilter`, `SecurityContextPersistenceFilter` 등)가 해당 요청의 HTTP 정보나 로그인 유저 인증 정보를 `ThreadLocal`에 저장합니다.

3. **비즈니스 로직 수행:** `ApplicationContext`가 관리하는 싱글톤 빈(Service, Repository 등)들은 인자값으로 요청 정보를 계속 넘겨받지 않고도, 필요할 때 `ThreadLocal` 기반의 Holder 클래스들(`RequestContextHolder` 등)을 통해 안전하게 데이터를 꺼내 씁니다.

4. **⚠️ 청소 및 반환 (가장 중요):** 플랫폼 스레드는 생성 비용이 비싸 스레드 풀에서 **재사용**됩니다. 따라서 Spring은 요청 처리가 끝나고 응답을 반환하기 직전에 필터 단에서 반드시 `ThreadLocal.remove()`를 호출하여 데이터를 삭제합니다. 만약 이를 누락하면 다음 요청을 처리하는 엉뚱한 사용자가 이전 사용자의 정보를 보게 되는 치명적인 **메모리 누수 및 보안 문제**가 발생합니다.

## 2. 이후 Spring 버전에서의 변화된 점 (Spring 6 / Boot 3 이후)

Java 21의 등장과 함께 **Spring Framework 6.x 및 Spring Boot 3.x** 버전부터는 자바 진영의 스레드 모델에 거대한 패러다임 변화가 생겼고, 이에 따라 `ThreadLocal`을 다루는 방식에도 큰 변화가 일어나고 있습니다.

### 2-1. ① 가상 스레드(Virtual Threads)의 전면 도입

- Spring Boot 3.2 버전부터는 `spring.threads.virtual.enabled=true` 설정 하나만으로 내장 톰캣의 스레드 모델을 수백만 개까지 생성이 가능한 가상 스레드로 전환할 수 있습니다.

- **ThreadLocal의 한계 직면:** 가상 스레드도 `ThreadLocal`을 지원은 하지만, 수백만 개의 가상 스레드가 각자 무거운 데이터를 `ThreadLocal`에 담아 보관하게 되면 메모리 사용량이 급증하는 병목이 생길 수 있습니다. 또한, 기존 플랫폼 스레드 풀 방식처럼 장시간 살아있는 스레드가 아니라 수 밀리초 만에 생성되고 죽는 구조이기에 기존의 `ThreadLocal` 수명 관리 방식과 결이 맞지 않습니다.

### 2-2. ② Scoped Values(스코프 값)로의 전환 준비

- Java 21에서 프리뷰로 도입되고 이후 최신 JDK 버전에서 정식 스펙으로 자리 잡은 **Scoped Values**는 `ThreadLocal`의 고질적인 문제를 해결하는 대안입니다.

- **불변성(Immutability):** `ThreadLocal`은 스레드 내부 어디서든 값을 임의로 바꿀 수 있어 추적이 어려웠지만, Scoped Values는 한 번 바인딩되면 해당 스코프 내에서 값을 변경할 수 없습니다.

- **명시적인 생명주기:** 명시적인 람다 블록 구조(`ScopedValue.where(KEY, value).run(...)`)로 실행되기 때문에, 스코프를 벗어나면 **자동으로 값이 소멸**됩니다. 더 이상 개발자나 Spring 프레임워크가 `remove()`를 깜빡해서 메모리가 누수될까 걱정할 필요가 없어졌습니다.

- **현재 동향:** Spring Security 및 주요 에코시스템들은 기존의 `ThreadLocal` 기반 컨텍스트 매커니즘을 가상 스레드와 구조적 동시성(Structured Concurrency)에 최적화된 Scoped Values 기반 전략으로 전환하거나 혼용할 수 있도록 내부 전략 인터페이스를 확장하는 추세입니다.

### 2-3. ③ 비동기 컨텍스트 전파(Context Propagation)의 정교화

- Reactive Stack(WebFlux)이나 비동기 처리(`@Async`), 혹은 가상 스레드 환경에서 스레드가 중간에 전환될 때 `ThreadLocal`의 데이터가 유실되는 고질적인 문제가 있었습니다.

- Spring 6.x 버전부터는 `RequestContextHolder`나 Micrometer의 `Observation` 등과 연계하여, 서로 다른 스레드 경계를 넘나들 때도 컨텍스트 데이터가 끊기지 않고 자동으로 복사 및 전파되도록 돕는 `ThreadLocalAccessor` 메커니즘이 프레임워크 전반에 깊숙이 통합되어 안전성을 높였습니다.

■ [Spring 5.3: ThreadLocal 방식 (플랫폼 스레드 수명 주기)]
[요청 인입] ──> [스레드 풀에서 스레드 대여] ──> [ThreadLocalMap에 데이터 set()]
                                                          │ (싱글톤 빈들이 공유)
[요청 종료] ──> [⚠️ 프레임워크가 명시적으로 remove() 호출] ──> [데이터 삭제 후 스레드 반납]

■ [Spring 6.0+: Scoped Values 방식 (가상 스레드 수명 주기)]
[요청 인입] ──> [가상 스레드 즉시 생성] ──> [ScopedValue.where(KEY, value).run()]
                                                       │ (지정된 스코프 내 불변 유지)
[스코프 종료] ──> [✨ 시스템이 자동으로 데이터 소멸] ──> [가상 스레드 소멸]

| **변화 요인**      | **기존 Spring 5.3 (ThreadLocal)**      | **현대 Spring 6+ (Virtual Threads & Scoped Values)** |
| -------------- | ------------------------------------ | -------------------------------------------------- |
| **스레드 모델**     | OS 스레드 기반의 무거운 플랫폼 스레드 풀 관리          | 수백만 개를 가볍게 생성 및 소멸시키는 가상 스레드 도입                    |
| **컨텍스트 공유 방식** | `ThreadLocal` 기반의 가변(Mutable) 맵 격리   | `ScopedValue` 기반의 불변(Immutable) 범위 격리              |
| **메모리 효율성**    | 스레드가 많아질수록 무거운 Map 복사로 병목 유발 가능      | 자식 스레드 생성 시 복사 비용 없이 포인터 공유 (Zero-cost)            |
| **생명주기 및 정리**  | 개발자나 프레임워크가 **반드시 `remove()` 명시 호출** | 실행 스코프(`run()`)가 종료되면 **JVM이 자동 소멸**               |

## 3. 참고 - GPT

Spring Framework 5.3에서 `ThreadLocal`은 **현재 실행 중인 Thread에 값을 붙여두는 저장소**입니다. Spring은 이를 이용해 **현재 요청 정보**, **현재 트랜잭션 정보**, **현재 보안 인증 정보**, **로그 추적 정보** 등을 메서드 파라미터로 계속 넘기지 않고도 같은 Thread 안에서 꺼내 쓸 수 있게 합니다. Java `ThreadLocal`은 `get()`, `set()`, `remove()`로 현재 Thread 전용 값을 다루며, `remove()`는 현재 Thread에 저장된 값을 제거합니다. ([Oracle Docs](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/ThreadLocal.html?utm_source=chatgpt.com "ThreadLocal (Java SE 11 & JDK 11 )"))

```text
Thread-1
 └─ ThreadLocal[A] = "사용자1 요청 정보"
Thread-2
 └─ ThreadLocal[A] = "사용자2 요청 정보"
```

같은 `ThreadLocal` 변수를 사용하더라도 Thread가 다르면 값은 서로 분리됩니다.

### 3-1. ThreadLocal을 쉽게 이해하기

`ThreadLocal`은 전역 변수처럼 보이지만 실제로는 **Thread별 개인 보관함**에 가깝습니다.

```java
private static final ThreadLocal<String> USER_ID = new ThreadLocal<>();
```

위 변수가 `static`이어도 값 자체가 모든 Thread에 공유되는 것은 아닙니다.

```java
USER_ID.set("userA");
String userId = USER_ID.get();
USER_ID.remove();
```

동작 흐름은 다음과 같습니다.

```text
1. ThreadLocal 객체 생성
2. 현재 Thread에서 set(value) 호출
3. 현재 Thread 내부 저장소에 값 저장
4. 같은 Thread에서 get() 호출 시 값 조회
5. 작업 종료 시 remove()로 제거
```

중요한 점은 `ThreadLocal`의 값이 **객체 기준이 아니라 Thread 기준으로 분리**된다는 것입니다.

### 3-2. 일반 변수와 ThreadLocal 차이

|구분|일반 static 변수|ThreadLocal|
|---|---|---|
|값 공유|모든 Thread가 같은 값 공유|Thread마다 별도 값 보유|
|동시성 위험|높음|상대적으로 낮음|
|대표 용도|공통 설정, 상수|요청 정보, 트랜잭션 정보, 인증 정보|
|주의점|동기화 필요|반드시 정리 필요|
|Thread Pool 영향|상대적으로 단순|Thread 재사용 시 값 오염 가능|
|예를 들어 일반 `static String userId`를 쓰면 A 사용자 요청 값이 B 사용자 요청에서 보일 수 있습니다. 반면 `ThreadLocal<String>`은 요청 처리 Thread별로 값이 분리됩니다.|||

### 3-3. Spring 5.3에서 ThreadLocal을 쓰는 대표 지점

Spring 5.3의 Servlet 기반 애플리케이션에서는 아래 영역에서 ThreadLocal 개념이 중요합니다.

|구분|대표 클래스|저장 내용|
|---|---|---|
|Request|`RequestContextHolder`|현재 요청의 `RequestAttributes`|
|Transaction|`TransactionSynchronizationManager`|현재 Thread에 묶인 Connection, Session, 트랜잭션 상태|
|Security|`SecurityContextHolder`|현재 인증 사용자, 권한 정보|
|Logging|MDC 등|traceId, userId, requestId 등 로그 추적 값|
|`TransactionSynchronizationManager`는 공식 Javadoc에서 “Thread별 리소스와 트랜잭션 동기화를 관리하는 중앙 delegate”라고 설명합니다. 또한 JDBC Connection이나 Hibernate Session 같은 리소스는 현재 Thread에 바인딩되어 조회될 수 있습니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.39/javadoc-api/org/springframework/transaction/support/TransactionSynchronizationManager.html "TransactionSynchronizationManager (Spring Framework 5.3.39 API)"))|||

### 3-4. Spring MVC 요청에서 ThreadLocal 흐름

Spring MVC 요청은 보통 아래 흐름으로 처리됩니다.

```text
HTTP 요청
 → WAS Thread 할당
 → Filter / DispatcherServlet 진입
 → RequestContextHolder에 현재 요청 정보 저장
 → Controller
 → Service
 → Repository / DAO
 → 응답 완료
 → ThreadLocal 정리
 → Thread Pool로 Thread 반환
```

예를 들어 Controller, Service, Interceptor 등에서 현재 요청 정보를 직접 파라미터로 넘기지 않아도 Spring 내부에서는 Thread에 묶인 요청 정보를 찾을 수 있습니다. `RequestContextHolder`는 현재 웹 요청을 Thread-bound `RequestAttributes` 형태로 노출하는 Holder 클래스이며, `inheritable` 플래그가 설정된 경우 자식 Thread에 상속될 수 있다고 설명됩니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/context/request/RequestContextHolder.html?utm_source=chatgpt.com "RequestContextHolder (Spring Framework 7.0.6 API)"))

```java
RequestAttributes attrs = RequestContextHolder.getRequestAttributes();
```

다만 실무에서는 Service 계층에서 `RequestContextHolder`에 직접 의존하는 설계는 조심해야 합니다. Service가 웹 요청에 강하게 묶이면 배치, 스케줄러, 비동기 작업, 테스트에서 문제가 생길 수 있습니다.

### 3-5. Spring Transaction과 ThreadLocal

Spring의 선언적 트랜잭션은 ThreadLocal 이해가 매우 중요합니다.

```java
@Transactional
public void order() {
    orderDao.insertOrder();
    paymentDao.insertPayment();
}
```

위 메서드가 실행되면 대략 아래 흐름이 됩니다.

```text
1. TransactionInterceptor 진입
2. TransactionManager가 트랜잭션 시작
3. JDBC Connection 확보
4. ConnectionHolder를 현재 Thread에 바인딩
5. DAO에서 같은 DataSource 사용 시 현재 Thread의 Connection 재사용
6. 정상 종료 시 commit
7. 예외 발생 시 rollback
8. ThreadLocal에서 트랜잭션 리소스 정리
```

Spring의 `@Transactional` Javadoc은 이 어노테이션이 일반적으로 `PlatformTransactionManager`가 관리하는 **Thread-bound transaction**과 함께 동작하며, 현재 실행 Thread 안의 모든 데이터 접근 작업에 트랜잭션을 노출한다고 설명합니다. 또한 새로 시작한 Thread에는 트랜잭션이 전파되지 않는다고 명시합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.39/javadoc-api/org/springframework/transaction/annotation/Transactional.html "Transactional (Spring Framework 5.3.39 API)"))  
즉 아래 코드는 주의해야 합니다.

```java
@Transactional
public void order() {
    orderDao.insertOrder();
    new Thread(() -> {
        paymentDao.insertPayment(); // 기존 @Transactional 트랜잭션에 자동 참여하지 않음
    }).start();
}
```

`new Thread`, `@Async`, 별도 Executor에서 실행되는 코드는 기본적으로 기존 요청 Thread의 트랜잭션 ThreadLocal을 공유하지 않습니다.

### 3-6. TransactionSynchronizationManager의 실제 의미

`TransactionSynchronizationManager`는 실무에서 직접 많이 쓰지는 않지만, Spring 트랜잭션 동작 이해에는 핵심입니다.

```text
현재 Thread
 └─ TransactionSynchronizationManager
     ├─ DataSource → ConnectionHolder
     ├─ transactionName
     ├─ readOnly 여부
     ├─ isolation level
     ├─ actualTransactionActive
     └─ TransactionSynchronization 목록
```

공식 Javadoc에 따르면 `bindResource()`는 주어진 리소스를 현재 Thread에 바인딩하고, `unbindResource()`는 현재 Thread에서 제거합니다. `clear()`는 현재 Thread의 트랜잭션 동기화 상태 전체를 정리합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.39/javadoc-api/org/springframework/transaction/support/TransactionSynchronizationManager.html "TransactionSynchronizationManager (Spring Framework 5.3.39 API)"))  
이 구조 때문에 같은 `@Transactional` 범위 안에서 여러 DAO가 같은 DB Connection을 재사용할 수 있습니다.

```text
Service @Transactional
 ├─ ADao.select()
 │   └─ 현재 Thread의 Connection 사용
 └─ BDao.insert()
     └─ 같은 Thread의 같은 Connection 사용
```

### 3-7. @Async, Executor, Scheduler에서 문제가 생기는 이유

ThreadLocal은 **Thread가 바뀌면 값이 자동 전달되지 않습니다.**

```java
@Async
public void sendMail() {
    // 요청 Thread의 RequestContextHolder, TransactionSynchronizationManager,
    // SecurityContextHolder 값이 기본적으로 그대로 있다고 보면 안 됨
}
```

예시 흐름:

```text
요청 Thread: http-nio-8080-exec-1
 ├─ RequestContextHolder 있음
 ├─ TransactionSynchronizationManager 있음
 └─ SecurityContextHolder 있음

@Async Thread: task-1
 ├─ RequestContextHolder 없음
 ├─ 기존 트랜잭션 없음
 └─ SecurityContextHolder 없음 또는 별도 전파 필요
```

그래서 비동기 작업에서는 필요한 값을 명시적으로 넘기는 방식이 안전합니다.

```java
public void requestProcess() {
    String userId = loginUser.getUserId();
    asyncService.sendMail(userId);
}
```

요청 객체, 세션 객체, 트랜잭션 상태 자체를 비동기 Thread로 그대로 가져가려는 설계는 피하는 것이 좋습니다.

### 3-8. Thread Pool에서 remove가 중요한 이유

WAS Thread는 요청마다 새로 만들어지는 것이 아니라 Thread Pool에서 재사용됩니다.

```text
요청 A → Thread-10 사용 → ThreadLocal에 userA 저장
요청 A 완료 → remove 안 함
요청 B → 같은 Thread-10 재사용
요청 B에서 userA 값이 남아 있을 수 있음
```

따라서 직접 만든 ThreadLocal은 반드시 `finally`에서 제거해야 합니다.

```java
private static final ThreadLocal<String> TENANT_ID = new ThreadLocal<>();
public void execute(String tenantId) {
    try {
        TENANT_ID.set(tenantId);
        // business logic
    } finally {
        TENANT_ID.remove();
    }
}
```

Java 공식 Javadoc에서도 `remove()`는 현재 Thread의 값을 제거하고, 이후 다시 읽으면 `initialValue()`에 의해 재초기화될 수 있다고 설명합니다. ([Oracle Docs](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/ThreadLocal.html?utm_source=chatgpt.com "ThreadLocal (Java SE 11 & JDK 11 )"))

### 3-9. Spring이 대신 정리해주는 경우와 직접 정리해야 하는 경우

|상황|정리 주체|주의|
|---|---|---|
|Spring MVC 요청 정보|Spring MVC / Filter / Listener|일반 요청 흐름이면 보통 자동 정리|
|`@Transactional` 트랜잭션|TransactionManager|정상적인 Spring 트랜잭션 경계라면 자동 정리|
|Spring Security 인증 정보|Security Filter Chain|요청 처리 후 정리됨|
|개발자가 만든 `ThreadLocal`|개발자|반드시 `finally remove()` 필요|
|MDC 로그 컨텍스트|개발자 또는 필터|Thread Pool 재사용 시 정리 필요|
|Spring Security 공식 문서도 기본적으로 `SecurityContextHolder`가 `ThreadLocal`을 사용하며, 요청 처리 후 Thread를 clear하는 것이 안전하고 `FilterChainProxy`가 `SecurityContext`를 clear한다고 설명합니다. ([Home](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html?utm_source=chatgpt.com "Servlet Authentication Architecture :: Spring Security"))|||

### 3-10. InheritableThreadLocal 주의

`InheritableThreadLocal`은 부모 Thread의 값을 자식 Thread에 복사할 수 있습니다.

```java
private static final InheritableThreadLocal<String> USER = new InheritableThreadLocal<>();
```

하지만 WAS, Executor, Thread Pool 환경에서는 주의해야 합니다.

```text
일반 new Thread
 └─ 생성 시점에 부모 값 복사 가능

Thread Pool
 └─ Thread가 미리 만들어져 있거나 재사용됨
 └─ 요청 시점의 부모 값 전파가 기대대로 안 될 수 있음
```

따라서 Spring 실무에서는 `InheritableThreadLocal`로 요청/인증/테넌트 정보를 해결하려 하기보다, 필요한 값을 DTO나 메서드 인자로 명시적으로 넘기거나, Executor의 context propagation 전략을 쓰는 편이 안전합니다.

### 3-11. Spring 5.3 기준 실무 사용 예시

#### 3-11-1. TenantContext 예시

멀티테넌트 시스템에서 자주 보이는 패턴입니다.

```java
public final class TenantContext {
    private static final ThreadLocal<String> TENANT_ID = new ThreadLocal<>();
    private TenantContext() {}
    public static void setTenantId(String tenantId) {
        TENANT_ID.set(tenantId);
    }
    public static String getTenantId() {
        return TENANT_ID.get();
    }
    public static void clear() {
        TENANT_ID.remove();
    }
}
```

Filter에서 설정하고 반드시 제거합니다.

```java
public class TenantFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        try {
            HttpServletRequest httpRequest = (HttpServletRequest) request;
            String tenantId = httpRequest.getHeader("X-Tenant-Id");
            TenantContext.setTenantId(tenantId);
            chain.doFilter(request, response);
        } finally {
            TenantContext.clear();
        }
    }
}
```

#### 3-11-2. 잘못된 예

```java
public void doSomething() {
    TenantContext.setTenantId("A");
    businessLogic();
    // remove 없음
}
```

이 코드는 Thread Pool 환경에서 다음 요청에 이전 tenant 값이 남을 수 있습니다.

#### 3-11-3. 권장 예

```java
public void doSomething() {
    try {
        TenantContext.setTenantId("A");
        businessLogic();
    } finally {
        TenantContext.clear();
    }
}
```

### 3-12. Spring 5.3에서 특히 조심할 지점

|지점|문제|
|---|---|
|`@Async`|기존 요청 Thread의 ThreadLocal 값이 자동 전달되지 않음|
|`new Thread()`|트랜잭션, 요청, 인증 Context가 자동 전파되지 않음|
|Scheduler|웹 요청 Thread가 아니므로 `RequestContextHolder` 없음|
|Batch|웹 요청과 무관하므로 request/session 기반 ThreadLocal 사용 위험|
|Thread Pool|remove 누락 시 이전 작업 값이 다음 작업에 노출 가능|
|Service에서 RequestContextHolder 사용|웹 계층 의존성이 생겨 테스트/배치/비동기에서 취약|
|트랜잭션 내부 비동기 실행|기존 DB 트랜잭션에 자동 참여하지 않음|

### 3-13. Spring 5.3 이후 변화

#### 3-13-1. Imperative MVC/Transaction의 기본 구조는 크게 바뀌지 않음

Spring 6.x 이후에도 Servlet MVC 기반 요청 처리와 `PlatformTransactionManager` 기반 선언적 트랜잭션은 여전히 **Thread-bound context** 개념이 중요합니다. 즉, `@Transactional`이 같은 Thread 안에서 동작하고 새 Thread로 자동 전파되지 않는다는 기본 원칙은 유지됩니다.

#### 3-13-2. Reactive에서는 ThreadLocal 대신 Reactor Context가 중요

Spring 5.3의 `@Transactional` Javadoc에도 이미 Reactive Transaction은 ThreadLocal이 아니라 Reactor Context를 사용한다고 설명되어 있습니다. 따라서 WebFlux/R2DBC 기반에서는 ThreadLocal 중심 사고보다 Reactor Context 중심으로 봐야 합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.39/javadoc-api/org/springframework/transaction/annotation/Transactional.html "Transactional (Spring Framework 5.3.39 API)"))

```text
Spring MVC + JDBC
 └─ ThreadLocal 기반 트랜잭션 이해가 중요

Spring WebFlux + R2DBC
 └─ Reactor Context 기반 트랜잭션 이해가 중요
```

#### 3-13-3. Spring Framework 6.1에서 ContextPropagatingTaskDecorator 추가

Spring Framework 6.1에는 `ContextPropagatingTaskDecorator`가 추가되었습니다. 공식 Javadoc은 이 데코레이터가 다른 Thread에서 작업이 실행될 때 context propagation을 돕고, logging context나 observation context 복원에 유용하다고 설명합니다. 단, 작은 작업이 매우 많은 애플리케이션에서는 오버헤드 때문에 권장되지 않는다고 명시합니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/task/support/ContextPropagatingTaskDecorator.html?utm_source=chatgpt.com "Class ContextPropagatingTaskDecorator"))

```java
@Bean
public ThreadPoolTaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setTaskDecorator(new ContextPropagatingTaskDecorator());
    return executor;
}
```

이 기능은 “모든 ThreadLocal을 무조건 안전하게 복사한다”는 의미가 아닙니다. 주로 로그/관측성 context 전파에 적합하며, 트랜잭션 Connection을 다른 Thread로 옮기는 용도로 보면 안 됩니다.

#### 3-13-4. Spring Framework 6.1에서 Virtual Thread 지원 강화

Spring Framework 6.1 릴리스 노트에는 JDK 21 Virtual Thread와의 일반 호환성, `VirtualThreadTaskExecutor`, `SimpleAsyncTaskExecutor`의 virtual threads mode, `SimpleAsyncTaskScheduler` 등이 언급되어 있습니다. ([GitHub](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-6.1-Release-Notes?utm_source=chatgpt.com "Spring Framework 6.1 Release Notes"))  
Spring Framework 공식 문서도 6.1부터 `SimpleAsyncTaskScheduler`가 JDK 21 Virtual Thread와 정렬된 방식으로 동작하며, 단일 scheduler Thread를 사용하되 각 scheduled task 실행마다 새 Thread를 시작한다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/reference/integration/scheduling.html?utm_source=chatgpt.com "Task Execution and Scheduling :: Spring Framework"))

```text
Spring 5.3
 └─ Java 8+ / Java 11 LTS 지원
 └─ 전통적 Thread Pool 중심

Spring 6.1+
 └─ Java 17+ 기반
 └─ Java 21 Virtual Thread 지원 강화
 └─ Context propagation 보조 기능 강화
```

#### 3-13-5. Spring Boot 사용 시 Virtual Thread 설정도 가능

Spring Boot 문서 기준으로 Java 21 이상에서 `spring.threads.virtual.enabled=true`를 설정하면 관련 executor/scheduler builder가 Virtual Thread를 사용하도록 자동 구성될 수 있습니다. ([Home](https://docs.spring.io/spring-boot/reference/features/task-execution-and-scheduling.html?utm_source=chatgpt.com "Task Execution and Scheduling"))

```properties
spring.threads.virtual.enabled=true
```

다만 이것은 Spring Framework 단독 기능이라기보다는 Spring Boot 자동 설정 영역입니다.

### 3-14. Virtual Thread와 ThreadLocal의 관계

Virtual Thread에서도 `ThreadLocal` 자체는 사용할 수 있습니다. 다만 설계 관점이 달라집니다.

|구분|Platform Thread Pool|Virtual Thread|
|---|---|---|
|Thread 재사용|많이 재사용|작업 단위로 많이 생성 가능|
|remove 중요성|매우 중요|여전히 중요하지만 재사용 오염 위험은 상대적으로 감소|
|ThreadLocal 비용|장기 생존 Thread에 값 누적 위험|대량 Virtual Thread에서 과도한 ThreadLocal 사용 비용 주의|
|트랜잭션 전파|같은 Thread 안에서만|같은 원칙|
|비동기 전파|자동 아님|자동 아님|
|Virtual Thread가 도입되어도 “Thread가 바뀌면 ThreadLocal 값은 자동으로 따라가지 않는다”는 원칙은 그대로입니다.|||

### 3-15. 실무 판단 기준

|질문|판단|
|---|---|
|현재 요청 안에서만 필요한 값인가?|ThreadLocal 사용 가능|
|비동기/스케줄러/배치에서도 필요한 값인가?|명시적 파라미터 전달 우선|
|DB 트랜잭션 정보를 다른 Thread로 넘기려는가?|하면 안 됨|
|로그 traceId 전파가 필요한가?|MDC + TaskDecorator 검토|
|사용자 인증 정보가 필요한가?|Spring Security의 공식 Delegating 계열 또는 SecurityContext 전파 방식 검토|
|직접 만든 ThreadLocal인가?|반드시 `try-finally-remove` 적용|
|Service에서 RequestContextHolder를 쓰려는가?|가급적 Controller/Interceptor에서 필요한 값 추출 후 전달|

### 3-16. 최종 요약

```text
ThreadLocal = 현재 Thread 전용 저장소
Spring MVC 요청 정보 = RequestContextHolder에 Thread 단위 저장
Spring Transaction = TransactionSynchronizationManager에 Thread 단위 저장
Spring Security = SecurityContextHolder에 Thread 단위 저장
@Async/new Thread = 기존 ThreadLocal 자동 전파 안 됨
Thread Pool = remove 누락 시 값 오염 위험
Reactive = ThreadLocal보다 Reactor Context 중심
Spring 6.1+ = Virtual Thread와 Context Propagation 지원 강화
```

Spring Framework 5.3 기반 커머스/업무 시스템에서는 `ThreadLocal`을 “편리한 전역 저장소”로 보면 안 됩니다. 정확히는 **현재 요청을 처리하는 하나의 Thread 안에서만 유효한 실행 컨텍스트 저장소**로 봐야 합니다. 특히 트랜잭션, 사용자, 테넌트, 로그 추적값을 다룰 때는 `@Async`, Scheduler, Batch, Thread Pool 재사용, `remove()` 누락 여부를 반드시 점검해야 합니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
