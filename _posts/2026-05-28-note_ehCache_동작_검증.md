---
layout: single
title: "ehCache_동작_검증"
excerpt: "ehCache_동작_검증"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-05-28"
last_modified_at: "2026-05-28 10:55:59 +0900"
---
## 1. ehCache 설정

```xml
  <cache:annotation-driven cache-manager="ehCacheManager" />
    <bean id="ehCacheManager" class="org.springframework.cache.jcache.JCacheCacheManager">
        <property name="cacheManager">
            <bean class="org.springframework.cache.jcache.JCacheManagerFactoryBean" p:cacheManagerUri="classpath:egovframework/cache/ehcache-config.xml" />
        </property>
    </bean>
```

Spring 프레임워크에서 XML 설정을 통해 Ehcache(JSR-107 JCache API 기반)를 잘 구성하셨습니다. 제시해주신 설정을 바탕으로, 실무에서 이 캐시를 **어떻게 사용(적용)하는지**와 **실무 적용 시 반드시 피해야 할 주의점**을 핵심만 명확하게 정리해 드리겠습니다.

### 1-1. Spring Ehcache 실무 사용법 (애노테이션 기반)

`<cache:annotation-driven />`을 설정했기 때문에, 자바 코드에서는 Spring이 제공하는 캐시 애노테이션을 사용하여 아주 간단하게 적용할 수 있습니다.

#### 1-1-1. ① `@Cacheable` : 캐시 등록 및 조회

메서드 호출 전 캐시를 확인하고, 데이터가 있으면 메서드를 실행하지 않고 캐시된 값을 반환합니다. 데이터가 없으면 메서드를 실행 후 결과를 캐시에 저장합니다.

Java

```
// ehcache-config.xml에 정의된 캐시 이름(value)을 지정합니다.
@Cacheable(value = "userCache", key = "#userId")
public UserVO selectUser(String userId) {
    // 캐시가 없다면 이 비즈니스 로직(DB 조회 등)이 실행됩니다.
    return userMapper.selectUser(userId);
}
```

#### 1-1-2. ② `@CachePut` : 캐시 업데이트

메서드를 항상 실행하고, 그 실행 결과를 캐시에 업데이트합니다. 주로 데이터 수정(Update) 로직에 사용됩니다.

Java

```
@CachePut(value = "userCache", key = "#userVO.userId")
public UserVO updateUser(UserVO userVO) {
    userMapper.updateUser(userVO);
    return userVO; // 반환되는 이 객체가 캐시에 저장됩니다.
}
```

#### 1-1-3. ③ `@CacheEvict` : 캐시 삭제

데이터가 삭제되거나 변경되어 기존 캐시를 완전히 비워야 할 때 사용합니다.

Java

```
// 특정 데이터만 캐시에서 삭제할 때
@CacheEvict(value = "userCache", key = "#userId")
public void deleteUser(String userId) {
    userMapper.deleteUser(userId);
}

// 해당 캐시 저장소의 모든 데이터를 한 번에 비울 때 (allEntries = true)
@CacheEvict(value = "userCache", allEntries = true)
public void clearUserCache() {
    // 캐시 전체 초기화 로직
}
```

### 1-2. 실무 적용 시 ⚠️최고 중요⚠️ 주의 사항

Ehcache를 실무에 도입할 때 가장 많이 실수하고, 이로 인해 장애나 버그가 발생하는 포인트들입니다.

#### 1-2-1. 내부 메서드 호출 시 캐시 작동 안 함 (Self-Invocation 문제)

Spring의 캐시는 AOP(Proxy 방식)로 동작합니다. 외부에서 객체의 메서드를 호출할 때 Proxy가 중간에 가로채서 캐시 처리를 해주는 원리입니다.

- **문제 상황:** 같은 클래스(Service) 내부에서 `A()` 메서드가 `@Cacheable`이 붙은 `B()` 메서드를 호출하면 캐시가 작동하지 않고 매번 DB를 조회합니다.

- **해결책:** 캐시가 적용될 메서드는 반드시 외부에서 호출되는 구조(별도 서비스로 분리 등)로 설계해야 합니다.

#### 1-2-2. 직렬화(Serialization) 구현 필수

Ehcache 설정에 따라 캐시 데이터를 메모리(Heap) 외에 오프힙(Off-Heap)이나 디스크(Disk)에 저장하게 될 수 있습니다.

- **주의점:** 캐시 대상이 되는 객체(VO, DTO 등)는 반드시 `Serializable` 인터페이스를 구현해야 합니다. 구현하지 않으면 캐시 저장 시 `NotSerializableException` 장애가 발생합니다.

Java

```
public class UserVO implements Serializable {
    private static final long serialVersionUID = 1L; // 설정 권장
    private String userId;
    // ...
}
```

#### 1-2-3. 힙 메모리(Heap Memory) OutOfMemoryError (OOM) 조심

Ehcache는 기본적으로 로컬 서버의 메모리를 사용합니다.

- **주의점:** `ehcache-config.xml` 설정 시 캐시의 최대 개수(`maxEntriesLocalHeap`)나 용량을 무제한으로 설정하거나 너무 크게 잡으면, 사용자가 몰릴 때 서버 메모리가 가득 차서 서버가 다운(OOM)될 수 있습니다.

- **대책:** 서비스의 트래픽과 데이터 크기를 고려하여 적절한 `maxEntries` 값을 제한하고, TTL(Time To Live, 만료시간)을 반드시 설정하세요.

#### 1-2-4. 멀티 서버(다중 WAS) 환경에서의 데이터 불일치

Ehcache의 기본 설정은 **로컬 캐시**입니다. 즉, WAS 1호기와 WAS 2호기가 각각 자기 메모리에 캐시를 들고 있습니다.

- **문제 상황:** 사용자가 WAS 1번에서 데이터를 수정하여 1번의 캐시는 갱신되었지만, WAS 2번의 캐시는 과거 데이터를 그대로 들고 있어서 사용자가 접속할 때마다 데이터가 맞지 않는 현상이 발생할 수 있습니다.

- **대책:** * 자주 변경되지 않는 '공통 코드', '약관' 같은 마스터성 데이터에만 Ehcache를 적용합니다.

    - 실시간으로 계속 변하고 정합성이 중요한 데이터는 Redis 같은 **글로벌 공유 캐시**를 도입하는 것이 맞습니다.

#### 1-2-5. Key 관리 및 멱등성 주의

`@Cacheable`에서 `key`를 지정하지 않으면 Spring은 메서드의 파라미터들을 조합해서 자동으로 키를 생성합니다.

