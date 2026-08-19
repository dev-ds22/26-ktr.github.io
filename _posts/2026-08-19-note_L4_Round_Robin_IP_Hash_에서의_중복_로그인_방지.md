---
layout: single
title: "L4_Round_Robin_IP_Hash_에서의_중복_로그인_방지"
excerpt: "L4_Round_Robin_IP_Hash_에서의_중복_로그인_방지"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-08-19"
last_modified_at: "2026-08-19 16:28:34 +0900"
mermaid: false
---
### 1. 결론

**중복로그인 방지 로직은 L4의 Round-Robin/IP Hash 방식에 의존해서 설계하면 안 됩니다.**
두 방식의 차이는 주로 **“같은 사용자의 요청이 같은 WAS로 계속 들어올 가능성”**에 있고, 중복로그인 여부를 판단하는 **공유 로그인 상태 저장소가 필요한가**라는 본질은 동일합니다.
다만 구현 난이도는 크게 다릅니다.

| 구분                                                                                 | Round-Robin                | IP Hash                           |
| ---------------------------------------------------------------------------------- | -------------------------- | --------------------------------- |
| 요청 분배                                                                              | 연결/요청을 WAS들에 순차 분산         | Source IP 기준 특정 WAS 선택            |
| 동일 사용자 WAS 고정성                                                                     | 낮음                         | 상대적으로 높음                          |
| Local SessionRegistry만 사용                                                          | **불가능에 가까움**               | 동작하는 것처럼 보이나 **안전하지 않음**          |
| Session Cluster 필요성                                                                | 매우 높음                      | 장애/Failover 고려하면 필요               |
| 공유 LoginToken 저장소                                                                  | **필수**                     | **필수 권장**                         |
| 중복로그인 구현 차이                                                                        | 매 요청 어느 WAS에서도 검증 가능해야 함   | 같은 WAS로 오는 동안 Local cache 활용도가 높음 |
| 권장 설계                                                                              | DB/Redis/JMS + Local Cache | 동일                                |
| 즉 현재 시스템에서는 **L4 방식과 무관하게 `USER_ID → LOGIN_TOKEN`을 클러스터 전체가 공유하도록 구현하는 것이 정답**입니다. |                            |                                   |

---

### 2. Round-Robin에서는 왜 문제가 명확한가

WAS가 2대라고 가정합니다.

```text
               L4
          Round-Robin
             /    \
            /      \
         WAS1      WAS2
```

사용자 A가 로그인합니다.

```text
Request #1 로그인
      ↓
     WAS1
Session TOKEN=A
```

다음 요청이:

```text
Request #2
      ↓
     WAS2
```

로 갈 수 있습니다.
물론 실제 L4에서는 TCP connection, keep-alive, persistence 설정에 따라 **매 HTTP Request마다 정확하게 WAS가 바뀐다고 단정할 수는 없습니다.**
하지만 핵심은:

> Round-Robin 환경에서는 동일 사용자의 후속 연결이 다른 WAS로 갈 수 있다는 점입니다.
> 따라서 WAS1 내부에만:

```java
ConcurrentHashMap<String, String> loginUsers;
```

를 두고:

```text
WAS1
USER01 → TOKEN-A

WAS2
없음
```

이라면 중복로그인을 제대로 판단할 수 없습니다.

### 3. Round-Robin에서 Local SessionRegistry만 사용하면 발생하는 문제

Spring Security의 기본적인:

```java
SessionRegistryImpl
```

같은 JVM Local Registry를 생각하면 됩니다.
WAS1:

```text
USER01
SESSION-A
```

를 가지고 있습니다.
그런데 USER01이 다시 로그인하여 L4가 WAS2로 보냅니다.

```text
                L4
                 │
        ┌────────┴────────┐
        ▼                 ▼
      WAS1              WAS2

USER01=SESSION-A      USER01 로그인
                     SESSION-B
```

WAS2에서는:

```text
USER01 로그인 기록 없음
```

으로 보입니다.
따라서:

