### 핵심 결론

| 질문                                                                                                                                                                                                                                                                                                                                                                   | 답변                                                                                                                                      |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| GTM이 FrontEnd가 `dataLayer`에 저장한 값을 읽을 수 있는가?                                                                                                                                                                                                                                                                                                                         | 가능. GTM의 **Data Layer Variable**로 읽음                                                                                                    |
| GTM이 Meta 정보를 보강할 수 있는가?                                                                                                                                                                                                                                                                                                                                             | 가능. **Custom JavaScript Variable** 또는 **Custom HTML Tag**에서 DOM/Meta를 읽어 보강                                                             |
| GTM 처리가 끝났는지 FrontEnd가 자동으로 알 수 있는가?                                                                                                                                                                                                                                                                                                                                 | 자동으로는 알 수 없음. GTM이 명시적으로 `CustomEvent`, `callback`, `dataLayer.push()` 등으로 완료 신호를 보내야 함                                                 |
| 권장 방식                                                                                                                                                                                                                                                                                                                                                                | FrontEnd가 `dataLayer.push({event:'landing_context_ready', ...})` → GTM이 보강 → GTM이 `window.dispatchEvent()`로 완료 통지 → FrontEnd가 추천 API 호출 |
| GTM의 `dataLayer`는 Tag Manager와 태그에 정보를 전달하는 객체이며, `event` 값이 Push되면 해당 이벤트를 기준으로 트리거와 태그 실행이 가능하다. Google 공식 문서에서도 `dataLayer.push()`로 이벤트와 변수를 전달하고, 이벤트 이름을 Push한 뒤 Custom Event Trigger로 처리하는 방식을 안내한다. ([Google for Developers](https://developers.google.com/tag-platform/tag-manager/datalayer "The data layer  \|  Tag Platform  \|  Google for Developers")) |                                                                                                                                         |

### 1. 전체 처리 구조

```text
1. FrontEnd가 페이지 컨텍스트를 dataLayer에 Push
2. GTM이 landing_context_ready 이벤트를 감지
3. GTM이 Data Layer Variable로 FrontEnd 값을 읽음
4. GTM이 Meta Tag, Canonical URL, DOM 정보를 추가로 읽음
5. GTM이 보강된 Context 객체 생성
6. GTM이 FrontEnd에 처리 완료 신호 전달
7. FrontEnd가 보강된 Context로 BackEnd 추천 API 호출
8. FrontEnd가 추천 상품을 사용자에게 노출
```

### 2. 주체별 역할

|주체|역할|
|---|---|
|사용자|랜딩페이지 접속, 추천 상품 확인|
|브라우저|HTML 로딩, GTM 실행, JavaScript 실행, 화면 렌더링|
|FrontEnd|기본 페이지 컨텍스트 생성, `dataLayer.push()`, GTM 완료 신호 수신, 추천 API 호출, 추천 결과 렌더링|
|GTM|`dataLayer` 값 읽기, Meta/DOM 정보 보강, 완료 이벤트 발생|
|BackEnd|추천 API Proxy, 요청값 검증, 추천 결과 반환|
|DB|이 프로세스에서는 기본적으로 사용하지 않음|

### 3. FrontEnd에서 dataLayer에 기본 컨텍스트 저장

FrontEnd는 페이지 렌더링 시점에 가능한 구조화 데이터를 먼저 `dataLayer`에 저장합니다. GTM 공식 문서는 `window.dataLayer = window.dataLayer || []` 방식으로 초기화하고, 값을 추가할 때 `dataLayer.push()`를 사용하는 것을 권장한다. 또한 `dataLayer`를 덮어쓰면 GTM 이벤트/변수 처리가 깨질 수 있으므로 재할당하지 않아야 한다. ([Google for Developers](https://developers.google.com/tag-platform/tag-manager/datalayer "The data layer  |  Tag Platform  |  Google for Developers"))

```javascript
window.dataLayer = window.dataLayer || [];
window.dataLayer.push({
  event: 'landing_context_ready',
  pageType: 'LANDING',
  pageUrl: location.href,
  productId: window.__PAGE_CONTEXT__?.productId || '',
  productName: window.__PAGE_CONTEXT__?.productName || '',
  categoryId: window.__PAGE_CONTEXT__?.categoryId || '',
  categoryName: window.__PAGE_CONTEXT__?.categoryName || '',
  brandId: window.__PAGE_CONTEXT__?.brandId || '',
  price: window.__PAGE_CONTEXT__?.price || '',
  slotId: 'LANDING_RELATED_01'
});
```

### 4. GTM에서 dataLayer 값 읽는 방법

GTM 관리자 화면에서 아래와 같이 **Data Layer Variable**을 생성합니다.

|GTM 변수명|Variable Type|Data Layer Variable Name|
|---|---|---|
|`DLV - pageType`|Data Layer Variable|`pageType`|
|`DLV - pageUrl`|Data Layer Variable|`pageUrl`|
|`DLV - productId`|Data Layer Variable|`productId`|
|`DLV - productName`|Data Layer Variable|`productName`|
|`DLV - categoryId`|Data Layer Variable|`categoryId`|
|`DLV - categoryName`|Data Layer Variable|`categoryName`|
|`DLV - brandId`|Data Layer Variable|`brandId`|
|`DLV - price`|Data Layer Variable|`price`|
|`DLV - slotId`|Data Layer Variable|`slotId`|
|GTM은 `dataLayer`에 Push된 변수와 이벤트를 기준으로 태그를 실행할 수 있으며, 이벤트 Push는 `dataLayer.push({'event':'event_name'})` 형태로 처리한다. ([Google for Developers](https://developers.google.com/tag-platform/tag-manager/datalayer "The data layer  \|  Tag Platform  \|  Google for Developers"))|||

### 5. GTM Trigger 설정

|항목|설정값|
|---|---|
|Trigger Type|Custom Event|
|Event name|`landing_context_ready`|
|This trigger fires on|All Custom Events 또는 조건부|
|이 Trigger는 FrontEnd가 `event: 'landing_context_ready'`를 Push했을 때 실행됩니다.||

### 6. GTM에서 Meta 데이터 보강

#### 6.1 Custom JavaScript Variable 방식

GTM에서 아래와 같은 **Custom JavaScript Variable**을 생성합니다.

##### `JS - ogTitle`

```javascript
function() {
  var meta = document.querySelector('meta[property="og:title"]');
  return meta && meta.content ? meta.content : document.title;
}
```

##### `JS - metaDescription`

```javascript
function() {
  var meta = document.querySelector('meta[name="description"]');
  return meta && meta.content ? meta.content : '';
}
```

##### `JS - ogDescription`

```javascript
function() {
  var meta = document.querySelector('meta[property="og:description"]');
  return meta && meta.content ? meta.content : '';
}
```

##### `JS - canonicalUrl`

```javascript
function() {
  var link = document.querySelector('link[rel="canonical"]');
  return link && link.href ? link.href : location.href;
}
```

### 7. GTM에서 보강된 Context 생성 후 FrontEnd에 완료 통지

GTM의 **Custom HTML Tag**를 사용하면 직접 JavaScript를 실행할 수 있습니다. Google 공식 도움말에서도 Custom HTML Tag에 직접 HTML 또는 JavaScript 코드를 넣을 수 있고, JavaScript는 `<script></script>` 안에 넣어야 한다고 안내한다. ([구글 도움말](https://support.google.com/tagmanager/answer/6107167?hl=en "Custom tags - Tag Manager Help"))

#### GTM Custom HTML Tag 예시

Trigger: `landing_context_ready`

```html
<script>
(function() {
  var enrichedContext = {
    basisType: 'LANDING_CONTEXT',
    pageType: '{{DLV - pageType}}',
    pageUrl: '{{DLV - pageUrl}}',
    canonicalUrl: '{{JS - canonicalUrl}}',
    metaTitle: '{{JS - ogTitle}}',
    metaDescription: '{{JS - metaDescription}}',
    ogDescription: '{{JS - ogDescription}}',
    productId: '{{DLV - productId}}',
    productName: '{{DLV - productName}}',
    categoryId: '{{DLV - categoryId}}',
    categoryName: '{{DLV - categoryName}}',
    brandId: '{{DLV - brandId}}',
    price: '{{DLV - price}}',
    slotId: '{{DLV - slotId}}'
  };
  window.__LANDING_RECOMMEND_CONTEXT__ = enrichedContext;
  window.dataLayer = window.dataLayer || [];
  window.dataLayer.push({
    event: 'landing_context_enriched',
    landingRecommendContext: enrichedContext
  });
  window.dispatchEvent(new CustomEvent('gtm:landingContextEnriched', {
    detail: enrichedContext
  }));
})();
</script>
```

### 8. FrontEnd에서 GTM 처리 완료를 아는 방법

GTM 처리가 끝났다는 사실은 FrontEnd가 자동으로 알 수 없습니다. GTM에서 아래 중 하나의 방식으로 **명시적 완료 신호**를 보내야 합니다.

|방식|권장도|설명|
|---|--:|---|
|`window.dispatchEvent(new CustomEvent(...))`|상|FrontEnd가 가장 명확하게 수신 가능|
|GTM에서 FrontEnd callback 함수 호출|중|`window.onLandingContextEnriched(context)` 형태|
|GTM에서 `dataLayer.push({event:'landing_context_enriched'})`|중|GTM 내부 후속 태그에는 좋지만 FrontEnd가 직접 감지하려면 별도 감시 필요|
|FrontEnd Polling|하|`window.__LANDING_RECOMMEND_CONTEXT__`를 일정 시간 확인|
|`eventCallback` 의존|하|태그 실행 완료와 비동기 로직 완료를 혼동할 수 있어 추천 프로세스 완료 신호로는 비권장|

### 9. 권장 방식: CustomEvent 수신

#### FrontEnd 코드

중요한 점은 **이벤트 리스너를 먼저 등록한 후** `dataLayer.push()`를 해야 합니다. 그렇지 않으면 GTM이 매우 빨리 실행될 경우 완료 이벤트를 놓칠 수 있습니다.

```javascript
function waitForGtmLandingContext(timeoutMs) {
  return new Promise(function(resolve) {
    var done = false;
    var timer = setTimeout(function() {
      if (done) return;
      done = true;
      resolve(null);
    }, timeoutMs || 1500);
    window.addEventListener('gtm:landingContextEnriched', function(event) {
      if (done) return;
      done = true;
      clearTimeout(timer);
      resolve(event.detail);
    }, { once: true });
  });
}
async function initLandingRecommend() {
  var waitPromise = waitForGtmLandingContext(1500);
  window.dataLayer = window.dataLayer || [];
  window.dataLayer.push({
    event: 'landing_context_ready',
    pageType: 'LANDING',
    pageUrl: location.href,
    productId: window.__PAGE_CONTEXT__?.productId || '',
    productName: window.__PAGE_CONTEXT__?.productName || '',
    categoryId: window.__PAGE_CONTEXT__?.categoryId || '',
    categoryName: window.__PAGE_CONTEXT__?.categoryName || '',
    brandId: window.__PAGE_CONTEXT__?.brandId || '',
    price: window.__PAGE_CONTEXT__?.price || '',
    slotId: 'LANDING_RELATED_01'
  });
  var enrichedContext = await waitPromise;
  if (!enrichedContext) {
    enrichedContext = buildFallbackContextByFrontEnd();
  }
  requestRelatedRecommend(enrichedContext);
}
function buildFallbackContextByFrontEnd() {
  return {
    basisType: 'LANDING_CONTEXT',
    pageType: 'LANDING',
    pageUrl: location.href,
    canonicalUrl: document.querySelector('link[rel="canonical"]')?.href || location.href,
    metaTitle: document.querySelector('meta[property="og:title"]')?.content || document.title,
    metaDescription: document.querySelector('meta[name="description"]')?.content || '',
    productId: window.__PAGE_CONTEXT__?.productId || '',
    productName: window.__PAGE_CONTEXT__?.productName || '',
    categoryId: window.__PAGE_CONTEXT__?.categoryId || '',
    categoryName: window.__PAGE_CONTEXT__?.categoryName || '',
    brandId: window.__PAGE_CONTEXT__?.brandId || '',
    price: window.__PAGE_CONTEXT__?.price || '',
    slotId: 'LANDING_RELATED_01'
  };
}
function requestRelatedRecommend(context) {
  fetch('/api/recommend/context-related', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(context)
  })
  .then(function(response) {
    return response.json();
  })
  .then(function(result) {
    renderRecommendProducts(result.items || []);
  })
  .catch(function() {
    renderFallbackRecommendProducts();
  });
}
```

### 10. 대안: callback 함수 방식

FrontEnd가 callback 함수를 먼저 등록하고, GTM Custom HTML에서 이 함수를 호출하는 방식입니다.

#### FrontEnd

```javascript
window.onLandingContextEnriched = function(context) {
  requestRelatedRecommend(context);
};
window.dataLayer = window.dataLayer || [];
window.dataLayer.push({
  event: 'landing_context_ready',
  pageType: 'LANDING',
  pageUrl: location.href
});
```

#### GTM Custom HTML

```html
<script>
(function() {
  var enrichedContext = {
    pageUrl: '{{DLV - pageUrl}}',
    metaTitle: '{{JS - ogTitle}}',
    metaDescription: '{{JS - metaDescription}}',
    canonicalUrl: '{{JS - canonicalUrl}}'
  };
  if (typeof window.onLandingContextEnriched === 'function') {
    window.onLandingContextEnriched(enrichedContext);
  }
})();
</script>
```

이 방식은 단순하지만 GTM과 FrontEnd가 함수명으로 강하게 결합됩니다. 운영 안정성은 `CustomEvent` 방식이 더 좋습니다.

### 11. 실무 권장 프로세스

```text
1. FrontEnd가 이벤트 리스너 등록
2. FrontEnd가 기본 pageContext를 dataLayer에 Push
3. GTM Custom Event Trigger가 landing_context_ready 감지
4. GTM Data Layer Variable로 productId, categoryId, pageUrl 읽음
5. GTM Custom JavaScript Variable로 metaTitle, metaDescription, canonicalUrl 읽음
6. GTM Custom HTML Tag가 enrichedContext 생성
7. GTM이 window.dispatchEvent()로 완료 이벤트 발생
8. FrontEnd가 완료 이벤트 수신
9. FrontEnd가 enrichedContext로 BackEnd 추천 API 호출
10. 추천 결과를 사용자 화면에 렌더링
```

### 12. 기술적인 주의점

|구분|주의점|대책|
|---|---|---|
|이벤트 타이밍|FrontEnd가 수신 리스너 등록 전에 GTM이 완료 이벤트를 쏘면 놓칠 수 있음|리스너 등록 후 `dataLayer.push()`|
|GTM 차단|광고 차단, CSP, 네트워크 문제로 GTM이 실행되지 않을 수 있음|FrontEnd fallback context 생성|
|DOM 의존|GTM이 DOM에서 상품명/카테고리를 읽으면 HTML 변경에 취약|가능하면 FrontEnd가 구조화된 값을 `dataLayer`에 Push|
|Meta 신뢰도|Meta는 SEO/마케팅 문구라 실제 상품 정보와 다를 수 있음|`productId`, `categoryId`, `productName` 우선|
|비동기 처리|Custom HTML 안에서 외부 API 호출까지 하면 완료 시점 관리가 복잡|GTM은 보강까지만, 추천 API 호출은 FrontEnd 또는 BackEnd|
|보안|FrontEnd/GTM 값은 사용자 조작 가능|BackEnd에서 길이, 형식, 허용 도메인 검증|
|개인정보|`pageUrl`, `referrer`, query string에 개인정보가 포함될 수 있음|QueryString 제거 또는 허용 파라미터만 전송|
|dataLayer 재할당|`window.dataLayer = []`로 덮어쓰면 GTM 처리 깨질 수 있음|항상 `window.dataLayer = window.dataLayer|

### 최종 권장안

| 항목                | 권장                                                                    |
| ----------------- | --------------------------------------------------------------------- |
| FrontEnd → GTM 전달 | `dataLayer.push({event:'landing_context_ready', ...})`                |
| GTM 값 읽기          | Data Layer Variable + Custom JavaScript Variable                      |
| GTM 보강 대상         | `og:title`, `meta description`, `canonicalUrl`, 보조 DOM 정보             |
| GTM 완료 통지         | `window.dispatchEvent(new CustomEvent('gtm:landingContextEnriched'))` |
| 추천 API 호출         | FrontEnd가 BackEnd API 호출                                              |
| GTM 실패 대응         | FrontEnd fallback context로 추천 호출                                      |
| DB 사용             | 기본 불필요                                                                |