- **주의점:** 파라미터가 없는 메서드이거나, 파라미터 중 캐시 키로 쓰이면 안 되는 값(예: 현재 시간, HttpServletRequest 등)이 섞여 있으면 캐시가 꼬이거나 무용지물이 됩니다. 명확하게 구별되는 고유 ID 값(`key = "#userId"`)을 매핑하는 습관이 중요합니다.

## 2. 동작 로직 검증
```java
@Resource(name = "ehCache")
private final CacheService cacheService;

private final String CACHE_KEY_ALIAS = "PRD_UNIT_";

private List<GoodsUnitVO> selectGoodsUnitListForCache(GoodsUnitParamVO goodsUnitParamVo)
    throws Exception {
  List<GoodsUnitVO> listGoodsUnitVo = new ArrayList<GoodsUnitVO>();

  if (goodsUnitParamVo == null
      || goodsUnitParamVo.getGoodsList() == null
      || goodsUnitParamVo.getGoodsList().size() == 0) {
    return listGoodsUnitVo;
  }

  List<GoodsUnitParamVO> goodsListDB = new ArrayList<GoodsUnitParamVO>();
  List<GoodsUnitParamVO> unitParamVo = goodsUnitParamVo.getGoodsList();

  for (GoodsUnitParamVO vo : unitParamVo) {
    if (StringUtils.isNotEmpty(String.valueOf(vo.getGoodsSn()))) {
      Object cacheData =
          cacheService.get(
              CommonConstants.CACHE_ALIAS_GOODS_UNIT,
              this.CACHE_KEY_ALIAS + String.valueOf(vo.getGoodsSn()));
      if (cacheData != null) {
        GoodsUnitVO unitVo = (GoodsUnitVO) cacheData;
        listGoodsUnitVo.add(unitVo);

      } else {
        goodsListDB.add(vo);
      }
    }
  }

  // DB축출
  if (goodsListDB.size() > 0) {
    goodsUnitParamVo.setGoodsList(goodsListDB);
    List<GoodsUnitVO> listGoodsUnitVoDB = this.selectGoodsUnitListForDB(goodsUnitParamVo);
    if (listGoodsUnitVoDB != null) {
      for (GoodsUnitVO vo : listGoodsUnitVoDB) {
        listGoodsUnitVo.add(vo);
        cacheService.set(
            CommonConstants.CACHE_ALIAS_GOODS_UNIT,
            this.CACHE_KEY_ALIAS + String.valueOf(vo.getGoodsSn()),
            vo); // 객체저장
      }
    }
  }

  return listGoodsUnitVo;
}

```

제공해주신 소스 코드는 캐시(ehCache)를 활용하여 DB 조회 부하를 줄이려는 좋은 접근 방식을 취하고 있습니다. 하지만 **사이드 이펙트(Side Effect), 성능 저하 요소, 그리고 가독성 측면**에서 몇 가지 중요한 문제점들이 보입니다.

요청하신 대로 **문제점을 정리**하고, **함수형 람다(Stream API)를 적극 활용**하여 안전하고 깔끔하게 개선한 로직을 공유해 드립니다.

## 3. 기존 코드의 문제점 분석

### 3-1. ⚠️ 파라미터 변조 (Side Effect)

- **문제점:** `goodsUnitParamVo.setGoodsList(goodsListDB);` 부분에서 메서드 외부에서 넘어온 파라미터 객체의 내부 상태를 변경하고 있습니다.

- **영향:** 이 메서드를 호출한 상위 레이어에서 `goodsUnitParamVo`를 재사용할 경우, 원본 리스트가 누락되어 심각한 비즈니스 로직 오류를 유발할 수 있습니다.

### 3-2. 🐌 데이터 순서 보장 불가

- **문제점:** 캐시에서 찾은 데이터(`listGoodsUnitVo.add(unitVo)`)를 먼저 넣고, 캐시에 없는 데이터는 나중에 DB에서 조회해서 추가(`listGoodsUnitVo.add(vo)`)하고 있습니다.

- **영향:** 요청 데이터의 순서와 결과 리스트의 데이터 순서가 달라지므로, 화면 정렬이나 정렬 조건이 깨질 수 있습니다.

### 3-3. 🛠️ 비효율적인 타입 변환 및 Null 체크

- **문제점:** `String.valueOf(vo.getGoodsSn())`을 반복 호출하고 있으며, 리스트 공백 체크 시 `size() == 0`을 사용하고 있습니다. `goodsListDB.size() > 0` 검증도 불필요한 중복 처리가 생길 수 있습니다.

- **영향:** 미미하지만 불필요한 객체 생성 및 가독성 저하를 야기합니다.

## 4. 개선된 로직 (Java Stream / Lambda 적용)

원본 파라미터를 훼손하지 않고, **요청 데이터의 순서를 그대로 유지**하며, **Stream API를 활용**해 선언적으로 작성한 개선 코드입니다.

```java
@Resource(name="ehCache") 
private final CacheService cacheService;
private static final String CACHE_KEY_ALIAS = "PRD_UNIT_";

private List<GoodsUnitVO> selectGoodsUnitListForCache(GoodsUnitParamVO goodsUnitParamVo) throws Exception {
    // 1. 방어 코드 (유효성 검증)
    if (goodsUnitParamVo == null || goodsUnitParamVo.getGoodsList() == null || goodsUnitParamVo.getGoodsList().isEmpty()) {
        return new ArrayList<>();
    }

    // 2. 캐시 조회 및 DB 타겟 분리
    List<GoodsUnitParamVO> targetGoodsList = goodsUnitParamVo.getGoodsList();
    
    // 캐시에서 가져온 데이터 Map (Key: GoodsSn, Value: GoodsUnitVO)
    Map<Long, GoodsUnitVO> cacheMap = targetGoodsList.stream()
            .filter(vo -> vo != null && vo.getGoodsSn() != null) // 안전한 Null 체크
            .map(vo -> (GoodsUnitVO) cacheService.get(CommonConstants.CACHE_ALIAS_GOODS_UNIT, CACHE_KEY_ALIAS + vo.getGoodsSn()))
            .filter(Objects::nonNull)
            .collect(Collectors.toMap(GoodsUnitVO::getGoodsSn, vo -> vo, (p1, p2) -> p1));

    // 3. 캐시에 없는 데이터만 모아서 DB 조회 대상 파라미터 생성 (원본 변조 방지)
    List<GoodsUnitParamVO> missingDbParams = targetGoodsList.stream()
            .filter(vo -> vo != null && vo.getGoodsSn() != null)
            .filter(vo -> !cacheMap.containsKey(vo.getGoodsSn()))
            .collect(Collectors.toList());

    // 4. DB 조회 및 캐시 적재
    if (!missingDbParams.isEmpty()) {
        // 원본 객체 보호를 위해 새로운 Param 객체 생성 (프로젝트 상황에 맞는 생성자나 빌더 사용 권장)
        GoodsUnitParamVO dbQueryParam = new GoodsUnitParamVO(); 
        dbQueryParam.setGoodsList(missingDbParams); 

        List<GoodsUnitVO> dbResultList = this.selectGoodsUnitListForDB(dbQueryParam);
        
        if (dbResultList != null) {
            dbResultList.stream()
                    .filter(vo -> vo != null && vo.getGoodsSn() != null)
                    .forEach(vo -> {
                        // 캐시 저장
                        cacheService.set(CommonConstants.CACHE_ALIAS_GOODS_UNIT, CACHE_KEY_ALIAS + vo.getGoodsSn(), vo);
                        // 다음 단계를 위해 Map에 병합
                        cacheMap.put(vo.getGoodsSn(), vo);
                    });
        }
    }

    // 5. [중요] 원본 요청 순서(targetGoodsList)대로 정렬하여 결과 반환
    return targetGoodsList.stream()
            .filter(vo -> vo != null && vo.getGoodsSn() != null)
            .map(vo -> cacheMap.get(vo.getGoodsSn()))
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
}
```

