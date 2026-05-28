---
layout: single
title: "ApplicationContext_의_종류"
excerpt: "ApplicationContext_의_종류"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-05-28"
last_modified_at: "2026-05-28 12:08:16 +0900"
---
### 1. 핵심 개념

Spring MVC 기반 Web Application에서는 `ApplicationContext`가 **1개만 있을 수도 있고**, 보통 XML 기반 레거시/Spring 5.3 프로젝트에서는 **Root ApplicationContext + DispatcherServlet별 WebApplicationContext** 구조로 여러 개가 존재할 수 있습니다.  
Spring 공식 문서 기준으로 `<cache:annotation-driven/>`은 **자신이 선언된 ApplicationContext 안의 Bean만 대상으로** `@Cacheable`, `@CachePut`, `@CacheEvict`, `@Caching`을 찾습니다. 그래서 이 설정을 `DispatcherServlet`용 `WebApplicationContext`에만 넣으면 Controller Bean은 보지만 Service Bean은 보지 못할 수 있습니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/integration.html "Integration"))

### 2. ApplicationContext란 무엇인가

`ApplicationContext`는 Spring Bean을 생성·관리하는 컨테이너입니다.

```text
ApplicationContext
= Spring Bean 등록/생성/의존성 주입/AOP 적용/라이프사이클 관리 공간
```

즉, 아래 Bean들이 어디에 등록되는지에 따라 AOP 적용 범위가 달라집니다.

```text
@Controller
@Service
@Repository
TransactionManager
CacheManager
DataSource
SqlSessionFactory
ViewResolver
HandlerMapping
```

### 3. Spring MVC에서 주로 나오는 Context 구조

Spring MVC 공식 문서에서는 `DispatcherServlet`이 자기 설정을 위한 `WebApplicationContext`를 기대한다고 설명하고, 하나의 Root `WebApplicationContext`를 여러 `DispatcherServlet`이 공유하면서 각 Servlet이 별도 Child `WebApplicationContext`를 가질 수 있다고 설명합니다. Root Context에는 보통 repository, business service 같은 공유 Bean이 들어가고, Servlet별 Child Context에는 해당 Servlet에 로컬인 MVC Bean이 들어갑니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/web.html "Web on Servlet Stack"))

#### 3-1. 일반적인 구조

```text
ServletContext
└─ Root WebApplicationContext
   ├─ DataSource
   ├─ TransactionManager
   ├─ CacheManager
   ├─ Service
   ├─ Repository / Mapper
   └─ Batch / Scheduler / 공통 Bean
      ▲
      │ parent
      │
      └─ DispatcherServlet WebApplicationContext
         ├─ Controller
         ├─ HandlerMapping
         ├─ HandlerAdapter
         ├─ ViewResolver
         ├─ MessageConverter
         └─ Interceptor
```

### 4. 왜 여러 개의 ApplicationContext를 쓰는가

#### 4-1. 역할 분리

Root Context는 업무/인프라 영역, DispatcherServlet Context는 Web MVC 영역을 담당하게 분리합니다.

|구분|주로 포함되는 Bean|이유|
|---|---|---|
|Root Context|`Service`, `Repository`, `DataSource`, `TransactionManager`, `CacheManager`|여러 Web 계층에서 공유되는 핵심 업무/인프라|
|Servlet Context|`Controller`, `ViewResolver`, `HandlerMapping`, `Interceptor`|특정 DispatcherServlet의 요청 처리 전용|

#### 4-2. 여러 DispatcherServlet 지원

관리자 화면, 사용자 화면, API 화면을 서로 다른 `DispatcherServlet`으로 분리할 수 있습니다.

```text
/fo/*  → fo DispatcherServlet
/bo/*  → bo DispatcherServlet
/api/* → api DispatcherServlet
```

각 Servlet마다 Controller, ViewResolver, Interceptor 설정이 다를 수 있습니다.

#### 4-3. 공통 Bean 공유

Root Context에 `Service`, `Repository`, `DataSource`를 두면 여러 DispatcherServlet이 같은 업무 Bean을 재사용할 수 있습니다. Spring 공식 문서도 Root `WebApplicationContext`가 여러 Servlet 인스턴스에서 공유될 수 있고, Root의 Bean은 Servlet별 Child Context에서 상속될 수 있다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/web.html "Web on Servlet Stack"))

