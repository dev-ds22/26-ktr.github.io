---
layout: single
title: "ApplicatonContext_와_ContextHolder"
excerpt: "ApplicatonContext_와_ContextHolder"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-05-28"
last_modified_at: "2026-05-28 13:56:21 +0900"
---
Spring 프레임워크에서 `ApplicationContext`와 `ContextHolder`는 **직접적인 상속이나 포함 관계는 아니지만, 웹 애플리케이션의 아키텍처 상에서 서로를 보완하며 긴밀하게 협력하는 관계**입니다.

Spring에는 단독으로 `ContextHolder`라는 이름을 가진 클래스는 없으며, 대신 특정 목적을 위해 컨텍스트를 홀딩(저장/관리)하는 `RequestContextHolder`, `SecurityContextHolder`, `LocaleContextHolder` 등이 존재합니다. 이 클래스들은 대개 **ThreadLocal(쓰레드 로컬)** 기술을 기반으로 동작합니다.

두 개념의 차이점과 이들이 어떻게 관계를 맺고 협력하는지 자세히 설명해 드리겠습니다.

## 1. 핵심 개념 및 역할 비교

두 구조는 관리하는 데이터의 범위(Scope)와 **목적**에서 가장 큰 차이가 납니다.

- **ApplicationContext (애플리케이션 전역 관리자)**

    - **역할:** Spring의 IoC 컨테이너로서, 애플리케이션 구동 시 필요한 설정과 **싱글톤 빈(Bean)들을 생성하고 관리**합니다.

    - **범위:** 애플리케이션 전체(Global Scope)에서 공유됩니다. 애플리케이션이 켜져 있는 동안 평생 유지됩니다.

- **ContextHolder (쓰레드 단위의 데이터 보관소)**

    - **역할:** 현재 실행 중인 **쓰레드(Thread)에 종속된 동적인 데이터(HTTP 요청 정보, 인증 정보 등)를 보관**하고, 이를 어디서나 정적(static)인 방법으로 접근할 수 있게 해줍니다.

    - **범위:** 쓰레드 범위(Thread Scope) 또는 요청 범위(Request Scope)입니다. 하나의 HTTP 요청이 들어와서 응답이 나갈 때까지만 유지되고 사라집니다.
## 2. 둘 사이의 상호작용 및 관계 (어떻게 협력하는가?)

구조적으로 독립되어 있지만, 실제 런타임 환경에서는 다음과 같은 방식으로 연결되어 동작합니다.
### 2-1. ① 싱글톤 빈이 동적 데이터에 접근하는 가교 역할

`ApplicationContext`가 관리하는 빈들은 기본적으로 싱글톤(Singleton)입니다. 즉, 수많은 사용자의 요청이 들어와도 단 하나의 객체만 생성되어 공유됩니다.

이때, 싱글톤 객체(예: Service 레이어의 빈)가 "현재 이 요청을 보낸 사용자의 ID"나 "현재 요청의 HTTP 헤더 정보" 같은 **사용자마다 다른 동적인 데이터**를 알고 싶을 때 문제가 발생합니다. 싱글톤 빈 내부에 멤버 변수로 이런 데이터를 저장하면 멀티쓰레드 환경에서 데이터 동기화 오류(공유 데이터 오염)가 발생하기 때문입니다.

이 문제를 해결하기 위해 `ContextHolder`를 사용합니다.

- `ApplicationContext`에 등록된 싱글톤 빈은 메서드 내부에서 `RequestContextHolder.getRequestAttributes()`나 `SecurityContextHolder.getContext().getAuthentication()` 같은 static 메서드를 호출합니다.

- 이를 통해 싱글톤 객체의 안전성을 유지하면서도, 현재 쓰레드에 할당된 고유한 요청/인증 데이터에 안전하게 접근할 수 있습니다.
### 2-2. ② Spring의 특수 스코프 빈(Request, Session Scope) 구현의 기반

`ApplicationContext`는 빈의 생명주기를 관리할 때 기본 싱글톤 외에도 `Request`, `Session` 같은 특수한 웹 스코프를 지원합니다. 사용자의 HTTP 요청이 들어올 때마다 새로운 빈 객체를 만들고 요청이 끝나면 삭제하는 기능입니다.

`ApplicationContext`가 내부적으로 이 Request 스코프 빈을 생성하고 관리할 수 있는 이유가 바로 `RequestContextHolder` 덕분입니다.

- 웹 요청이 들어오면 Spring의 `DispatcherServlet`이나 Filter가 요청 정보를 `RequestContextHolder`에 먼저 저장합니다.