## 5. 주요 개선 포인트 요약

> 1. **부수 효과(Side Effect) 제거:** `dbQueryParam`이라는 별도의 조회용 파라미터 객체를 생성하여, 호출한 쪽의 원본 `goodsUnitParamVo` 데이터가 오염되는 것을 완벽히 방지했습니다.
>     
> 2. **순서 보장 (Order-Safe):** 캐시 데이터와 DB 데이터를 따로 누적하지 않고, 최종 단계(5번)에서 **최초 요청했던 리스트 순서 그대로** `Map`에서 꺼내어 정렬을 유지했습니다.
>     
> 3. **Stream API 활용:** `for`문과 `if-else` 계층 구조를 제거하고, `filter`, `map`, `collect`를 활용해 파이프라인 형태로 가독성을 높였습니다.
>     
> 4. **Null-Safe 처리:** `String.valueOf()` 대신 `vo.getGoodsSn() != null` 검증을 도입하여 예기치 못한 NPE(NullPointerException)나 `"null"` 문자열 키 생성을 방지했습니다. (만약 `goodsSn` 타입이 String이라면 그대로 사용하시고, Long/Integer라면 자동 Boxing되므로 문자열 결합(`+`) 시 자동으로 처리됩니다.)

## 6. 참고-1

### 6-1. 전제

첨부된 전체 Java 파일은 현재 대화에 실제 파일로는 확인되지 않아, 질문에 포함된 코드 기준으로 분석합니다. 현재 XML은 **Ehcache 3.x + JCache(JSR-107) + Spring Cache 추상화** 조합으로 보는 것이 맞습니다. Spring 5.3 문서에서도 Ehcache 3.x는 JSR-107 호환 캐시로 사용하며, 별도 Ehcache 전용 Spring 연동 없이 `JCacheCacheManager` 방식으로 사용할 수 있다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/integration.html "Integration"))

### 6-2. 현재 설정의 핵심 판단

|항목|판단|
|---|---|
|XML 설정|`JCacheCacheManager` 사용이므로 Ehcache 3.x 기준 구성|
|`cache:annotation-driven`|`@Cacheable`, `@CacheEvict`, `@CachePut` 같은 Spring Cache Annotation을 활성화하는 설정|
|현재 Java 코드|Annotation 방식이 아니라 `cacheService.get/set()`을 직접 호출하는 수동 Cache-Aside 방식|
|실무 적합성|단건 조회보다 “목록 중 캐시 미스만 DB 조회”하는 구조라 상품 목록 조회에는 적합|
|주요 문제|null 키 처리, 파라미터 변경 부작용, 타입 캐스팅 위험, 캐시 무효화 정책 부족|

Spring Cache는 캐시 구현체가 아니라 추상화 계층이므로 TTL, TTI, heap/offheap/disk, eviction 정책은 Spring XML이 아니라 Ehcache 설정에서 직접 제어해야 합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/integration.html "Integration"))

### 6-3. XML 설정 개선점

현재 설정 자체는 큰 방향은 맞지만, Bean 이름과 의도를 명확히 분리하는 편이 실무 유지보수에 좋습니다.

```xml
<cache:annotation-driven cache-manager="cacheManager" />

<bean id="cacheManager" class="org.springframework.cache.jcache.JCacheCacheManager">
    <property name="cacheManager" ref="jCacheManager" />
</bean>

<bean id="jCacheManager" class="org.springframework.cache.jcache.JCacheManagerFactoryBean">
    <property name="cacheManagerUri" value="classpath:egovframework/cache/ehcache-config.xml" />
</bean>
```

#### 6-3-1. 주의

`cache:annotation-driven`은 Annotation 기반 캐싱에 필요한 설정입니다. 현재 코드처럼 `cacheService.get/set()`을 직접 쓰는 메서드에는 AOP 캐싱이 적용되지 않습니다. 즉, 이 메서드 자체에는 `@Cacheable` 효과가 없습니다. Spring 문서 기준으로 Cache Annotation은 기능을 별도로 활성화해야 동작하며, 기본 proxy 모드에서는 외부에서 프록시를 통해 호출되는 public 메서드만 캐싱 인터셉트 대상이 됩니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/integration.html "Integration"))

### 6-4. Ehcache 설정 파일 체크 예시

`CommonConstants.CACHE_ALIAS_GOODS_UNIT` 값과 Ehcache XML의 `cache alias`가 반드시 같아야 합니다.

```xml
<config
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns="http://www.ehcache.org/v3"
    xmlns:jsr107="http://www.ehcache.org/v3/jsr107"
    xsi:schemaLocation="
        http://www.ehcache.org/v3 http://www.ehcache.org/schema/ehcache-core-3.0.xsd
        http://www.ehcache.org/v3/jsr107 http://www.ehcache.org/schema/ehcache-107-ext-3.0.xsd">

    <service>
        <jsr107:defaults enable-management="true" enable-statistics="true"/>
    </service>

    <cache alias="GOODS_UNIT">
        <key-type>java.lang.String</key-type>
        <value-type>bk.app.xxx.GoodsUnitVO</value-type>

        <expiry>
            <ttl unit="minutes">30</ttl>
        </expiry>

        <resources>
            <heap unit="entries">10000</heap>
        </resources>
    </cache>
</config>
```