### 5. XML 기반 프로젝트에서 확인하는 방법

#### 5-1. `web.xml`에서 Root Context 확인

아래 설정이 있으면 Root ApplicationContext가 생성됩니다.

```xml
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>
        classpath*:egovframework/spring/context-*.xml
        /WEB-INF/config/spring/context-*.xml
    </param-value>
</context-param>
```

의미:

```text
ContextLoaderListener
→ Root WebApplicationContext 생성
contextConfigLocation
→ Root Context가 읽을 XML 목록
```

Spring 5.3 공식 문서의 `web.xml` 예시도 `ContextLoaderListener`와 `contextConfigLocation`으로 Root Context를 구성하고, 별도로 `DispatcherServlet`의 `contextConfigLocation`을 지정하는 구조를 보여줍니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/web.html "Web on Servlet Stack"))

#### 5-2. `web.xml`에서 DispatcherServlet Context 확인

아래 설정이 있으면 DispatcherServlet 전용 Child WebApplicationContext가 생성됩니다.

```xml
<servlet>
    <servlet-name>action</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/config/springmvc/dispatcher-servlet.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>action</servlet-name>
    <url-pattern>*.do</url-pattern>
</servlet-mapping>
```

의미:

```text
DispatcherServlet 이름: action
Servlet 전용 Context XML: /WEB-INF/config/springmvc/dispatcher-servlet.xml
처리 URL: *.do
```

만약 `DispatcherServlet`의 `contextConfigLocation`이 생략되면 일반적으로 `/WEB-INF/[servlet-name]-servlet.xml` 형태를 찾는 구조가 사용됩니다.

### 6. 프로젝트에서 Context 확인용 검색 명령

#### 6-1. Windows PowerShell

```powershell
Get-ChildItem -Path . -Recurse -Include web.xml,*.xml,*.java |
  Select-String -Pattern 'ContextLoaderListener|contextConfigLocation|DispatcherServlet|cache:annotation-driven|tx:annotation-driven|component-scan|@EnableCaching|@EnableTransactionManagement'
```

검증 포인트:

```text
PowerShell Select-String은 기본적으로 대소문자를 구분하지 않음
정확한 대소문자까지 확인하려면 -CaseSensitive 추가
XML 태그명은 대소문자에 민감하므로 실제 설정 파일에서는 원문 그대로 확인 필요
```

대소문자까지 엄격히 검색:

```powershell
Get-ChildItem -Path . -Recurse -Include web.xml,*.xml,*.java |
  Select-String -CaseSensitive -Pattern 'ContextLoaderListener|contextConfigLocation|DispatcherServlet|cache:annotation-driven|tx:annotation-driven|component-scan|@EnableCaching|@EnableTransactionManagement'
```

#### 6-2. Linux

```bash
grep -RInE "ContextLoaderListener|contextConfigLocation|DispatcherServlet|cache:annotation-driven|tx:annotation-driven|component-scan|@EnableCaching|@EnableTransactionManagement" .
```

### 7. 실제로 어떤 XML이 어떤 Context인지 구분하는 기준

#### 7-1. Root Context에 속하는 XML

`web.xml`의 아래 영역에 포함된 XML입니다.

```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>
        ...
    </param-value>
</context-param>
```

보통 이름 예시:

```text
applicationContext.xml
context-common.xml
context-datasource.xml
context-transaction.xml
context-mapper.xml
context-cache.xml
context-scheduler.xml
```

주로 들어가는 설정:

```xml
<context:component-scan base-package="bk.app">
    <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
<bean id="dataSource" .../>
<bean id="txManager" .../>
<bean id="ehCacheManager" .../>
<cache:annotation-driven cache-manager="ehCacheManager"/>
<tx:annotation-driven transaction-manager="txManager"/>
```

#### 7-2. DispatcherServlet Context에 속하는 XML

`web.xml`의 아래 영역에 포함된 XML입니다.

```xml
<servlet>
    <servlet-name>...</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>...</param-value>
    </init-param>
</servlet>
```

보통 이름 예시:

```text
dispatcher-servlet.xml
action-servlet.xml
servlet-context.xml
egov-com-servlet.xml
springmvc.xml
```

주로 들어가는 설정:

```xml
<context:component-scan base-package="bk.app">
    <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
<mvc:annotation-driven/>
<mvc:resources mapping="/css/**" location="/css/"/>
<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver"/>
```

### 8. `<cache:annotation-driven/>` 위치가 중요한 이유

#### 8-1. Service가 Root Context에 있는 경우

예를 들어 `GoodsService`가 Root Context에 등록되어 있다고 가정합니다.

```text
Root Context
└─ GoodsService
DispatcherServlet Context
└─ GoodsController
```

이때 `<cache:annotation-driven/>`이 DispatcherServlet Context XML에만 있으면 아래처럼 됩니다.

```text
dispatcher-servlet.xml
└─ <cache:annotation-driven/>
   └─ Controller Bean만 검사
      └─ GoodsService는 Root Context Bean이라 대상 아님
```

결과:

```java
@Service
public class GoodsService {
    @Cacheable(cacheNames = "GOODS_UNIT", key = "#goodsSn")
    public GoodsUnitVO selectGoodsUnit(Long goodsSn) {
        ...
    }
}
```

위 `@Cacheable`이 적용되지 않을 수 있습니다.  
Spring 5.3 공식 문서는 `<cache:annotation-driven/>`이 자신이 정의된 ApplicationContext 안의 Bean에서만 캐시 애노테이션을 찾으며, DispatcherServlet의 `WebApplicationContext`에 두면 Controller만 확인하고 Service는 확인하지 않는다고 명시합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/integration.html "Integration"))

#### 8-2. 올바른 위치

Service에 `@Cacheable`을 적용하려면 `<cache:annotation-driven/>`은 Service Bean이 등록된 Context에 있어야 합니다.

```text
Service가 Root Context에 있음
→ Root Context XML에 <cache:annotation-driven/> 선언
Service가 DispatcherServlet Context에 있음
→ 해당 DispatcherServlet Context XML에 선언
```

실무에서는 보통 Service는 Root Context에 두므로 아래가 일반적입니다.

```xml
<!-- context-cache.xml 또는 applicationContext.xml 계열 -->
<cache:annotation-driven cache-manager="ehCacheManager"/>
<bean id="ehCacheManager" class="org.springframework.cache.jcache.JCacheCacheManager">
    ...
</bean>
```

### 9. 자주 쓰이는 구성 예시

#### 9-1. 예시 1. 표준적인 XML 기반 Spring MVC

```text
web.xml
├─ ContextLoaderListener
│  └─ Root Context
│     ├─ context-datasource.xml
│     ├─ context-transaction.xml
│     ├─ context-mapper.xml
│     ├─ context-cache.xml
│     └─ context-service.xml
└─ DispatcherServlet
   └─ Servlet Context
      ├─ dispatcher-servlet.xml
      ├─ mvc-controller.xml
      └─ mvc-view.xml
```

권장 배치:

```text
Root Context
→ Service, Repository, DB, Transaction, Cache
Servlet Context
→ Controller, ViewResolver, HandlerMapping, Interceptor
```

#### 9-2. 예시 2. FO/BO가 나뉜 프로젝트

```text
Root Context
├─ 공통 Service
├─ 공통 DAO/Mapper
├─ DataSource
├─ TransactionManager
└─ CacheManager
FO DispatcherServlet Context
├─ FO Controller
├─ FO ViewResolver
└─ FO Interceptor
BO DispatcherServlet Context
├─ BO Controller
├─ BO ViewResolver
└─ BO Interceptor
```

이 구조에서 `<cache:annotation-driven/>`을 FO Servlet Context에만 넣으면 BO Controller나 Root Service에는 적용되지 않습니다. Service 캐싱을 공통 적용하려면 Root Context에 넣는 것이 맞습니다.

#### 9-3. 예시 3. 단일 Context 구성

Spring 공식 문서는 Context hierarchy가 필요 없으면 모든 설정을 Root 쪽에 두고 Servlet의 `contextConfigLocation`을 비울 수 있다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/web.html "Web on Servlet Stack"))

```text
Root Context 하나만 사용
└─ Controller, Service, Repository, MVC 설정 모두 포함
```

장점:

```text
구조 단순
annotation-driven 위치 혼선 적음
```

단점:

```text
FO/BO/API 분리 어려움
Web 설정과 업무 설정이 섞임
대형 XML 프로젝트에서는 유지보수 어려움
```

