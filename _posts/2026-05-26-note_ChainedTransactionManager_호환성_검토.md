---
layout: single
title: "ChainedTransactionManager_호환성_검토"
excerpt: "ChainedTransactionManager_호환성_검토"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-05-26"
last_modified_at: "2026-05-26 16:23:35 +0900"
---
### 1. Spring Framework 5.3과 매칭되는 Spring Data Commons 버전

Spring Framework 5.3 버전은 매우 긴 생태계 주기를 가졌던 버전(Spring Boot 2.4부터 2.7까지의 기반 프레임워크)이기 때문에 딱 하나의 버전과 실행되지 않고, **Spring Data Commons 2.4.x ~ 2.7.x 버전 전체**와 호환됩니다.

대표적인 Spring Boot 버전별 매칭 구조는 다음과 같습니다.

| **Spring Boot 버전**  | **기반 Spring Framework 버전** | **연동되는 Spring Data Commons 버전 (릴리즈 트레인명)** |
| ------------------- | -------------------------- | ------------------------------------------ |
| **Spring Boot 2.4** | Spring Framework 5.3.x     | **Spring Data Commons 2.4.x** (2020.0)     |
| **Spring Boot 2.5** | Spring Framework 5.3.x     | **Spring Data Commons 2.5.x** (2021.0)     |
| **Spring Boot 2.6** | Spring Framework 5.3.x     | **Spring Data Commons 2.6.x** (2021.1)     |
| **Spring Boot 2.7** | Spring Framework 5.3.x     | **Spring Data Commons 2.7.x** (2021.2)     |

### 2. Spring 5.3 버전에서 Deprecated 되었나?

정확한 팩트를 체크하자면 **"Spring 5.3 기반의 생태계(Spring Boot 2.5 이상)를 사용 중이라면 Deprecated 상태가 맞습니다."** 다만 소스코드 레벨에서의 정확한 구분이 필요합니다.

> **⚠️ 핵심 구조적 차이**
> 
> `ChainedTransactionManager`는 Spring Framework 코어 프로젝트(`spring-tx`)의 클래스가 아니라, **Spring Data Commons** 프로젝트의 클래스입니다.
> 
> 따라서 **Spring Framework 자체 버전(5.3) 가이드라인에는 이 클래스의 Deprecated 명세가 존재하지 않습니다.**

실제 프로젝트 적용 기준으로 보시면 이해가 쉽습니다.

- **Spring Framework 5.3 + Spring Data 2.4 조합 (Spring Boot 2.4 환경)**

    - 이 환경에서는 아직 Deprecated 되지 않았습니다. 정상적으로 경고 없이 사용 가능합니다.

- **Spring Framework 5.3 + Spring Data 2.5 조합 (Spring Boot 2.5 환경)**

    - 이 시점부터 Spring Data Commons 2.5가 적용되면서 `ChainedTransactionManager`에 공식적으로 `@Deprecated` 가 지정되었습니다.

**💡 요약하자면:** 동일한 Spring Framework 5.3 버전을 쓰고 계시더라도, 함께 쓰고 계신 **Spring Boot 버전이 2.5 이상이거나 Spring Data Commons 버전이 2.5 이상**이라면 Deprecated 된 것이 맞으므로 대안(JTA 도입 혹은 TransactionSynchronization 분리 등)을 고려하셔야 합니다.

## 3. spring framework 5.3 과 spring-data-commons 1.13.7

결론부터 말씀드리면, **Spring Framework 5.3 환경에서 `spring-data-commons 1.13.7` 버전을 사용하는 것은 타당성이 매우 낮으며 절대 권장하지 않습니다.** 이유와 함께, 해당 조합으로 `ChainedTransactionManager`를 실무에 도입했을 때 발생하는 치명적인 문제점들을 정리해 드립니다.

### 3-1. Spring Framework 5.3 + spring-data-commons 1.13.7 타당성 검토

두 라이브러리 사이에는 약 3~4년의 시간 차이와 세대 격차(Generation Gap)가 존재하므로 심각한 호환성 문제를 야기합니다.

- **기반 프레임워크의 불일치:** `spring-data-commons 1.13.7`은 **Spring Data Ingalls** 릴리즈 트레인에 포함된 버전으로, 2017년에 **Spring Framework 4.3.x**를 베이스라인으로 설계되었습니다. 반면 `Spring Framework 5.3`은 2020년 말에 릴리즈된 5세대의 핵심 버전입니다.