Ehcache 3 XML에서 `<cache>`는 `alias`로 식별되며, `key-type`, `value-type`, `expiry`, `resources` 등을 지정할 수 있습니다. JCache용 management/statistics MBean도 XML에서 활성화할 수 있습니다. ([ehcache.org](https://www.ehcache.org/documentation/3.9/107.html "The Ehcache 3.x JSR-107 Provider"))

### 6-5. 현재 Java 코드의 주요 문제점

|구분|문제|개선 방향|
|---|---|---|
|키 검증|`String.valueOf(null)`은 `"null"` 문자열이 되어 캐시 키 `PRD_UNIT_null` 생성 가능|`goodsSn != null` 먼저 검사|
|파라미터 변경|`goodsUnitParamVo.setGoodsList(goodsListDB)`로 입력 객체를 변경|DB 조회용 파라미터를 별도 생성|
|타입 안정성|`(GoodsUnitVO) cacheData` 직접 캐스팅|`GoodsUnitVO.class.isInstance()` 검사 후 cast|
|중복 조회|같은 `goodsSn`이 여러 번 들어오면 DB 조회 대상에 중복 포함 가능|`LinkedHashMap`으로 중복 제거|
|결과 순서|기존 코드는 캐시 hit 먼저, DB 결과 나중에 추가되어 요청 순서와 달라질 수 있음|원 요청 순서 기준으로 결과 재구성|
|캐시 장애|`cacheService.get/set()` 장애 시 전체 조회 실패 가능|get 실패 시 DB fallback, set 실패 시 로그 후 진행|
|캐시 무효화|상품 단위 정보 변경 시 stale data 발생 가능|상품 수정/삭제/전시상태 변경 시 evict 필요|

### 6-6. Injection 개선

제공 코드의 아래 형태는 그대로라면 문제가 있습니다.

```java
@Resource(name = "ehCache")
private final CacheService cacheService;
```

`final` 필드는 생성자에서 초기화하는 방식이 안전합니다.

```java
private final CacheService cacheService;

@Autowired
public GoodsUnitService(@Qualifier("ehCache") CacheService cacheService) {
    this.cacheService = cacheService;
}
```

`@Resource` 필드 주입을 유지한다면 `final`을 제거해야 합니다.

```java
@Resource(name = "ehCache")
private CacheService cacheService;
```

실무에서는 생성자 주입이 테스트, 불변성, 필수 의존성 표현 측면에서 더 안전합니다.

### 6-7. Lambda/Stream 기반 개선 코드

아래 코드는 기존 의도인 **캐시에 있으면 캐시 사용, 없으면 미스 목록만 DB 조회 후 캐시에 저장**하는 방식을 유지하면서 개선한 예시입니다.

```java
private static final String CACHE_KEY_ALIAS = "PRD_UNIT_";
private static final String GOODS_UNIT_CACHE_ALIAS = CommonConstants.CACHE_ALIAS_GOODS_UNIT;

private List<GoodsUnitVO> selectGoodsUnitListForCache(GoodsUnitParamVO goodsUnitParamVo) throws Exception {
    if (goodsUnitParamVo == null || goodsUnitParamVo.getGoodsList() == null || goodsUnitParamVo.getGoodsList().isEmpty()) {
        return Collections.emptyList();
    }

    List<GoodsUnitParamVO> requestList = goodsUnitParamVo.getGoodsList();

    Map<String, GoodsUnitVO> resultMap = new LinkedHashMap<>();
    Map<String, GoodsUnitParamVO> missParamMap = new LinkedHashMap<>();

    requestList.stream()
        .filter(this::hasValidGoodsSn)
        .forEach(paramVo -> {
            String cacheKey = getGoodsUnitCacheKey(paramVo.getGoodsSn());

            Optional<GoodsUnitVO> cachedVo = getCacheValue(
                GOODS_UNIT_CACHE_ALIAS,
                cacheKey,
                GoodsUnitVO.class
            );

            if (cachedVo.isPresent()) {
                resultMap.putIfAbsent(cacheKey, cachedVo.get());
            } else {
                missParamMap.putIfAbsent(cacheKey, paramVo);
            }
        });

    if (!missParamMap.isEmpty()) {
        GoodsUnitParamVO dbParamVo = copyParamForDbSearch(
            goodsUnitParamVo,
            new ArrayList<>(missParamMap.values())
        );

        List<GoodsUnitVO> dbList = Optional
            .ofNullable(this.selectGoodsUnitListForDB(dbParamVo))
            .orElse(Collections.emptyList());

        dbList.stream()
            .filter(Objects::nonNull)
            .filter(vo -> vo.getGoodsSn() != null)
            .forEach(vo -> {
                String cacheKey = getGoodsUnitCacheKey(vo.getGoodsSn());

                putCacheValue(GOODS_UNIT_CACHE_ALIAS, cacheKey, vo);

                resultMap.putIfAbsent(cacheKey, vo);
            });
    }

    return requestList.stream()
        .filter(this::hasValidGoodsSn)
        .map(paramVo -> resultMap.get(getGoodsUnitCacheKey(paramVo.getGoodsSn())))
        .filter(Objects::nonNull)
        .collect(Collectors.toList());
}

private boolean hasValidGoodsSn(GoodsUnitParamVO vo) {
    if (vo == null || vo.getGoodsSn() == null) {
        return false;
    }

    return org.springframework.util.StringUtils.hasText(String.valueOf(vo.getGoodsSn()));
}

private String getGoodsUnitCacheKey(Object goodsSn) {
    return CACHE_KEY_ALIAS + String.valueOf(goodsSn);
}

private <T> Optional<T> getCacheValue(String cacheAlias, String cacheKey, Class<T> type) {
    try {
        Object cacheData = cacheService.get(cacheAlias, cacheKey);

        if (cacheData == null) {
            return Optional.empty();
        }

        if (!type.isInstance(cacheData)) {
            return Optional.empty();
        }

        return Optional.of(type.cast(cacheData));

    } catch (RuntimeException e) {
        // 캐시 조회 실패는 DB 조회로 fallback 처리
        // log.warn("Cache get failed. alias={}, key={}", cacheAlias, cacheKey, e);
        return Optional.empty();
    }
}

private void putCacheValue(String cacheAlias, String cacheKey, GoodsUnitVO value) {
    if (value == null) {
        return;
    }

    try {
        cacheService.set(cacheAlias, cacheKey, value);
    } catch (RuntimeException e) {
        // 캐시 저장 실패 때문에 정상 DB 조회 결과까지 실패시키지 않는 것이 일반적으로 안전
        // log.warn("Cache put failed. alias={}, key={}", cacheAlias, cacheKey, e);
    }
}

private GoodsUnitParamVO copyParamForDbSearch(GoodsUnitParamVO source, List<GoodsUnitParamVO> goodsListForDb) {
    GoodsUnitParamVO target = new GoodsUnitParamVO();

    /*
     * 중요:
     * source에 goodsList 외 조회 조건이 있다면 여기서 반드시 복사해야 함.
     * 예: siteNo, mallNo, langCd, channelCd, dispYn, saleStatus, session 조건 등
     */

    target.setGoodsList(goodsListForDb);
    return target;
}
```

### 6-8. 위 코드의 개선 효과

|항목|기존|개선|
|---|---|---|
|null goodsSn|`PRD_UNIT_null` 가능|null goodsSn 제외|
|중복 goodsSn|DB 중복 조회 가능|miss 대상 중복 제거|
|캐시 타입 오류|즉시 `ClassCastException` 가능|타입 확인 후 사용|
|입력 파라미터|원본 `goodsUnitParamVo` 변경|DB 조회용 객체 분리|
|결과 순서|cache hit + DB 결과 순서|요청 goodsList 순서 기준|
|캐시 장애|전체 로직 실패 가능|DB fallback 가능|
|가독성|캐시 처리 로직이 메서드 내부에 혼재|get/put/cacheKey 분리|

### 6-9. 실무 주의점

#### 6-9-1. 캐시 키에 `goodsSn`만 써도 되는지 검증

현재 키는 아래 구조입니다.

```java
PRD_UNIT_ + goodsSn
```

상품 단위 정보가 `goodsSn`만으로 완전히 결정되면 괜찮습니다. 하지만 결과가 아래 조건에 따라 달라지면 캐시 키에 반드시 포함해야 합니다.

```text
mallNo
siteNo
channelCd
langCd
memberGrade
saleStatus
dispYn
deviceType
pricePolicy
```

예시:

```java
private String getGoodsUnitCacheKey(GoodsUnitParamVO vo) {
    return CACHE_KEY_ALIAS
        + vo.getSiteNo() + ":"
        + vo.getChannelCd() + ":"
        + vo.getGoodsSn();
}
```

#### 6-9-2. 상품 수정 시 Evict 필수

상품 단위 정보가 변경되는데 캐시를 지우지 않으면 사용자는 변경 전 데이터를 계속 볼 수 있습니다.

```java
cacheService.remove(
    CommonConstants.CACHE_ALIAS_GOODS_UNIT,
    CACHE_KEY_ALIAS + goodsSn
);
```

상품 대량 변경, 배치 반영, 전시상태 변경, 판매상태 변경, 옵션/단위 변경 시점에는 캐시 삭제 또는 갱신 정책이 필요합니다. Spring Cache Annotation을 사용하는 경우에는 `@CacheEvict`를 사용할 수 있고, 전체 영역 삭제가 필요할 때는 `allEntries=true`가 가능하지만 전체 삭제는 부하가 크므로 신중히 써야 합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/integration.html "Integration"))