- 이후 `ApplicationContext`가 Request 스코프 빈을 주입하거나 조회해야 할 때, `RequestContextHolder`를 참조하여 현재 쓰레드에 매핑된 요청 객체를 찾아내고 그 요청에 종속된 빈을 반환하게 됩니다.

## 3. 대표적인 ContextHolder 종류와 활용 예시

### 3-1. RequestContextHolder (Spring Web)

현재 쓰레드의 HTTP 요청 정보(`HttpServletRequest`, `HttpServletResponse`)를 가지고 있습니다.

- **협력 방식:** 컨트롤러가 아닌 일반 Service나 Utility 클래스(모두 `ApplicationContext`가 관리하는 빈)에서 현재 요청의 IP 주소나 헤더 값을 꺼내 쓰고 싶을 때 사용됩니다.

```java
// ApplicationContext가 관리하는 Service 빈 내부
public void doSomething() {
    ServletRequestAttributes attr = (ServletRequestAttributes) RequestContextHolder.currentRequestAttributes();
    HttpServletRequest request = attr.getRequest();
    String userAgent = request.getHeader("User-Agent"); // 현재 쓰레드의 요청 헤더 접근
}
```
### 3-2. SecurityContextHolder (Spring Security)

현재 인증된 사용자 정보(`Authentication`, `Principal`)를 가지고 있습니다.

- **협력 방식:** Spring Security 프레임워크 역시 `ApplicationContext` 위에서 빈으로 동작합니다. 보안 필터들이 인증을 완료하면 그 결과를 `SecurityContextHolder`에 저장합니다. 이후 비즈니스 로직 빈들은 이를 가져와 권한 체크를 수행합니다.

```java
// 비즈니스 로직 빈 내부
public void processPayment() {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    String username = auth.getName(); // 현재 로그인한 사용자 이름 가져오기
}
```

## 4. 요약
- **ApplicationContext**는 정적이고 구조적인 틀(애플리케이션의 뼈대와 부품인 **Bean을 관리**)을 제공합니다.
- **ContextHolder**는 동적이고 흐르는 데이터(현재 실행 중인 **쓰레드의 상태와 컨텍스트를 관리**)를 제공합니다.
- 결과적으로 `ApplicationContext`가 관리하는 무상태(Stateless) 빈들이, 유상태(Stateful)인 사용자 요청 데이터를 안전하게 처리할 수 있도록 **ContextHolder가 쓰레드 로컬(ThreadLocal) 저장소를 통해 데이터를 제공하는 보완적 관계**라고 이해하시면 됩니다.
## 5. 참고 - ApplicationContext 와 ServletContext

Spring 프레임워크 기반의 웹 애플리케이션을 개발할 때 `ServletContext`와 `ApplicationContext`는 설정과 빈(Bean) 관리의 핵심을 이루는 두 가지 큰 축입니다. 이름이 비슷해서 처음에는 헷갈리기 쉽지만, "누가 관리하는가"와 "어떤 영역을 담당하는가"를 이해하면 명확히 구분할 수 있습니다.

이해하기 쉽게 각각의 개념부터 둘 사이의 관계까지 자세히 정리해 드릴게요.

## 6. ServletContext (서블릿 컨테이너의 영역)

`ServletContext`는 Spring의 개념이 아니라, **Java EE(Jakarta EE) 서블릿 표준 사양**에 정의된 개념입니다.

- **관리 주체:** Tomcat, Jetty 같은 서블릿 컨테이너(Web Container)가 관리합니다.

- **범위(Scope):** 웹 애플리케이션(WAR)당 **단 1개만 생성**됩니다.

- **역할:** 웹 애플리케이션 전체가 공유하는 컨텍스트입니다. 애플리케이션이 구동될 때 생성되어 종료될 때까지 유지되며, 서블릿들이 서로 정보를 공유할 수 있는 저장소 역할을 합니다. 웹 애플리케이션의 자원(파일 경로 등)을 얻어오거나, 컨텍스트 초기화 파라미터를 참조할 때 사용됩니다.

## 7. ApplicationContext (Spring의 영역)

`ApplicationContext`는 **Spring 프레임워크의 핵심 컨테이너**입니다.

- **관리 주체:** **Spring 프레임워크**가 관리합니다.

- **범위(Scope):** 애플리케이션의 필요에 따라 1개 또는 그 이상 존재할 수 있습니다.

- **역할:** Spring 빈(Bean)의 생명주기(Creation, Lifecycle, Dependency Injection)를 관리하는 IoC 컨테이너입니다. 웹 환경에서는 이를 확장한 `WebApplicationContext` 인터페이스를 주로 사용하며, 이 컨텍스트를 통해 `ServletContext`에 접근할 수도 있습니다.