### 10. 현재 프로젝트에서 확인해야 할 핵심 포인트

#### 10-1. `@Cacheable`을 붙인 Service가 어느 Context에 있는가

아래를 확인합니다.

```powershell
Get-ChildItem -Path . -Recurse -Include *.xml |
  Select-String -Pattern 'base-package|include-filter|exclude-filter|Controller|Service|Repository'
```

확인할 내용:

```text
Service 패키지를 scan하는 XML이 Root Context에 포함되어 있는가?
Controller 패키지를 scan하는 XML이 DispatcherServlet Context에 포함되어 있는가?
Controller와 Service를 같은 Context에서 중복 scan하고 있지 않은가?
```

#### 10-2. `cache:annotation-driven`이 어느 XML에 있는가

```powershell
Get-ChildItem -Path . -Recurse -Include *.xml |
  Select-String -Pattern 'cache:annotation-driven'
```

찾은 XML이 어디에 물려 있는지 `web.xml`에서 역추적합니다.

```text
context-param > contextConfigLocation에 포함됨
→ Root Context
servlet > init-param > contextConfigLocation에 포함됨
→ DispatcherServlet Context
둘 다 아님
→ 실제 로딩 안 될 가능성
```

#### 10-3. `ehCacheManager` Bean이 어느 Context에 있는가

```powershell
Get-ChildItem -Path . -Recurse -Include *.xml |
  Select-String -Pattern 'ehCacheManager|JCacheCacheManager|JCacheManagerFactoryBean'
```

점검 기준:

```text
cache:annotation-driven과 ehCacheManager가 같은 Context에 있는가?
Child Context에서 Root Context의 ehCacheManager를 참조하는 구조인가?
동일 id의 CacheManager가 Root/Child에 중복 선언되어 있지 않은가?
```

### 11. 실무에서 자주 발생하는 문제

|문제|원인|증상|
|---|---|---|
|`@Cacheable` 미동작|`<cache:annotation-driven/>`이 Servlet Context에만 있음|Service 메서드가 매번 DB 조회|
|트랜잭션 미동작|`<tx:annotation-driven/>` 위치 오류|Service `@Transactional` 무시|
|Bean 중복 생성|Root와 Child가 같은 패키지를 둘 다 scan|Controller/Service가 중복 등록|
|CacheManager 불일치|Root/Child에 CacheManager 중복 선언|캐시 저장/삭제가 서로 안 맞음|
|Controller에서는 됨|Controller가 Child Context Bean이라 적용됨|Service에는 적용 안 됨|
|테스트에서는 됨|테스트 Context가 단일 Context|운영 Web Context 구조와 다름|

### 12. 특히 `component-scan` 분리가 중요함

#### 12-1. 잘못된 예

Root와 Servlet Context가 같은 패키지를 모두 scan합니다.

```xml
<!-- root-context.xml -->
<context:component-scan base-package="bk.app"/>
<!-- dispatcher-servlet.xml -->
<context:component-scan base-package="bk.app"/>
```

위 구조는 Controller, Service, Repository가 Root와 Child에 중복 등록될 수 있습니다.

#### 12-2. 권장 예

Root Context:

```xml
<context:component-scan base-package="bk.app">
    <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```

DispatcherServlet Context:

```xml
<context:component-scan base-package="bk.app">
    <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
</context:component-scan>
```

단, 실제 XML에서는 `use-default-filters` 설정까지 함께 봐야 합니다. Controller만 include하려면 보통 아래처럼 명확히 합니다.

```xml
<context:component-scan base-package="bk.app" use-default-filters="false">
    <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```

### 13. 런타임에서 직접 확인하는 방법

#### 13-1. Context 구조 출력 Bean 추가

개발/로컬 환경에서만 아래 Bean을 임시로 추가하면 현재 Bean이 어느 Context에 있는지 확인할 수 있습니다.

```java
@Component
public class ApplicationContextCheck implements ApplicationContextAware {
    private ApplicationContext applicationContext;
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }
    @PostConstruct
    public void printContextInfo() {
        System.out.println("Current Context ID = " + applicationContext.getId());
        System.out.println("Current Context Class = " + applicationContext.getClass().getName());
        ApplicationContext parent = applicationContext.getParent();
        System.out.println("Parent Context ID = " + (parent != null ? parent.getId() : "null"));
        System.out.println("Bean Count = " + applicationContext.getBeanDefinitionCount());
    }
}
```