```text
중복로그인 아님
```

이라고 판단하게 됩니다.
결과:

```text
WAS1 → SESSION-A 정상
WAS2 → SESSION-B 정상
```

- 두 세션이 동시에 살아 있게 됩니다. **Round-Robin에서는 Local Registry 방식의 한계가 바로 드러납니다.**
---------------------------------------------------

## 4. IP Hash에서는 왜 동작하는 것처럼 보이는가

IP Hash라면:

```text
hash(Source IP) → WAS
```

형태로 서버를 선택합니다.
예:

```text
211.100.10.1
     │
     ▼
    L4
     │
 hash(IP)
     │
     ▼
    WAS1
```

따라서 같은 PC에서 요청하면 상당 기간:

```text
USER01
 ↓
WAS1
 ↓
WAS1
 ↓
WAS1
```

으로 가기 때문에 WAS1 Local Registry만 사용해도:

```text
USER01 → SESSION-A
```

를 계속 찾을 수 있습니다.
그래서 테스트 환경에서는:

> "IP Hash면 WAS Local SessionRegistry만으로 중복로그인이 되는 것 아닌가?"
> 라는 착각이 생길 수 있습니다.
> 하지만 운영환경에서는 안전하지 않습니다.

---

## 5. IP Hash에서도 Local Registry만 사용하면 안 되는 이유

### 5-1. Case 1. 다른 네트워크에서 재로그인

첫 로그인:

```text
회사 PC
IP = 100.10.10.1

        L4
         │
         ▼
        WAS1

USER01 → TOKEN-A
```

같은 USER01이 집에서 로그인:

```text
집 PC
IP = 200.20.20.2

        L4
         │
         ▼
        WAS2

USER01 → TOKEN-B
```

상태:

```text
WAS1                      WAS2

USER01                    USER01
TOKEN-A                   TOKEN-B

정상이라고 판단            정상이라고 판단
```

따라서 중복로그인이 그대로 발생합니다.

## 6. 모바일 환경에서는 IP Hash가 더욱 불안정

모바일 사용자의 Source IP는 변할 수 있습니다.
예:

```text
Wi-Fi
192.x → 공인IP A
       ↓
      WAS1
```

사용자가 Wi-Fi를 끄고 LTE/5G로 전환:

```text
LTE
공인IP B
   ↓
  L4
   ↓
 WAS2
```

즉 동일 사용자라도:

```text
IP 변경
 ↓
Hash 변경
 ↓
WAS 변경
```

- 될 수 있습니다. VPN 사용도 마찬가지입니다.
----------------

## 7. NAT 때문에 반대 문제도 발생

기업이나 기관에서는 수백 명이 같은 NAT Public IP를 사용할 수 있습니다.

```text
PC1 ─┐
PC2 ─┤
PC3 ─┤ NAT
PC4 ─┤
PC5 ─┘
       │
       │ 203.10.10.1
       ▼
       L4
       │
     IP Hash
       │
       ▼
      WAS1
```

그러면 서로 다른 사용자들이 모두 WAS1으로 집중될 수 있습니다.
이는 중복로그인 판단 문제와 직접적인 동일 개념은 아니지만:

```text
IP Hash
=
User 단위 Sticky
```

가 아니라는 사실이 중요합니다.
실제로는:

```text
IP Hash
=
Source IP 단위 Sticky
```

## 8. WAS 장애 시 IP Hash도 깨진다

정상:

```text
USER01
IP-A
 ↓
L4
 ↓
WAS1
```

WAS1이 장애 나면:

```text
WAS1 DOWN
    ↓
L4 rehash / failover
    ↓
WAS2
```

로 이동합니다.
그런데 중복로그인 정보가:

```text
WAS1 Local Memory
USER01 → TOKEN-A
```

- 에만 있었다면 WAS2는 이를 모릅니다.따라서 HA 환경에서 Local Registry를 중복로그인의 authoritative state로 사용해서는 안 됩니다.
--------------------------------------------------------------------