## 8. 전통적인 Spring MVC에서의 계층 구조 (관계성)

전통적인 Spring MVC 웹 애플리케이션에서는 애플리케이션을 효율적으로 관리하기 위해 ApplicationContext를 부모-자식 관계의 계층 구조(Hierarchy)로 나누어 설계했습니다. 이 구조에서 `ServletContext`와의 연결 고리가 만들어집니다.

### 8-1. Root ApplicationContext (부모 컨테이너)

- **생성 주체:** `ContextLoaderListener`가 생성합니다.

- **역할:** 웹 기술(컨트롤러, 뷰 등)에 종속되지 않는 **비즈니스 로직(Service), 데이터 접근(Repository, DB 설정), 보안(Security) 관련 빈**들을 등록합니다.

- **특징:** 웹 환경 전체에서 사용되는 공통 빈들을 관리합니다.

### 8-2. Servlet ApplicationContext (자식 컨테이너)

- **생성 주체:** `DispatcherServlet`이 생성합니다. 흔히 '웹 컨텍스트'라고도 부릅니다.

- **역할:** 웹 레이어와 직접적인 관련이 있는 빈(Controller, ViewResolver, HandlerMapping 등)들을 등록합니다.

- **특징:** `Root ApplicationContext`를 부모 컨테이너로 지정합니다. 자식 컨테이너는 부모 컨테이너의 빈을 참조할 수 있지만, 반대로 부모 컨테이너는 자식 컨테이너의 빈을 참조할 수 없습니다.

> **💡 ServletContext와의 연결**
> 
> `ServletContext` 안에는 Spring의 `Root ApplicationContext`와 `Servlet ApplicationContext`가 속성(Attribute) 형태로 저장되어 관리됩니다. 즉, 서블릿 컨테이너라는 거대한 틀(`ServletContext`) 안에 Spring 컨테이너들(`ApplicationContext`)이 둥지를 틀고 있는 모양새입니다.

## 9. 한눈에 보는 비교 요약

|**구분**|**ServletContext**|**ApplicationContext (WebApplicationContext)**|
|---|---|---|
|**기반 기술**|Java EE / Jakarta EE 표준|Spring Framework|
|**관리 주체**|서블릿 컨테이너 (Tomcat 등)|Spring 컨테이너|
|**존재 목적**|웹 애플리케이션 전역 자원 및 정보 공유|Spring 빈(Bean)의 생명주기 및 의존성 관리|
|**생성 개수**|웹 애플리케이션당 **무조건 1개**|설정에 따라 1개 이상 존재 가능 (계층 구조 가능)|
|**주요 대상**|서블릿, 필터, 웹 전역 파라미터 등|Controller, Service, Repository 등 Spring 빈|

## 10. 현대 Spring Boot에서의 변화

과거 Spring MVC 시절에는 `web.xml` 파일에 `ContextLoaderListener`와 `DispatcherServlet`을 각각 설정하면서 위와 같은 복잡한 계층 구조를 직접 눈으로 확인해야 했습니다.

하지만 **Spring Boot**로 넘어오면서 구조가 매우 단순해졌습니다.

- Spring Boot는 내장 톰캣을 띄우면서 단 하나의 통합된 `WebApplicationContext`를 생성하는 것이 기본 전략입니다.

- 특별한 이유가 없다면 Root와 Servlet으로 컨텍스트를 분리하지 않고, 하나의 Spring 컨테이너 안에 컨트롤러, 서비스, 리포지토리를 모두 때려 넣고(?) 관리합니다.

- 따라서 현대의 Spring Boot 환경에서는 개발자가 `ServletContext`나 복잡한 계층형 `ApplicationContext` 설정을 직접 건드릴 일이 과거에 비해 현저히 줄어들었습니다.

## 11. 참고 2 - GPT

`ServletContext`와 `ApplicationContext`는 같은 “Context”라는 이름을 쓰지만 역할이 다릅니다. `ServletContext`는 **WAS/Servlet Container가 관리하는 웹 애플리케이션 전역 객체**이고, `ApplicationContext`는 **Spring이 관리하는 Bean Container**입니다. Spring MVC 웹 애플리케이션에서는 보통 `ServletContext` 안에 Spring의 `WebApplicationContext`가 등록되어 두 세계가 연결됩니다. Spring 공식 문서에서도 `DispatcherServlet`은 자신의 설정을 위해 `ApplicationContext`를 확장한 `WebApplicationContext`를 사용하며, 이 `WebApplicationContext`는 `ServletContext`와 연결되고 `ServletContext`에 바인딩된다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet/context-hierarchy.html "Context Hierarchy :: Spring Framework"))