- **런타임 오류 위험 (`NoSuchMethodError` 등):** Spring 5 버전으로 올라오면서 코어 프레임워크 내부 인터페이스와 트랜잭션 추상화 구조가 대거 변경되거나 리팩토링되었습니다. 1.x 대의 옛날 라이브러리를 5.3에 강제로 엮으면, 컴파일은 통과하더라도 서버 구동 시점이나 실제 트랜잭션이 수행되는 런타임에 클래스/메서드 참조 오류가 발생할 확률이 매우 높습니다.

- **치명적인 보안 취약점 (RCE):** `spring-data-commons` 1.13.11 미만 버전(1.13.7 포함)은 전 세계적으로 문제가 되었던 원격 코드 실행(RCE) 취약점(CVE-2018-1273)을 그대로 품고 있습니다. 실무 프로덕션 환경이라면 이 버전은 보안 검수를 통과할 수 없습니다.

### 3-2. 이 조합에서 ChainedTransactionManager 사용 시 실무 문제점

버전 호환성 문제를 제쳐두더라도, 이 환경에서 `ChainedTransactionManager`를 쓰는 것은 시스템에 데이터 정합성 결함을 방치하는 것과 같습니다.

### 3-3. ① 데이터 불일치 (Partial Commit / 롤백 불가)

`ChainedTransactionManager`는 완벽한 분산 트랜잭션(2PC)이 아니라 **여러 트랜잭션을 단순 체인(순차 실행)으로 묶은 1단계 커밋 에뮬레이션 구조**입니다.

만약 `TM 1`, `TM 2`, `TM 3` 세 개를 묶었다고 가정해 봅시다.

- **트랜잭션 시작:** `TM 1` ➡️ `TM 2` ➡️ `TM 3` 순서로 시작

- **커밋 단계:** 역순으로 진행됩니다. `TM 3(성공)` ➡️ `TM 2(성공)` ➡️ `TM 1(실패!)`

- **문제 발생:** `TM 1`에서 커밋이 실패하거나 네트워크가 끊어지면, **이미 커밋이 완료된 `TM 2`와 `TM 3`의 데이터는 물리적으로 롤백이 불가능**합니다. 결국 DB 간 데이터가 찢어지는(Inconsistency) 현상이 실무에서 빈번하게 발생합니다.

### 3-4. ② Spring 5.3 리소스 동기화 메커니즘과의 충돌

Spring 5.3은 트랜잭션 동기화(`TransactionSynchronizationManager`)와 스레드 로컬(ThreadLocal) 리소스 관리 방식이 현대적인 비동기/리액티브 흐름까지 고려하여 고도화되었습니다.

반면 옛날 버전인 1.13.7의 `ChainedTransactionManager`는 구형 동기화 구조에 머물러 있습니다. 이로 인해 다음과 같은 현상이 발생할 수 있습니다.

- 트랜잭션이 끝났음에도 커넥션이 제대로 반환되지 않고 스레드에 남아있는 리소스 누수(Leak)가 발생할 수 있습니다.

- 다른 비즈니스 로직이나 다음 요청이 해당 톰캣 스레드를 재사용할 때, 이전 트랜잭션의 찌꺼기 리소스가 오염을 일으켜 원인 모를 오동작을 유발합니다.

### 3-5. ③ 디버깅과 모니터링의 한계

1.13.7 버전의 체인 트랜잭션 매니저는 내부에서 예외가 발생했을 때, 정확히 어떤 데이터 소스(어떤 매니저)의 커밋/롤백 단계에서 에러가 전파되었는지 추적하기 어렵게 구현되어 있습니다. 복잡한 다중 DB 환경에서 에러 로그를 봐도 장애 원인을 파악하기가 극도로 어렵습니다.

### 3-6. 🛠️ 권장하는 해결 방향

현재 Spring Framework 5.3 환경을 유지하셔야 한다면 다음과 같이 조치하는 것이 올바른 방향입니다.

1. **Spring Data 버전 업그레이드:** Spring 5.3의 공식 짝꿍인 **Spring Data 2.5.x ~ 2.7.x** 버전으로 올리십시오. (이렇게 하면 호환성 및 보안 문제가 해결되며, `ChainedTransactionManager`에 공식 Deprecated 경고가 표시됩니다.)

2. **애플리케이션 이벤트 구조로 전환:** 체인 트랜잭션을 걷어내고, 메인 DB 트랜잭션 안에서 서브 작업들은 **Spring ApplicationEvent(및 `@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)`)** 방식을 활용해 분리해 내는 아키텍처로 전환하는 것이 실무에서 가장 안전하고 정석적인 방법입니다.