## 9. 따라서 Round-Robin은 이렇게 구현

현재 권장한 구조라면:

```text
                       L4
                  Round-Robin
                  /         \
                 /           \
              WAS1           WAS2
                │              │
       Local Token Cache Local Token Cache
                │              │
                └──────┬───────┘
                       │
                       ▼
                    MariaDB
              USER_ID → TOKEN
```

로그인:

```text
USER01
 ↓
WAS1

TOKEN-A 발행
 ↓
DB
USER01=TOKEN-A
```

두 번째 로그인:

```text
USER01
 ↓
WAS2

TOKEN-B 발행
 ↓
DB UPDATE
USER01=TOKEN-B
```

이제 기존 SESSION-A 요청이 어느 WAS로 들어오든:

```text
Session TOKEN = TOKEN-A
Current TOKEN = TOKEN-B
```

이므로:

```text
A != B
→ 중복로그인 감지
→ 기존 로그인 종료
```

- 됩니다. 이것이 **LB-independent한 중복로그인 설계**입니다.
------------------------------------

## 10. IP Hash도 동일한 구조를 사용

IP Hash라고 해서 구조를 바꾸지 않는 것이 중요합니다.

```text
                      L4
                    IP Hash
                  /         \
                WAS1        WAS2
                  │           │
              Local Cache Local Cache
                  │           │
                  └─────┬─────┘
                        ▼
                      DB
               USER_ID → TOKEN
```

다만 성능 관점에서는 이점이 있습니다.
IP Hash에서는 같은 사용자가 동일 WAS로 들어올 확률이 높으므로:

```text
Request
 ↓
WAS1
 ↓
Local Cache HIT
```

비율이 높아질 수 있습니다.
하지만 이것은:

> **성능 최적화 요소**
> 이지
> **중복로그인 정확성의 전제조건**
> 이어서는 안 됩니다.

---

## 11. 현재 구조에서 가장 중요한 차이

Round-Robin:

```text
USER01 요청
   │
   ├─ WAS1
   ├─ WAS2
   ├─ WAS1
   └─ WAS2
```

라고 생각하고 설계해야 하므로:

```text
어느 WAS에서도
USER01의 현재 LOGIN_TOKEN을 알아야 함
```

이 필수입니다.
IP Hash:

```text
USER01
   │
   └──────── WAS1
```

로 대부분 동작하기 때문에:

```text
Local Cache HIT
```

가 더 잘 나옵니다.
하지만 장애, IP 변경, 다른 장치 로그인 시:

```text
USER01 → WAS2
```

가 언제든 발생할 수 있으므로 결국 공유 저장소가 필요합니다.

## 12. `<distributable/>`이 있다면 Round-Robin에서도 HttpSession은 유지 가능

현재 JBoss에서:

```xml
<distributable/>
```

을 사용하고 있다면 중요한 의미가 있습니다.
예를 들어:

```text
Request 1
 ↓
WAS1
SESSION ID = ABC
```

다음 요청:

```text
Request 2
 ↓
WAS2
SESSION ID = ABC
```

가 되더라도 JBoss의 clustered web session을 통해 WAS2가 해당 세션 상태를 사용할 수 있도록 구성할 수 있습니다.
그러므로:

```text
L4 Round-Robin
+
JBoss Session Cluster
```

조합 자체는 정상적인 HA 구조입니다.
하지만 여기서:

```text
HttpSession 복제
```

와

```text
USER_ID 기준 중복 로그인 관리
```

는 다른 문제입니다.
다시 강조하면:

```text
Session Cluster
        ≠
Concurrent Login Registry
```

입니다.

## 13. Round-Robin에서는 오히려 Session Cluster와 Token Registry 역할 분리가 중요

권장 구조:

```text
┌───────────────────────────────────────────┐
│                   L4                      │
│              Round-Robin                  │
└───────────────────┬───────────────────────┘
                    │
           ┌────────┴────────┐
           ▼                 ▼
         WAS1              WAS2
           │                 │
           │<───────────────>│
           │ JBoss Session   │
           │   Cluster       │
           │                 │
           └────────┬────────┘
                    │
                    ▼
                Login Registry
                   DB
                    │
             USER01=TOKEN-B
```

역할:

| 구성요소                       | 책임                       |
| -------------------------- | ------------------------ |
| L4                         | Traffic distribution     |
| JBoss Infinispan web cache | HttpSession HA           |
| HttpSession                | 현재 Browser/Login Context |
| Login Token Registry       | 사용자별 현재 유효 로그인           |
| Local Cache                | Token validation 성능 향상   |
| JMS                        | Token 변경 즉시 전파(선택)       |
| 이렇게 책임을 분리해야 합니다.          |                          |

---

## 14. 가장 위험한 구현 두 가지

### 14-1. 잘못된 설계 ① IP Hash이므로 Local Map 사용

```java
private static final ConcurrentHashMap<String, String>
        LOGIN_USERS = new ConcurrentHashMap<>();
```

```text
IP Hash라서 항상 같은 WAS
→ Local Map이면 충분
```

은 잘못된 가정입니다.
다른 단말 로그인:

```text
PC → WAS1
Mobile → WAS2
```

이면 바로 깨집니다.

### 14-2. 잘못된 설계 ② Session ID를 기준으로 DB 관리

예:

```text
USER01 → JSESSIONID=ABC123
```

만 DB에 저장하는 것도 권장하지 않습니다.
JBoss clustered session/failover 및 session ID 변경 정책과 로그인 정책을 지나치게 결합시키기 때문입니다.
보다 명확하게:

```text
USER_ID
    ↓
LOGIN_TOKEN(UUID)
```

을 별도로 사용하는 것이 좋습니다.

## 15. 새 로그인 시 DB 동시성도 처리해야 함

L4 방식과 별개로 중요한 문제가 하나 더 있습니다.
같은 사용자가 거의 동시에 두 곳에서 로그인하면:

```text
PC
  \
   → WAS1 → TOKEN-A
  /
Mobile

Mobile
  \
   → WAS2 → TOKEN-B
```

처럼 race condition이 발생할 수 있습니다.
따라서 DB는:

```text
USER_ID = PK/UNIQUE
```

가 되어야 합니다.
예:

```sql
CREATE TABLE TB_LOGIN_CONTROL (
    USER_ID       VARCHAR(50) NOT NULL,
    LOGIN_TOKEN   VARCHAR(64) NOT NULL,
    LOGIN_DT      DATETIME NOT NULL,
    PRIMARY KEY (USER_ID)
);
```

MariaDB에서는 로그인 성공 시 atomic UPSERT:

```sql
INSERT INTO TB_LOGIN_CONTROL (
    USER_ID,
    LOGIN_TOKEN,
    LOGIN_DT
)
VALUES (
    #{userId},
    #{loginToken},
    NOW()
)
ON DUPLICATE KEY UPDATE
    LOGIN_TOKEN = VALUES(LOGIN_TOKEN),
    LOGIN_DT = NOW();
```

형태가 적절합니다.
최종 DB의:

```text
USER01 → TOKEN-B
```

가 **현재 유효한 로그인 하나**를 결정합니다.

## 16. Round-Robin일수록 Local Cache 설계를 조금 주의해야 함

앞서 추천했던:

```text
Local Cache TTL = 5초
```

방식에서 Round-Robin이면:

```text
WAS1
USER01 cache

WAS2
USER01 cache
```

가 각각 존재합니다.
그러므로 동일 사용자가 두 WAS를 왕복하면 두 서버가 각각 DB를 조회할 수 있습니다.
예:

```text
WAS 2대
TTL = 5 sec
```

이면 한 사용자가 활발하게 사용한다고 할 때 이론적으로:

```text
WAS1 → 5초마다 DB
WAS2 → 5초마다 DB
```

가 될 수 있습니다.
즉 DB 검증량은 대략:

```text
Active User × WAS 수 / TTL
```

성격으로 증가할 수 있습니다.
IP Hash에서는 대부분:

```text
Active User / TTL
```

- 에 가까워집니다.  따라서 **Local TTL Cache만 놓고 보면 IP Hash 쪽이 DB 검증 효율이 더 좋습니다.**
-----------------------------------------------------------

## 17. 이 문제는 JMS를 넣으면 대부분 사라진다

그래서 다중 WAS + Round-Robin 환경에서는:

```text
DB + Local Cache + JMS
```

가 특히 좋은 구조입니다.
로그인:

```text
WAS2
TOKEN-B 발급
  │
  ├─ DB UPDATE
  │
  └─ JMS
       │
       ├─ WAS1 Cache USER01=B
       └─ WAS2 Cache USER01=B
```

이후 어느 WAS로 요청하더라도:

```text
Session=A
Cache=B
```

- 이므로 즉시 중복로그인을 감지합니다. 그때 DB 조회가 필요하지 않습니다.
--------------------

## 18. 성능 기준으로 비교하면

| 구조                     | Round-Robin |              IP Hash |
| ---------------------- | ----------: | -------------------: |
| 매 요청 DB Token 조회       |    ❌ 매우 비효율 |                ❌ 비효율 |
| Local Map만 사용          |    ❌ 정확성 문제 |             ❌ 운영상 위험 |
| DB + 5초 Local Cache    |           ◎ | ◎ **Cache 효율은 더 좋음** |
| DB + Local Cache + JMS |      **◎◎** |               **◎◎** |
| JBoss Session Cluster  |   권장/사실상 필요 |     Failover 고려 시 권장 |

- 따라서 **L4 정책을 변경한다고 중복로그인 구현을 변경할 필요는 없습니다.**
---

## 19. 현재 시스템에 추천하는 구조

현재 조건을 종합하면:

```text
JBoss EAP 7.2
Spring 5.3
다중 WAS
<distributable/>
L4
MariaDB
```

라면 다음 설계를 추천합니다.

```text
                         L4
              RR 또는 IP Hash
                         │
               ┌─────────┴─────────┐
               ▼                   ▼
             WAS1                WAS2
               │                   │
          HttpSession          HttpSession
          LOGIN_TOKEN          LOGIN_TOKEN
               │                   │
          Local Cache          Local Cache
               │                   │
               └─────────┬─────────┘
                         │
                   MariaDB
                  LOGIN_CONTROL
                         │
                         │
                 Source of Truth
                         │
                  USER01=TOKEN-B
```

고도화하면:

```text
             새로운 로그인
                  │
             DB TOKEN 갱신
                  │
               JMS Topic
            ┌─────┴─────┐
            ▼           ▼
          WAS1         WAS2
        Cache=B       Cache=B
```

입니다.

### 19-1. 실무 판단

**IP Hash라고 해서 Local SessionRegistry만으로 단순화하지 않는 것이 가장 중요합니다.**
IP Hash는 단지:

```text
동일 Source IP
→ 동일 WAS 가능성 증가
```

라는 **부하 분산 특성**일 뿐입니다.
중복로그인의 보장 조건은:

```text
USER_ID 기준 현재 유효 LOGIN_TOKEN이
모든 WAS에서 동일하게 보이는가?
```

입니다.
따라서 설계 기준은 다음 한 문장으로 정리할 수 있습니다.

> **Round-Robin에서는 공유 Login Registry가 명백하게 필수이고, IP Hash에서는 Local Registry가 동작하는 것처럼 보일 수 있지만 장애·IP 변경·다른 단말 로그인 때문에 결국 동일한 공유 Login Registry 구조가 필요합니다.**
> 현재 환경이라면 저는 **`MariaDB LoginToken Registry + WAS Local Cache(5초) + <distributable/>`를 1차안**, 즉시 강제 로그아웃이 요구되면 **`+ JMS Topic invalidation`을 최종안**으로 잡겠습니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