## 12. 전체 구조

```text
[WAS / Servlet Container]
 └─ ServletContext
     ├─ Filter
     ├─ Listener
     ├─ Servlet
     │   └─ DispatcherServlet
     │       └─ Servlet별 WebApplicationContext
     │           ├─ Controller
     │           ├─ HandlerMapping
     │           ├─ ViewResolver
     │           └─ Web MVC 관련 Bean
     └─ Root WebApplicationContext
         ├─ Service
         ├─ Repository / DAO
         ├─ TransactionManager
         ├─ DataSource
         ├─ CacheManager
         └─ 공통 인프라 Bean
```

Spring MVC에서는 하나의 `Root WebApplicationContext`를 여러 `DispatcherServlet` 또는 다른 Servlet이 공유할 수 있고, 각 `DispatcherServlet`은 자기 전용 Child `WebApplicationContext`를 가질 수 있습니다. Root Context는 보통 Service, Repository, DataSource 같은 공통 Bean을 담고, Servlet별 Child Context는 Controller, ViewResolver, HandlerMapping 같은 웹 계층 Bean을 담는 구조가 일반적입니다. ([Home](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet/context-hierarchy.html "Context Hierarchy :: Spring Framework"))

## 13. ServletContext란?

`ServletContext`는 Spring 객체가 아니라 Servlet 표준 영역의 객체입니다. 하나의 웹 애플리케이션, 즉 하나의 WAR 또는 하나의 웹 모듈 단위로 생성되는 전역 Context라고 보면 됩니다. Spring 5.3 기반에서는 일반적으로 `javax.servlet.ServletContext`를 사용합니다.

| 구분          | 설명                                                               |
| ----------- | ---------------------------------------------------------------- |
| 관리 주체       | WAS, 예: Tomcat, JBoss, WebLogic 등                                |
| 생성 시점       | 웹 애플리케이션 배포/기동 시                                                 |
| 소멸 시점       | 웹 애플리케이션 종료/언로드 시                                                |
| 범위          | 웹 애플리케이션 전체                                                      |
| 주요 용도       | 전역 속성 저장, init-param 조회, 리소스 경로 조회, Listener/Filter/Servlet 간 공유 |
| Spring과의 관계 | Spring의 `WebApplicationContext`가 이 안에 attribute로 등록될 수 있음        |

- 예를 들어 `ServletContext`는 아래와 같은 정보를 다룰 수 있습니다.

```java
ServletContext servletContext = request.getServletContext();
String value = servletContext.getInitParameter("someParam");
servletContext.setAttribute("key", object);
Object object = servletContext.getAttribute("key");
```

다만 실무에서는 `ServletContext.setAttribute()`에 업무 객체를 마구 넣는 방식은 권장되지 않습니다. Spring Bean으로 관리해야 하는 객체는 `ApplicationContext`에 Bean으로 등록하는 것이 맞습니다.

## 14. ApplicationContext란?

`ApplicationContext`는 Spring IoC Container입니다. 즉, Spring Bean을 생성하고, 의존성을 주입하고, Bean의 생명주기, 이벤트, 메시지, 리소스 등을 관리하는 핵심 컨테이너입니다. Spring Javadoc 기준으로 `ApplicationContext`는 Bean 접근, 리소스 로딩, 이벤트 발행, 메시지 처리 등의 기능을 제공합니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/ApplicationContext.html?utm_source=chatgpt.com "ApplicationContext (Spring Framework 7.0.6 API)"))

| 구분      | 설명                                                                       |
| ------- | ------------------------------------------------------------------------ |
| 관리 주체   | Spring Framework                                                         |
| 생성 시점   | Spring Context 초기화 시                                                     |
| 소멸 시점   | Spring Context 종료 시                                                      |
| 범위      | Spring 설정 단위                                                             |
| 주요 용도   | Bean 생성, DI, AOP, Transaction, Event, MessageSource, Resource 관리         |
| 대표 Bean | Service, Repository, DAO, DataSource, TransactionManager, CacheManager 등 |

- 즉 `ApplicationContext`는 단순 저장소가 아니라 Spring 애플리케이션의 실행 기반입니다.
- 
```java
ApplicationContext context = ...;
MemberService memberService = context.getBean(MemberService.class);
```

하지만 일반적인 Spring 코드에서는 위처럼 직접 `getBean()`을 호출하기보다 생성자 주입을 사용하는 것이 더 좋습니다.

```java
@Service
public class OrderService {
    private final MemberService memberService;
    public OrderService(MemberService memberService) {
        this.memberService = memberService;
    }
}
```