## 4. 참고
```xml
<bean id="transactionManager" class="org.springframework.data.transaction.ChainedTransactionManager" > 

    <constructor-arg>
        <list>
            <ref bean="txManager"/>
            <ref bean="txManagerMail"/>
        </list>
    </constructor-arg>
</bean>
```
`ChainedTransactionManager`에 등록된 `txManager`와 `txManagerMail` **두 개의 트랜잭션 매니저 모두 각각의 DB 커넥션을 동시(순차적)에 획득하고 사용하게 됩니다.**

왜 그렇게 동작하는지, 그리고 이 설정을 사용할 때 **반드시 주의해야 할 점**을 정리해 드립니다.

### 4-1. 동작 원리: 왜 둘 다 커넥션이 필요할까?

`ChainedTransactionManager`는 여러 개의 독립된 트랜잭션 매니저를 하나로 묶어서 **순차적으로 트랜잭션을 시작하고, 역순으로 커넥션을 커밋/롤백**하는 방식으로 동작합니다.

만약 어떤 비즈니스 로직(Service 메서드)에 `@Transactional`이 걸려 있고, 여기에 `ChainedTransactionManager`가 적용된다면 다음과 같은 순서로 일이 진행됩니다.

1. **트랜잭션 시작 단계:**

    - 정의된 순서대로 `txManager`가 먼저 DB 커넥션을 가져와 트랜잭션을 시작합니다.

    - 곧이어 `txManagerMail`도 자신의 DB 커넥션을 가져와 트랜잭션을 시작합니다.

2. **비즈니스 로직 수행 (CRUD):**

    - 이 메서드 안에서 `txManagerMail`에 해당하는 DB 조작을 단 한 줄도 하지 않더라도, **이미 트랜잭션이 시작되었기 때문에 두 매니저 모두 각각의 커넥션 풀(Connection Pool)에서 커넥션을 하나씩 점유**하고 있는 상태가 됩니다.

3. **트랜잭션 종료 단계:**

    - 로직이 성공하면 역순(`txManagerMail` ➡️ `txManager`)으로 커밋을 진행하고 커넥션을 반납합니다.

따라서, 실제 CRUD 대상이 어떤 DB이든 상관없이 **해당 트랜잭션 범위에 들어오는 순간 두 DB 모두 커넥션이 필요**합니다.

### 4-2. ⚠️ 중요: Spring 5.3에서의 Deprecated 경고

스프링 프레임워크 5.3 버전을 쓰고 계신다면 한 가지 꼭 아셔야 할 점이 있습니다. **`ChainedTransactionManager`는 Spring 5.3부터 공식적으로 Deprecated(사용 권장 안 함)** 되었습니다.

> **이유는 100% 안전한 분산 트랜잭션(2PC)이 아니기 때문입니다.**
> 
> 두 개의 DB를 쓰다가 `txManagerMail`은 정상 커밋되었는데, 그 직후 `txManager`를 커밋하는 과정에서 네트워크 장애나 DB 서버 다운이 발생하면, 앞선 커밋은 취소할 수 없어 **데이터 원자성(Atomicity)이 깨지는 현상**이 발생할 수 있습니다.

### 4-3. 💡 대안 및 권장 사항

1. **가장 좋은 방법 (비즈니스 로직 분리):**

    - 두 DB 작업을 하나의 트랜잭션 묶음으로 처리하지 말고, 서비스를 쪼개는 것입니다.

    - 메인 로직(`txManager`)을 먼저 커밋 성공시킨 후, 이어서 메일 관련 로직(`txManagerMail`)을 별도의 트랜잭션(`Propagation.REQUIRES_NEW`)이나 비동기(이벤트 기반)로 처리하는 방향을 권장합니다.

2. **진짜 100% 동기화가 필요한 경우:**

    - 데이터가 절대 틀어지면 안 되는 강력한 글로벌 트랜잭션이 필요하다면, `ChainedTransactionManager` 대신 JTA(Java Transaction API)를 지원하는 **Atomikos** 같은 전문 외부 트랜잭션 매니저를 도입하셔야 합니다.

### 4-4. Connection Pool 불균형으로 인한 시스템 전체 마비(Deadlock) 위험

현재 설정(`dataSource`: 100, `dataSourceMail`: 10)과 앞서 정의된 `ChainedTransactionManager` 및 와일드카드(`*`) AOP 포인트컷이 결합될 경우, **동시 요청이 10개를 초과하는 순간 메인 DB(`dataSource`)의 커넥션까지 함께 고갈되어 시스템 전체가 마비되는 전면적 데드락(Deadlock) 장애**가 발생합니다.

#### 4-4-1. Connection Pool 불균형의 핵심 문제점 요약