#### 13-2. 특정 Bean이 어느 Context에 있는지 확인

```java
@PostConstruct
public void checkBeans() {
    printBeanLocation("goodsService");
    printBeanLocation("ehCacheManager");
    printBeanLocation("cacheOperationSource");
}
private void printBeanLocation(String beanName) {
    boolean current = applicationContext.containsLocalBean(beanName);
    boolean parent = applicationContext.getParent() != null
        && applicationContext.getParent().containsBean(beanName);
    System.out.println(beanName + " current=" + current + ", parent=" + parent);
}
```

해석:

```text
current=true
→ 현재 Context에 직접 등록된 Bean
parent=true
→ Parent Context에서 상속받아 보이는 Bean
containsBean=true지만 containsLocalBean=false
→ 상위 Context Bean일 가능성
```

### 14. Cache 관점에서 올바른 배치

#### 14-1. Service 캐싱을 쓰는 경우

```text
Root Context
├─ GoodsService
├─ ehCacheManager
└─ <cache:annotation-driven cache-manager="ehCacheManager"/>
```

이 구조가 가장 안전합니다.

#### 14-2. Controller 캐싱만 쓰는 경우

```text
DispatcherServlet Context
├─ GoodsController
├─ ehCacheManager 또는 Root의 ehCacheManager 참조
└─ <cache:annotation-driven cache-manager="ehCacheManager"/>
```

하지만 Controller에 캐시를 거는 것은 보통 권장 우선순위가 낮습니다. 캐시는 업무 메서드 단위, 즉 Service 계층에 두는 것이 재사용성과 일관성 측면에서 더 안전합니다.

### 15. Transaction도 같은 원리

`<tx:annotation-driven/>`도 같은 이슈가 있습니다. Spring 공식 문서는 `<tx:annotation-driven/>` 또는 `@EnableTransactionManagement` 역시 자신이 정의된 ApplicationContext 안의 Bean에서만 `@Transactional`을 찾는다고 설명합니다. DispatcherServlet Context에만 두면 Controller는 보지만 Service는 보지 못할 수 있습니다. ([Home](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html?utm_source=chatgpt.com "Using @Transactional :: Spring Framework"))  
따라서 실무에서는 보통 아래처럼 배치합니다.

```text
Root Context
├─ <tx:annotation-driven transaction-manager="txManager"/>
├─ <cache:annotation-driven cache-manager="ehCacheManager"/>
├─ txManager
├─ ehCacheManager
├─ Service
└─ Repository
```

### 16. 최종 점검 순서

```text
1. web.xml에서 ContextLoaderListener 존재 여부 확인
2. context-param의 contextConfigLocation 확인
3. DispatcherServlet의 init-param contextConfigLocation 확인
4. 각 XML이 Root Context인지 Servlet Context인지 분류
5. Service Bean이 어느 XML의 component-scan으로 등록되는지 확인
6. Controller Bean이 어느 XML의 component-scan으로 등록되는지 확인
7. <cache:annotation-driven/> 위치 확인
8. ehCacheManager Bean 위치 확인
9. @Cacheable 대상 메서드가 public인지 확인
10. 같은 클래스 내부 호출 self-invocation 여부 확인
11. Root/Child에 같은 Bean id 또는 같은 패키지 중복 scan 여부 확인
```

### 17. 결론

`<cache:annotation-driven/>`은 전역 JVM 설정이 아니라 **해당 ApplicationContext 안에서 Bean 후처리/AOP 프록시를 만드는 설정**으로 이해해야 합니다. Spring MVC XML 프로젝트에서는 보통 Root Context와 DispatcherServlet Context가 분리되어 있으므로, Service에 `@Cacheable`을 적용하려면 `<cache:annotation-driven/>`을 Service가 등록된 Root Context 쪽 XML에 두는 것이 안전합니다. 반대로 DispatcherServlet XML에만 두면 Controller에는 적용될 수 있지만 Root Context에 있는 Service에는 적용되지 않아 “캐시 설정은 있는데 Service 캐싱이 안 되는” 상황이 발생할 수 있습니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