## 15. WebApplicationContext란?

`WebApplicationContext`는 웹 환경용 `ApplicationContext`입니다. 공식 Javadoc 기준으로 `WebApplicationContext`는 `ApplicationContext`를 확장하며, `getServletContext()` 메서드를 추가로 제공합니다. 또한 Root `WebApplicationContext`는 부트스트랩 과정에서 잘 알려진 `ServletContext` attribute 이름으로 바인딩됩니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.28/javadoc-api/org/springframework/web/context/WebApplicationContext.html?is-external=true "WebApplicationContext (Spring Framework 5.3.28 API)"))

| 구분                | ApplicationContext                                                   | WebApplicationContext                                             |
| ----------------- | -------------------------------------------------------------------- | ----------------------------------------------------------------- |
| 기본 성격             | Spring Bean Container                                                | 웹 환경용 Spring Bean Container                                       |
| ServletContext 접근 | 직접 기능 없음                                                             | `getServletContext()` 제공                                          |
| 사용 위치             | 일반 Java/Spring 애플리케이션                                                | Spring MVC 웹 애플리케이션                                               |
| 대표 구현             | ClassPathXmlApplicationContext, AnnotationConfigApplicationContext 등 | XmlWebApplicationContext, AnnotationConfigWebApplicationContext 등 |
| 웹 Scope           | 기본적으로 없음                                                             | request, session, application scope 등 웹 Scope와 연계                 |

- 즉, Spring MVC 프로젝트에서 말하는 `ApplicationContext`는 실제로는 `WebApplicationContext`인 경우가 많습니다.
## 16. Root ApplicationContext와 Servlet ApplicationContext

Spring MVC XML 기반 프로젝트에서는 보통 Context가 2개 이상 존재할 수 있습니다.

```text
Root WebApplicationContext
 └─ DispatcherServlet WebApplicationContext
```

### 16-1. Root WebApplicationContext

Root Context는 보통 `ContextLoaderListener`가 생성합니다.

```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/spring/root-context.xml</param-value>
</context-param>
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

Root Context에는 일반적으로 아래 Bean을 둡니다.

```text
Service
Repository / DAO
DataSource
TransactionManager
SqlSessionFactory
CacheManager
Scheduler
공통 Component
```

Spring 공식 API 문서의 `WebApplicationContextUtils`도 Root `WebApplicationContext`는 일반적으로 `ContextLoaderListener`를 통해 로드된다고 설명합니다. ([docs.enterprise.spring.io](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/web/context/support/WebApplicationContextUtils.html "WebApplicationContextUtils (Spring Framework 5.3.46 API)"))

### 16-2. DispatcherServlet WebApplicationContext

DispatcherServlet별 Context는 `DispatcherServlet`이 생성합니다.

```xml
<servlet>
    <servlet-name>appServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/spring/servlet-context.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>appServlet</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

Servlet Context에는 일반적으로 아래 Bean을 둡니다.

```text
Controller
HandlerMapping
HandlerAdapter
ViewResolver
MessageConverter
Interceptor
MultipartResolver
mvc:annotation-driven
mvc:resources
```

## 17. Parent-Child Context 관계

Spring MVC의 전형적인 구조에서는 `DispatcherServlet`의 WebApplicationContext가 Root WebApplicationContext를 부모로 가집니다.

```text
Child Context에서 Bean 조회
  1. 먼저 Child Context에서 찾음
  2. 없으면 Parent, 즉 Root Context에서 찾음
```

그래서 Controller는 Root Context에 있는 Service Bean을 주입받을 수 있습니다.

```java
@Controller
public class OrderController {
    private final OrderService orderService;
    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }
}
```

반대로 Root Context의 Service가 Child Context의 Controller Bean을 참조하는 구조는 일반적으로 맞지 않습니다. Root는 웹 계층에 의존하지 않는 것이 좋습니다.

| 방향                     |       가능 여부 | 설명                       |
| ---------------------- | ----------: | ------------------------ |
| Controller → Service   |          가능 | Child가 Parent Bean 조회 가능 |
| Service → Repository   |          가능 | 보통 둘 다 Root에 있음          |
| Service → Controller   | 비권장/구조상 부적절 | 업무 계층이 웹 계층에 의존함         |
| Root Bean → Child Bean |    일반적으로 불가 | Parent는 Child Bean을 모름   |
| Child Bean → Root Bean |          가능 | Child는 Parent Bean 조회 가능 |

- Spring 공식 문서도 Root Context의 Bean은 Servlet별 Child Context에서 상속되며, Child Context에서 재정의할 수 있다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet/context-hierarchy.html "Context Hierarchy :: Spring Framework"))
## 18. 왜 Context를 여러 개 쓰는가?