#### 6-9-3. 다중 WAS 환경에서는 로컬 캐시 불일치 가능

Ehcache를 각 WAS JVM의 로컬 캐시로 사용하면, A 서버에서 상품 정보 변경 후 캐시를 지워도 B 서버의 캐시는 그대로 남을 수 있습니다. 커머스 시스템에서는 이 문제가 자주 발생합니다.

실무 대안:

|방식|설명|
|---|---|
|짧은 TTL|데이터 불일치 시간을 제한|
|변경 이벤트 기반 evict|상품 변경 시 모든 WAS에 캐시 삭제 이벤트 전파|
|Redis 같은 중앙 캐시|여러 WAS가 같은 캐시 저장소 사용|
|캐시 대상 축소|변경 빈도가 낮은 기준성 데이터 위주로 캐시|

#### 6-9-4. VO 객체 변경 주의

캐시에 저장한 `GoodsUnitVO`가 mutable 객체이면, 조회한 쪽에서 값을 변경했을 때 캐시 객체 자체가 오염될 수 있습니다. 특히 heap 기반 local cache에서 동일 객체 참조가 공유되는 구성이면 위험합니다.

대안:

```text
1. GoodsUnitVO를 조회 후 수정하지 않는 규칙 적용
2. setter 사용 제한
3. DTO 복사본 반환
4. 캐시에 저장할 객체를 불변 객체로 설계
```

#### 6-9-5. 캐시 크기와 TTL을 반드시 운영 기준으로 설정