| **지표/상황**      | **핵심 DB (dataSource)** | **메일 DB (dataSourceMail)** | **아키텍처적 결과 및 현상**                                       |
| -------------- | ---------------------- | -------------------------- | ------------------------------------------------------- |
| **최대 커넥션 풀**   | `maxTotal = 100`       | `maxTotal = 10`            | **병목 지점:** 시스템의 실질 최대 동시 처리량은 **10**으로 제한됨              |
| **트랜잭션 시작 시**  | 커넥션 즉시 획득 (성공)         | 커넥션 부족 시 대기 (Blocking)     | 메인 DB 커넥션을 쥔 채 메일 DB 커넥션을 기다리는 **자원 데드락** 발생            |
| **11번째 동시 요청** | 커넥션 획득 완료 (11/100)     | 풀 고갈로 인한 무한 대기             | `maxWaitMillis` 초과 시 `GetConnectionTimeoutException` 발생 |

##### 4-4-1-1. 세부 장애 메커니즘 분석

#### 4-4-2. ① 자원 보유 및 대기(Hold and Wait)로 인한 데드락

`ChainedTransactionManager`는 생성자 인자에 주입된 순서(`txManager` ➡️ `txManagerMail`)대로 트랜잭션을 시작하며 커넥션을 점유합니다.

1. **요청 유입:** 11개의 동시 요청이 서비스 계층으로 유입됩니다.

2. **메인 DB 점유:** 11개 스레드 모두 `txManager`로부터 커넥션 획득에 성공합니다. (사용량: 11/100)

3. **메일 DB 요청:** 이어서 11개 스레드가 `txManagerMail`에 커넥션을 요청합니다.

4. **교착 상태 진입:** 먼저 도달한 10개 스레드가 메일 커넥션을 모두 고갈시킵니다. (사용량: 10/10). **11번째 스레드는 메일 DB 커넥션을 얻기 위해 대기(Blocking) 상태로 전환되는데, 이때 이미 획득한 메인 DB 커넥션을 반환하지 않고 붙잡고 있습니다.**

#### 4-4-3. ② 메인 DB 커넥션 풀의 도미노 고갈 (Cascading Exhaustion)

트래픽이 지속해서 유입되면 메일 DB의 풀(`maxTotal=10`) 때문에 대기하는 스레드가 90개까지 늘어날 수 있습니다. 결과적으로 **메인 DB는 풀 용량(100)이 남아있음에도 불구하고, 대기 스레드들이 커넥션을 독점하여 단순 조회 마저도 불가능한 상태**로 전염됩니다.

##### 4-4-3-1. 해결을 위한 아키텍처 가이드라인

본 문제를 해결하기 위해 풀 개수를 강제로 맞추는 것은 임시방편일 뿐이며, 아래와 같이 설정을 분리해야 합니다.

- **최우선 조치 (AOP 분리):** `ChainedTransactionManager` 체인에서 `txManagerMail`을 즉시 제거하고, 메일 발송 서비스는 메인 트랜잭션과 독립된 별도의 전파 속성(`PROPAGATION_REQUIRES_NEW`) 또는 비동기 레이어로 격리해야 합니다.

- **적정 Pool 산정 가이드:** 메일 DB처럼 핵심 비즈니스가 아닌 서브 시스템은 커넥션을 동기적으로 묶지 말고, 메시징 큐(예: RabbitMQ, Kafka)나 인메모리 큐를 거쳐 가도록 아키텍처를 변경하여 메인 시스템의 `maxTotal` 성능을 온전히 활용하도록 설계해야 합니다.

### 4-5. [추가 정리] 기술 심화: Connection Pool Deadlock 공식 계산법

HikariCP 및 DBCP 환경에서 다중 데이터 소스나 스레드 풀이 결합될 때 데드락을 방지하기 위한 최소 커넥션 풀 공식은 다음과 같습니다.

   ![Pasted_image_20260526133959](./../../images/2026-05-26-note_ChainedTransactionManage/Pasted_image_20260526133959.png)
- $Tn$ : 동시 실행 가능한 최대 스레드 수 (톰캣 스레드 풀 등)
- $Cm$ : 하나의 스레드가 하나의 트랜잭션을 완료하기 위해 동시에 필요한 최대 커넥션 수

현재 시스템은 구조적으로 하나의 스레드가 완전히 트랜잭션을 끝내기 위해 반드시 **2개**의 커넥션($Cm = 2$)이 동시 확보되어야 하므로, 메일 DB의 `maxTotal`이 최소 톰캣 스레드 개수 이상 확보되지 않으면 무조건 구조적 데드락 위험 노출군에 속하게 됩니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