Spring MVC에서 Context를 나누는 이유는 계층 분리 때문입니다.

| 목적                      | 설명                                                                          |
| ----------------------- | --------------------------------------------------------------------------- |
| 공통 Bean 공유              | Service, Repository, DataSource 등을 여러 Servlet이 공유                           |
| 웹 설정 분리                 | Controller, ViewResolver, Interceptor 등 웹 전용 Bean을 DispatcherServlet 단위로 분리 |
| 여러 DispatcherServlet 지원 | `/api/*`, `/admin/*`처럼 Servlet별 MVC 설정 분리 가능                                |
| 계층 의존성 정리               | Web Layer가 Service Layer를 참조하고, Service Layer는 Web Layer를 모르게 유지            |

- 실무에서는 보통 하나의 DispatcherServlet만 쓰더라도 Root Context와 Servlet Context를 분리한 레거시 Spring MVC 프로젝트가 많습니다. 특히 Spring 5.3 XML 기반 프로젝트에서는 `root-context.xml`, `dispatcher-servlet.xml`, `servlet-context.xml`, `applicationContext.xml` 같은 파일명이 혼재할 수 있습니다.
## 19. ServletContext와 ApplicationContext의 관계

정확한 관계는 다음과 같습니다.

```text
ServletContext
  └─ Attribute로 Root WebApplicationContext를 보관할 수 있음
       └─ Spring Bean들을 관리함
```

즉 `ServletContext`가 `ApplicationContext`를 “생성하는 컨테이너”는 아닙니다. `ContextLoaderListener` 또는 `DispatcherServlet`이 Spring Context를 만들고, 그 결과 만들어진 `WebApplicationContext`가 `ServletContext`에 연결/등록됩니다.  
Spring의 `WebApplicationContextUtils.getRequiredWebApplicationContext(ServletContext)`는 `ServletContext`에서 Root `WebApplicationContext`를 찾아오는 유틸 메서드입니다. 해당 API는 Root Context를 찾지 못하면 `IllegalStateException`을 던진다고 설명합니다. ([docs.enterprise.spring.io](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/web/context/support/WebApplicationContextUtils.html "WebApplicationContextUtils (Spring Framework 5.3.46 API)"))

```java
ServletContext servletContext = request.getServletContext();
WebApplicationContext context =
    WebApplicationContextUtils.getRequiredWebApplicationContext(servletContext);
OrderService orderService = context.getBean(OrderService.class);
```

하지만 이 방식은 예외적인 경우에만 사용해야 합니다. Spring Bean 내부에서는 `getBean()` 직접 호출보다 의존성 주입이 우선입니다.

## 20. Spring에서 ContextHolder란?

중요한 점은 `ContextHolder`라는 이름 자체는 Spring Framework의 핵심 표준 `ApplicationContext` 객체명이 아닙니다. 실무 프로젝트에서 말하는 `ContextHolder`는 보통 아래 중 하나입니다.

| 유형                     | 예시                                                                 | ApplicationContext와 관계                                |
| ---------------------- | ------------------------------------------------------------------ | ----------------------------------------------------- |
| 커스텀 Holder             | `ApplicationContextHolder`, `SpringContextHolder`, `ContextHolder` | `ApplicationContext`를 static으로 보관하는 프로젝트 유틸           |
| Spring Web Holder      | `RequestContextHolder`                                             | 현재 Thread의 Request 정보를 보관, ApplicationContext 자체와는 다름 |
| Spring i18n Holder     | `LocaleContextHolder`                                              | 현재 Thread의 Locale 정보를 보관                              |
| Spring Security Holder | `SecurityContextHolder`                                            | 현재 인증/보안 Context 보관, Spring Security 영역               |