상품 단위 정보는 상품 수가 많아질수록 캐시 entry 수가 빠르게 증가합니다. `heap unit="entries"` 값을 너무 크게 잡으면 WAS heap 사용량이 증가하고, 너무 작게 잡으면 hit ratio가 낮아집니다. Ehcache XML에서는 heap/offheap/disk 같은 resource tier와 TTL/TTI 설정을 구성할 수 있습니다. ([ehcache.org](https://www.ehcache.org/documentation/3.11/xml.html "XML Configuration"))

### 6-10. `@Cacheable`로 바꾸는 방식은 신중해야 함

단건 조회 메서드라면 아래처럼 Annotation 기반으로 단순화할 수 있습니다.

```java
@Cacheable(
    cacheNames = CommonConstants.CACHE_ALIAS_GOODS_UNIT,
    key = "'PRD_UNIT_' + #goodsSn",
    unless = "#result == null"
)
public GoodsUnitVO selectGoodsUnitForDB(Long goodsSn) {
    return goodsUnitMapper.selectGoodsUnit(goodsSn);
}
```

하지만 현재 코드는 **목록 요청 중 캐시 미스만 모아서 DB를 한 번 조회**하는 구조입니다. 이 구조는 상품 목록, 전시 목록, 장바구니 상품 목록처럼 여러 상품을 한 번에 처리하는 업무에 더 적합합니다. 단순히 `@Cacheable` 단건 메서드로 바꾸면 캐시 미스가 여러 개일 때 DB 조회가 N번 발생할 수 있습니다.

### 6-11. 최종 권장안

| 우선순위 | 조치                                                                                   |
| ---: | ------------------------------------------------------------------------------------ |
|    1 | `String.valueOf(vo.getGoodsSn())` 방식 제거, null goodsSn 차단                             |
|    2 | `goodsUnitParamVo.setGoodsList(goodsListDB)`로 원본 파라미터 변경하지 않기                        |
|    3 | 캐시 get 결과 타입 확인 후 cast                                                               |
|    4 | 동일 goodsSn 중복 miss 제거                                                                |
|    5 | 요청 순서 기준으로 결과 재구성                                                                    |
|    6 | 상품 변경/삭제/상태변경 시 cache evict 정책 추가                                                    |
|    7 | Ehcache XML에 TTL, heap size, statistics 설정 명시                                        |
|    8 | 다중 WAS 환경이면 local cache 불일치 대응책 마련                                                   |
|    9 | 단건 조회는 `@Cacheable`, 목록 조회는 현재처럼 batch cache-aside 유지                                |
|   10 | Spring 5.3은 2024년 8월 31일 기준 OSS 지원이 종료된 계열이므로, 신규 개선 시 Spring 6.x 또는 상용 지원 여부도 같이 검토 |

Spring 5.x 계열은 2024년 8월 31일 OSS 지원 종료가 공지되어 있어, Spring 5.3을 계속 유지하는 시스템에서는 보안 패치와 라이브러리 호환성 리스크를 별도로 관리해야 합니다. ([github.com](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-5.3-Release-Notes "Spring Framework 5.3 Release Notes · spring-projects/spring-framework Wiki · GitHub"))

## 7. 참고-2

### 7-1. cache:annotation-driven

```text
<cache:annotation-driven>은 @Cacheable, @CachePut, @CacheEvict 같은 애노테이션 기반 캐싱을 Spring AOP로 동작시키는 설정이고, 현재 Java 코드는 그 애노테이션 경로를 사용하지 않고 CacheService.get/set을 직접 호출한다.
```

즉, 현재 설정과 코드는 **둘 다 Ehcache를 사용할 수는 있지만, 동작 경로가 다릅니다.**

### 7-2. 현재 XML 설정이 하는 일

```xml
<cache:annotation-driven cache-manager="ehCacheManager" />
<bean id="ehCacheManager" class="org.springframework.cache.jcache.JCacheCacheManager">
    <property name="cacheManager">
        <bean class="org.springframework.cache.jcache.JCacheManagerFactoryBean"
              p:cacheManagerUri="classpath:egovframework/cache/ehcache-config.xml" />
    </property>
</bean>
```

이 설정의 역할은 아래와 같습니다.

|구분|설명|
|---|---|
|`JCacheManagerFactoryBean`|`ehcache-config.xml`을 읽어 JCache `CacheManager` 생성|
|`JCacheCacheManager`|JCache `CacheManager`를 Spring Cache 추상화용 `CacheManager`로 감쌈|
|`<cache:annotation-driven>`|`@Cacheable`, `@CachePut`, `@CacheEvict`, `@Caching` 등을 AOP로 처리|
|`cache-manager="ehCacheManager"`|애노테이션 캐싱에서 사용할 Spring CacheManager 지정|

Spring 5.3 공식 문서 기준으로 캐시 애노테이션 처리의 기본 모드는 `proxy`이며, 이 방식은 Spring AOP 프록시를 통해 들어오는 메서드 호출만 인터셉트합니다. 또한 `<cache:annotation-driven/>`은 `@Cacheable`, `@CachePut`, `@CacheEvict`, `@Caching`이 붙은 Bean을 대상으로 동작합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/integration.html "Integration"))

### 7-3. 현재 Java 코드가 실제로 하는 일

현재 코드는 아래처럼 동작합니다.

```java
Object cacheData = cacheService.get(
    CommonConstants.CACHE_ALIAS_GOODS_UNIT,
    this.CACHE_KEY_ALIAS + String.valueOf(vo.getGoodsSn())
);
...
cacheService.set(
    CommonConstants.CACHE_ALIAS_GOODS_UNIT,
    this.CACHE_KEY_ALIAS + String.valueOf(vo.getGoodsSn()),
    vo
);
```

이 방식은 Spring Cache 애노테이션이 아니라 **직접 CacheService를 호출하는 수동 Cache-Aside 패턴**입니다.

```text
요청
→ Service 메서드 실행
→ cacheService.get()
→ 없으면 DB 조회
→ cacheService.set()
→ 결과 반환
```

반대로 `@Cacheable` 방식은 아래 흐름입니다.

```text
요청
→ Spring Proxy
→ CacheInterceptor
→ CacheManager
→ Cache Hit이면 메서드 실행 생략
→ Cache Miss이면 메서드 실행 후 결과 저장
```

따라서 현재 코드에는 `@Cacheable`이 없기 때문에 `<cache:annotation-driven>`이 이 메서드의 캐싱 동작에 직접 개입하지 않습니다.

### 7-4. “설정과 코드 사용 방식이 어긋난다”는 뜻

이 표현은 “오류”라는 뜻이 아니라, **활성화한 기능과 실제 코드가 사용하는 기능이 다르다**는 뜻입니다.

|항목|애노테이션 기반 캐싱|현재 코드|
|---|---|---|
|설정|`<cache:annotation-driven>` 필요|필수 아님|
|사용 방식|`@Cacheable`, `@CacheEvict`|`cacheService.get/set`|
|실행 주체|Spring AOP Proxy|개발자가 직접 호출|
|캐시 키|SpEL 또는 KeyGenerator|문자열 직접 생성|
|캐시 저장 시점|Spring Interceptor가 처리|코드에서 직접 처리|
|캐시 삭제|`@CacheEvict` 가능|`cacheService.remove` 등 직접 호출|
|Self-invocation 영향|있음|없음|
|private/protected 메서드 영향|있음|없음|
|캐시 예외 처리|Spring CacheErrorHandler 영향|CacheService 구현에 따름|

### 7-5. 실무에서 가장 위험한 착각

#### 7-5-1. `cache:annotation-driven`을 켰으니 모든 CacheService 호출이 Spring Cache 정책을 탄다고 오해

그렇지 않습니다.

```java
cacheService.get(alias, key);
cacheService.set(alias, key, value);
```

이 호출은 `@Cacheable` 경로가 아닙니다. 따라서 아래 항목은 적용되지 않습니다.

```text
@Cacheable
@CachePut
@CacheEvict
@Caching
KeyGenerator
CacheResolver
CacheErrorHandler
unless
condition
sync
```

#### 7-5-2. `@Cacheable`을 붙였는데 내부 호출이라 동작하지 않는 경우

Spring Cache의 기본 `proxy` 모드에서는 외부에서 Spring Bean 프록시를 통해 호출되는 public 메서드만 캐시 인터셉트 대상입니다. 같은 클래스 내부에서 `this.someCacheableMethod()`처럼 호출하면 캐싱이 적용되지 않습니다. Spring 공식 문서도 proxy 모드에서는 외부 메서드 호출만 인터셉트되며 self-invocation은 `@Cacheable`이 붙어도 실제 캐싱으로 이어지지 않는다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/integration.html "Integration"))  
예시:

```java
public List<GoodsUnitVO> selectGoodsUnitList(...) {
    return this.selectGoodsUnit(goodsSn); // 같은 클래스 내부 호출이면 @Cacheable 미적용 가능
}
@Cacheable(cacheNames = "GOODS_UNIT", key = "'PRD_UNIT_' + #goodsSn")
public GoodsUnitVO selectGoodsUnit(Long goodsSn) {
    return mapper.selectGoodsUnit(goodsSn);
}
```

이 경우 `selectGoodsUnit()`에 `@Cacheable`이 있어도 같은 클래스 내부 호출이면 캐시가 안 먹을 수 있습니다.

#### 7-5-3. Service가 아닌 Controller 쪽 Context에만 설정한 경우

Spring 공식 문서에 따르면 `<cache:annotation-driven/>`은 자신이 정의된 ApplicationContext 안의 Bean만 대상으로 봅니다. 예를 들어 DispatcherServlet의 WebApplicationContext에만 설정하면 Controller는 보지만 Service Bean은 보지 못할 수 있습니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/integration.html "Integration"))  
Spring XML 기반 프로젝트에서는 특히 아래 구조를 점검해야 합니다.

```text
root-context.xml
applicationContext.xml
dispatcher-servlet.xml
egov 설정 XML
```

`@Cacheable`을 Service에 붙일 계획이면 `<cache:annotation-driven>`은 Service Bean이 등록되는 Context에 있어야 합니다.

### 7-6. 현재 코드 기준으로 실제 점검해야 할 것

#### 7-6-1. `ehCacheManager`와 `ehCache`가 같은 캐시를 보는지 확인

현재 XML Bean 이름은 `ehCacheManager`입니다.

```xml
<bean id="ehCacheManager" class="org.springframework.cache.jcache.JCacheCacheManager">
```

그런데 Java에서는 아래 Bean을 주입합니다.

```java
@Resource(name = "ehCache")
private CacheService cacheService;
```

즉, 코드가 직접 쓰는 것은 `ehCacheManager`가 아니라 `ehCache`라는 이름의 `CacheService`입니다.  
반드시 확인해야 합니다.

```text
ehCache CacheService가 내부적으로 같은 ehcache-config.xml을 쓰는가?
ehCache CacheService가 JCacheCacheManager를 쓰는가?
아니면 별도 Ehcache 인스턴스를 생성하는가?
```

이게 다르면 실무에서 아래 문제가 생깁니다.

|문제|설명|
|---|---|
|캐시 이중화|`@Cacheable` 캐시와 `cacheService` 캐시가 서로 다른 저장소를 봄|
|Evict 불일치|`@CacheEvict`로 지웠는데 `cacheService.get()`은 여전히 값 조회|
|TTL 불일치|애노테이션 캐시와 수동 캐시의 만료 시간이 다름|
|모니터링 혼선|통계상 hit/miss가 실제 코드와 맞지 않음|

#### 7-6-2. `CommonConstants.CACHE_ALIAS_GOODS_UNIT`와 Ehcache alias 일치 확인

코드:

```java
CommonConstants.CACHE_ALIAS_GOODS_UNIT
```

Ehcache XML:

```xml
<cache alias="...">
```

이 둘이 정확히 같아야 합니다.  
예:

```java
public static final String CACHE_ALIAS_GOODS_UNIT = "GOODS_UNIT";
```

```xml
<cache alias="GOODS_UNIT">
```

점검 포인트:

```text
대소문자 일치
공백 없음
운영/개발 환경별 ehcache-config.xml 차이 없음
동일 alias가 중복 선언되지 않음
```

#### 7-6-3. 실제 `cache:annotation-driven`이 필요한지 판단

현재 코드가 전부 `cacheService.get/set` 방식이라면 `<cache:annotation-driven>`은 당장 이 메서드에는 필요 없습니다.  
다만 프로젝트 내 다른 곳에서 아래 애노테이션을 쓰고 있다면 필요합니다.

```java
@Cacheable
@CachePut
@CacheEvict
@Caching
```

점검 명령 예시:

```bash
grep -R "@Cacheable\|@CachePut\|@CacheEvict\|@Caching" ./src/main/java
```

Windows PowerShell:

```powershell
Get-ChildItem -Path .\src\main\java -Recurse -Filter *.java |
  Select-String -Pattern '@Cacheable|@CachePut|@CacheEvict|@Caching'
```

검증 포인트:

```text
대소문자 구분: PowerShell Select-String은 기본적으로 대소문자 구분 안 함
정확 검색 필요 시 -CaseSensitive 추가
Java 애노테이션은 보통 대소문자가 정해져 있으므로 위 패턴으로 충분
```

#### 7-6-4. 애노테이션 기반 캐싱을 쓸 경우 public 메서드인지 확인

Spring 공식 문서는 proxy 사용 시 cache annotation은 public 메서드에 적용하는 것을 권장하며, protected/private/package-visible 메서드에 붙여도 오류는 나지 않지만 캐싱 설정이 적용되지 않을 수 있다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/integration.html "Integration"))  
즉, 아래는 위험합니다.

```java
@Cacheable(cacheNames = "GOODS_UNIT", key = "#goodsSn")
private GoodsUnitVO selectGoodsUnit(Long goodsSn) {
    ...
}
```

아래처럼 public Service 메서드에 적용해야 합니다.

```java
@Cacheable(cacheNames = "GOODS_UNIT", key = "'PRD_UNIT_' + #goodsSn")
public GoodsUnitVO selectGoodsUnit(Long goodsSn) {
    ...
}
```

#### 7-6-5. 캐시 예외 처리 방식 확인

Spring Cache 애노테이션 기반에서는 기본적으로 캐시 처리 중 발생한 예외가 호출자에게 전파됩니다. Spring 5.3 문서의 cache annotation 설정 표에서도 기본 `error-handler`는 `SimpleCacheErrorHandler`이며, 기본적으로 cache 관련 operation에서 발생한 exception은 client로 throw된다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/integration.html "Integration"))  
수동 `CacheService` 방식에서는 구현에 따라 다릅니다.

```java
cacheService.get(...)
cacheService.set(...)
```

이 메서드가 예외를 던지면 상품 조회 자체가 실패할 수 있습니다.  
실무 권장:

```text
cache get 실패 → DB 조회로 fallback
cache set 실패 → 로그만 남기고 정상 결과 반환
cache remove 실패 → 중요도에 따라 재시도 또는 운영 알림
```

### 7-7. 현재 구조에서 선택 가능한 방향

#### 7-7-1. 방향 A. 현재처럼 수동 CacheService 유지

상품 목록 중 **캐시 미스만 모아서 DB 조회**해야 한다면 현재 방식이 더 실무적일 수 있습니다.

```text
장점:
- 여러 상품 중 miss만 모아 DB 1회 조회 가능
- 캐시 hit/miss 흐름을 직접 제어 가능
- 캐시 장애 시 fallback 처리하기 쉬움
단점:
- 키 생성, 타입 캐스팅, evict, 예외 처리를 개발자가 모두 관리해야 함
- Spring Cache의 condition/unless/keyGenerator/errorHandler를 자동 활용하기 어려움
```

이 경우 `<cache:annotation-driven>`은 해당 메서드에는 영향이 없으므로, 프로젝트 내 `@Cacheable` 사용 여부에 따라 유지 여부를 판단하면 됩니다.

#### 7-7-2. 방향 B. 단건 조회는 `@Cacheable`, 목록 조회는 수동 조합

가장 현실적인 절충안입니다.

```java
@Cacheable(
    cacheNames = CommonConstants.CACHE_ALIAS_GOODS_UNIT,
    key = "'PRD_UNIT_' + #goodsSn",
    unless = "#result == null"
)
public GoodsUnitVO selectGoodsUnitForCache(Long goodsSn) {
    return mapper.selectGoodsUnit(goodsSn);
}
```

다만 상품 목록 조회에서 이 메서드를 반복 호출하면 cache miss 상품 수만큼 DB가 여러 번 호출될 수 있습니다. 따라서 대량 목록 조회에서는 기존처럼 miss 목록을 모아 한 번에 DB 조회하는 방식이 더 유리할 수 있습니다.

#### 7-7-3. 방향 C. 전부 `@Cacheable`로 전환

단건 조회 중심 서비스라면 가능합니다.

```java
@Cacheable(cacheNames = "GOODS_UNIT", key = "'PRD_UNIT_' + #param.goodsSn")
public GoodsUnitVO selectGoodsUnit(GoodsUnitParamVO param) {
    return mapper.selectGoodsUnit(param);
}
```

하지만 현재 코드처럼 `goodsList`를 받아 여러 상품을 한 번에 조회하는 구조에는 부적합할 수 있습니다.

### 7-8. 실무 점검 체크리스트

#### 7-8-1. 설정 점검

```text
[ ] ehcache-config.xml이 실제 classpath에 존재하는가?
[ ] 운영 서버에 배포된 ehcache-config.xml이 최신인가?
[ ] CommonConstants.CACHE_ALIAS_GOODS_UNIT와 cache alias가 일치하는가?
[ ] JCacheCacheManager가 실제로 초기화되는가?
[ ] ehCache CacheService가 동일 CacheManager를 사용하는가?
[ ] cache:annotation-driven이 Service Bean이 있는 Context에 선언되어 있는가?
```

#### 7-8-2. 코드 점검

```text
[ ] @Cacheable 사용 메서드는 public인가?
[ ] 같은 클래스 내부 호출로 @Cacheable 메서드를 호출하지 않는가?
[ ] cacheService.get/set 예외 시 DB 조회가 실패하지 않도록 처리했는가?
[ ] goodsSn null일 때 PRD_UNIT_null 키가 생성되지 않는가?
[ ] 캐시 key에 site, mall, channel, language, grade 등 필요한 조건이 포함되는가?
[ ] 캐시 value 타입이 GoodsUnitVO와 일치하는가?
[ ] 캐시 저장 객체가 호출자에 의해 변경되지 않는가?
```

#### 7-8-3. 운영 점검

```text
[ ] 상품 수정/삭제/상태변경 시 캐시 evict가 수행되는가?
[ ] 다중 WAS 환경에서 다른 서버의 로컬 캐시도 무효화되는가?
[ ] TTL이 너무 길어 stale data가 발생하지 않는가?
[ ] 캐시 hit ratio를 확인할 수 있는가?
[ ] 캐시 entry 수와 WAS heap 사용량을 모니터링하는가?
[ ] 캐시 장애 시 전체 상품 조회 장애로 전파되지 않는가?
```

### 7-9. 현재 코드에서 특히 조심해야 할 부분

현재 방식은 아래 구조입니다.

```java
캐시 hit → listGoodsUnitVo.add(unitVo)
캐시 miss → goodsListDB.add(vo)
DB 조회 → listGoodsUnitVo.add(vo) 후 cacheService.set(...)
```

이 구조에서 중요한 것은 `cache:annotation-driven`이 아니라 아래입니다.

```text
1. cacheService가 어떤 CacheManager를 쓰는지
2. cache alias가 ehcache-config.xml과 맞는지
3. 상품 변경 시 remove/evict가 되는지
4. goodsSn 외 조건에 따라 결과가 달라지지 않는지
5. 캐시 장애가 DB 조회 장애로 번지지 않는지
```

### 7-10. 판단 기준

|상황|권장 방식|
|---|---|
|단건 조회가 많음|`@Cacheable` 사용 검토|
|목록 조회에서 일부만 캐시 miss|현재 같은 수동 Cache-Aside 방식 유지|
|상품 수정/삭제가 잦음|짧은 TTL + 명시적 evict 필수|
|다중 WAS 운영|로컬 Ehcache만으로는 불일치 가능성 관리 필요|
|캐시 장애가 서비스 장애가 되면 안 됨|수동 방식에서 try-catch fallback 권장|
|캐시 정책을 표준화하고 싶음|Spring Cache Annotation 기반으로 정리|

### 7-11. 결론

현재 Java 코드가 `cacheService.get/set()`을 직접 호출한다면, `<cache:annotation-driven cache-manager="ehCacheManager" />`은 해당 메서드의 캐싱 흐름에 직접 영향을 주지 않습니다. 이 설정은 `@Cacheable`, `@CachePut`, `@CacheEvict`, `@Caching` 같은 애노테이션을 Spring AOP Proxy로 처리하기 위한 설정입니다. 실무에서는 “애노테이션 기반 캐시”와 “수동 CacheService 기반 캐시”가 같은 Ehcache 설정과 같은 alias를 바라보는지 반드시 확인해야 하며, 두 방식이 섞이면 evict 불일치, TTL 불일치, hit/miss 통계 혼선, self-invocation 미적용 같은 문제가 발생할 수 있습니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