- `RequestContextHolder`는 현재 웹 요청을 노출하기 위한 Spring Web 유틸이며, `DispatcherServlet`은 기본적으로 현재 request를 노출한다고 Javadoc에 설명되어 있습니다. ([docs.enterprise.spring.io](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/web/context/request/RequestContextHolder.html?utm_source=chatgpt.com "RequestContextHolder (Spring Framework 5.3.46 API)"))
- `SecurityContextHolder`는 Spring Security에서 사용하는 별도 Holder이며, static 메서드를 통해 `SecurityContextHolderStrategy`에 위임하는 구조입니다. JVM-wide 설정이라는 설명도 공식 API에 있습니다. ([Home](https://docs.spring.io/spring-security/reference/api/java/org/springframework/security/core/context/SecurityContextHolder.html?utm_source=chatgpt.com "SecurityContextHolder (spring-security-docs 7.0.5 API)"))
## 21. ApplicationContextHolder 패턴의 관계

실무에서 흔한 커스텀 `ContextHolder`는 보통 이렇게 생겼습니다.

```java
@Component
public class ApplicationContextHolder implements ApplicationContextAware {
    private static ApplicationContext applicationContext;
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        ApplicationContextHolder.applicationContext = applicationContext;
    }
    public static <T> T getBean(Class<T> clazz) {
        return applicationContext.getBean(clazz);
    }
}
```

이 경우 관계는 다음과 같습니다.

```text
Spring ApplicationContext
  └─ ApplicationContextHolder Bean 생성
       └─ ApplicationContextAware.setApplicationContext(...) 호출
            └─ static 변수에 ApplicationContext 저장
                 └─ 비Spring 객체나 static 메서드에서 getBean() 호출 가능
```

`ApplicationContextAware`는 Spring이 해당 객체가 실행되는 `ApplicationContext`를 설정해 주기 위한 인터페이스입니다. 공식 API도 `setApplicationContext(ApplicationContext applicationContext)`는 해당 객체가 실행되는 ApplicationContext를 설정한다고 설명합니다. ([Spring Pleiades](https://spring.pleiades.io/spring-framework/docs/current/javadoc-api/org/springframework/context/ApplicationContextAware.html?utm_source=chatgpt.com "ApplicationContextAware (Spring Framework API) - Javadoc"))

## 22. ContextHolder 사용 시 실무 주의점

`ApplicationContextHolder` 방식은 편리하지만 남용하면 설계 문제가 생깁니다.

| 문제         | 설명                                                                           |
| ---------- | ---------------------------------------------------------------------------- |
| DI 우회      | 생성자 주입 대신 어디서든 `getBean()`을 호출하게 되어 의존성이 숨겨짐                                 |
| 테스트 어려움    | 단위 테스트에서 static ApplicationContext 상태가 남아 테스트 격리가 어려워짐                       |
| Context 혼동 | Root Context와 DispatcherServlet Child Context 중 어느 Context가 들어갔는지 불명확해질 수 있음 |
| 초기화 순서 문제  | Context 초기화 전 static getBean 호출 시 NPE 또는 초기화 오류 가능                           |
| 메모리 누수 위험  | 재배포 환경에서 static 참조가 ClassLoader 해제를 방해할 수 있음                                 |
| AOP 오해     | Bean이 아닌 객체를 직접 new 해서 쓰면 Spring AOP, Transaction, Cache 적용 대상이 아님           |
| 계층 침범      | Service/Domain 코드가 Spring Container에 직접 의존하면 순수성이 떨어짐                        |

- 특히 JBoss 같은 WAS 재배포 환경에서는 static holder가 이전 ClassLoader의 객체를 계속 참조하는 문제가 생길 수 있어 주의해야 합니다.
## 23. ContextHolder를 써도 되는 경우

완전히 금지할 필요는 없지만, 사용 범위는 제한하는 것이 좋습니다.

| 상황                                            | 판단                                                          |
| --------------------------------------------- | ----------------------------------------------------------- |
| Spring Bean 내부에서 Service를 가져오려고 사용            | 비권장, 생성자 주입 사용                                              |
| static util에서 DAO/Service 호출                  | 비권장, util을 Bean으로 전환 권장                                     |
| 레거시 non-Spring 객체에서 부득이하게 Spring Bean 접근      | 제한적 허용                                                      |
| 서드파티 콜백 객체에서 Spring Bean 접근                   | 제한적 허용                                                      |
| Filter/Listener가 Spring Bean이 아닌 상태에서 Bean 접근 | `DelegatingFilterProxy`, `WebApplicationContextUtils` 우선 검토 |
| 테스트 코드에서 편의상 사용                               | 비권장, TestContext 또는 DI 사용                                   |

## 24. 실무 권장 구조

### 24-1. 권장: 생성자 주입

```java
@Service
public class MailService {
    private final MailRepository mailRepository;
    public MailService(MailRepository mailRepository) {
        this.mailRepository = mailRepository;
    }
}
```

### 24-2. 제한적 허용: Spring 밖 객체에서만 Holder 사용

```java
public class LegacyCallback {
    public void execute() {
        MailService mailService = ApplicationContextHolder.getBean(MailService.class);
        mailService.send();
    }
}
```

### 24-3. 더 나은 개선 방향

```text
1. static util을 @Component Bean으로 변경
2. new로 생성하던 객체를 Spring Bean으로 등록
3. 외부 라이브러리 콜백은 Adapter Bean으로 감싸기
4. Filter는 DelegatingFilterProxy 사용 검토
5. ApplicationContext 직접 접근은 WebApplicationContextUtils로 범위 명확화
```

## 25. Spring 5.3 XML 프로젝트 기준 점검 포인트

아래는 실제 프로젝트에서 확인해야 할 항목입니다.

```text
web.xml
 ├─ ContextLoaderListener 존재 여부
 ├─ contextConfigLocation 위치
 ├─ DispatcherServlet 설정 위치
 ├─ DispatcherServlet init-param contextConfigLocation
 └─ servlet-name에 따른 xxx-servlet.xml 자동 로딩 여부
```

```text
root-context.xml 또는 applicationContext.xml
 ├─ service/repository component-scan 포함 여부
 ├─ dataSource 등록 여부
 ├─ transactionManager 등록 여부
 ├─ tx:annotation-driven 위치
 ├─ cache:annotation-driven 위치
 └─ mybatis/sqlSessionFactory 위치
```

```text
dispatcher-servlet.xml 또는 servlet-context.xml
 ├─ controller component-scan 포함 여부
 ├─ mvc:annotation-driven 위치
 ├─ mvc:resources 위치
 ├─ interceptor 위치
 ├─ viewResolver 위치
 └─ multipartResolver 위치
```

## 26. 자주 발생하는 문제

### 26-1. 문제 1. Service에 `@Transactional`이 적용되지 않음

`@Transactional`을 활성화하는 `<tx:annotation-driven/>`이 Service Bean이 존재하는 Context가 아니라 DispatcherServlet Context에만 있으면 Service에 적용되지 않을 수 있습니다. Spring의 Context 계층에서는 설정이 놓인 Context와 Bean이 생성된 Context의 위치가 중요합니다.

```text
잘못된 예
dispatcher-servlet.xml
 ├─ <tx:annotation-driven/>
root-context.xml
 └─ @Service Bean
```

```text
권장 예
root-context.xml
 ├─ <tx:annotation-driven transaction-manager="txManager"/>
 └─ @Service Bean
```

### 26-2. 문제 2. Controller가 Service를 못 찾음

Root Context에 Service가 없거나, Service component-scan이 DispatcherServlet Context와 분리되어 잘못 설정된 경우 발생합니다.

```xml
<context:component-scan base-package="com.example.service"/>
```

Service는 Root Context에서 scan하고, Controller는 Servlet Context에서 scan하는 구성이 명확합니다.

### 26-3. 문제 3. 같은 Bean이 Root와 Child에 중복 등록됨

같은 Service가 Root와 Child 양쪽에 등록되면 서로 다른 Bean 인스턴스가 생길 수 있습니다. 이 경우 Transaction, Cache, AOP 적용 여부가 예상과 달라질 수 있습니다.

```text
root-context.xml      → com.example 전체 scan
dispatcher-servlet.xml → com.example 전체 scan
```

이런 설정은 피하고, 계층별 scan 범위를 분리하는 것이 좋습니다.

```text
root-context.xml      → service, repository, infra
dispatcher-servlet.xml → controller, web
```

## 27. 최종 정리

| 항목                    | 핵심                                                          |
| --------------------- | ----------------------------------------------------------- |
| ServletContext        | WAS가 관리하는 웹 애플리케이션 전역 Context                               |
| ApplicationContext    | Spring이 관리하는 Bean Container                                 |
| WebApplicationContext | 웹 환경용 ApplicationContext이며 ServletContext와 연결됨              |
| Root Context          | Service, Repository, DataSource, Transaction 등 공통 Bean 영역   |
| Servlet Context       | Controller, MVC 설정, ViewResolver 등 웹 계층 영역                  |
| ContextHolder         | 대부분 프로젝트 커스텀 유틸이며 ApplicationContext를 static으로 보관하는 패턴      |
| 권장 방식                 | Spring Bean 내부에서는 Holder보다 생성자 주입 사용                        |
| 주의점                   | Context 계층, Bean 중복 등록, AOP/Transaction 적용 위치, static 참조 누수 |

- 핵심만 말하면, `ServletContext`는 **웹 애플리케이션의 실행 환경**, `ApplicationContext`는 **Spring Bean 관리 환경**, `WebApplicationContext`는 **둘을 연결하는 Spring MVC용 Context**입니다. `ContextHolder`는 Spring의 필수 구성요소가 아니라, 보통 `ApplicationContext`를 정적으로 꺼내 쓰기 위해 프로젝트에서 만든 보조 클래스이며, 실무에서는 꼭 필요한 경계 영역에서만 제한적으로 사용하는 것이 안전합니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
