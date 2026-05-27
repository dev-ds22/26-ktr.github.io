---
layout: single
title: "FO_DB_Connection_모니터링"
excerpt: "FO_DB_Connection_모니터링"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-05-27"
last_modified_at: "2026-05-27 14:28:26 +0900"
---
## 1. SS 를 통한 모니터링
### 1-1. [결론] OS 레벨의 실시간 TCP 커넥션 모니터링 완전 가능

`ss -Hant '( dport = :3306 )'` 명령어는 WAS 서버 인프라 환경에서 MySQL/MariaDB(`3306`)로 맺어진 **실제 네트워크 소켓의 개수와 상태를 실시간으로 추적하는 데 매우 강력하고 타당한 방법**입니다. 애플리케이션(DBCP) 지표의 신뢰성을 OS 계층에서 교차 검증할 때 필수적으로 사용됩니다.

### 1-2. 명령어 옵션 및 필터링 메커니즘 해석

- `-H` (No Header): 헤더 행을 생략하여 `wc -l` 등의 스크립트 카운팅 파이프라인 연산에 적합합니다.

- `-a` (All): 연결된 세션뿐만 아니라 `TIME_WAIT`, `CLOSE_WAIT` 등 모든 소켓 상태를 출력합니다.

- `-n` (Numeric): 도메인/서비스명 해석(Resolving)을 차단하여 명령 실행 속도를 극대화하고 CPU 부하를 방지합니다.

- `-t` (TCP): TCP 프로토콜 소켓만 필터링합니다.

- `'( dport = :3306 )'`: 목적지 포트(DB 서버)가 3306인 아웃바운드 커넥션만 정확히 격리합니다.

### 1-3. 소켓 상태별 장애 진단 매트릭스

`ss` 명령어 출력 결과의 `State` 열을 분석하여 현재 DBCP 및 트랜잭션의 상태를 판별할 수 있습니다.

|**TCP State**|**e-commerce 아키텍처 관점의 현상 해석**|**대응 및 확인 필요 사항**|
|---|---|---|
|**ESTABLISHED**|WAS와 DB 간 정상 연결되어 사용 중이거나 대기 중인 활성 커넥션|이 개수가 DBCP의 `maxTotal`과 일치하면 풀 고갈 상태|
|**TIME_WAIT**|연결 종료 후 소켓 재사용을 위해 OS가 임시 유지 중인 상태|단시간 내 대량의 커넥션 생성/소멸 발생 (DBCP 미사용 의심)|
|**CLOSE_WAIT**|DB 서버는 커넥션을 닫았으나 WAS가 소켓을 닫지 못하고 누수된 상태|**Critical.** 애플리케이션 내 커넥션 누수(Leak) 발생 확증|

### 1-4. 실무 모니터링 활용 및 확장 스크립트

단순 텍스트 출력을 넘어, 실시간 개수 집계 및 모니터링을 위해 아래 명령어로 확장하여 사용해야 효과적입니다.

Bash

```
# 1. 현재 DB로 맺어진 총 TCP 커넥션 개수만 즉시 카운트 (maxTotal 비교용)
ss -Hant '( dport = :3306 )' | wc -l

# 2. TCP 상태별(ESTABLISHED, TIME_WAIT 등) 커넥션 개수 그룹핑 집계
ss -Hant '( dport = :3306 )' | awk '{print $1}' | sort | uniq -c

# 3. watch 명령어를 조합하여 1초 주기로 실시간 커넥션 변화 추이 모니터링
watch -n 1 "ss -Hant '( dport = :3306 )' | awk '{print \$1}' | sort | uniq -c"
```

### 1-5. 아키텍처적 한계점 및 보완 사항 (확인 불가능한 내용)

- **한계점:** `ss` 명령어는 네트워크 계층의 물리적 소켓 상태만 보여줄 뿐, **WAS 내부의 어떤 스레드(예: `default task-72`)가 해당 소켓을 점유하고 있는지, 어떤 SQL이나 외부 API(OkHttp3) 때문에 멈춰 있는지는 식별할 수 없습니다.**

- **보완책:** 따라서 `ss` 명령어로 `ESTABLISHED` 개수가 `maxTotal`에 도달한 것을 인지한 즉시, 이전 단계에서 제시한 **Application 레이어의 스레드 덤프(Thread Dump)를 병행 추출**하여 원인 코드를 찾아내야 합니다.
### 1-6. [결론] 임시 쉘 스크립트 로깅의 한계와 APM/메트릭 기반 통합 모니터링으로의 전환

`watch` 명령어와 `ss`를 조합하면 가볍게 실시간 커넥션 상태를 파일로 기록할 수 있으나, 가용성과 유지보수성이 핵심인 E-commerce 아키텍처 환경에서는 **폴링 공백(Polling Gap)으로 인한 순간적인 커넥션 스파이크(Spike) 누락 위험**이 존재합니다. 따라서 단기적인 디버깅용으로만 쉘 로깅을 사용하고, 궁극적으로는 **Spring Actuator + Prometheus + Grafana** 또는 **eBPF 기반의 커넥션 모니터링**체계로 개선해야 합니다.

## 2. 개선 된 모니터링 방법
### 2-1. watch 명령어 기반 DB Connection 실시간 오류 로깅 방법

`watch` 명령어는 본래 터미널 출력용이므로, 지속적인 파일 기록(Logging)을 위해서는 내부에서 리다이렉션(`>>`)을 수행하거나 프로덕션 환경에 적합한 `while loop` 백그라운드 데몬 방식을 사용해야 합니다.

#### 2-1-1. ① 단기 디버깅용 watch 로깅 명령어 (터미널 포그라운드 실행)

Bash

```
# 1초 주기로 현재 시간, ESTABLISHED 소켓 수, TIME_WAIT 소켓 수를 한 줄로 포맷팅하여 로그 파일에 누적
watch -n 1 "echo \$(date +'%Y-%m-%d %H:%M:%S') '[ESTABLISHED]:' \$(ss -Hant '( dport = :3306 )' | grep -c ESTAB) '[TIME_WAIT]:' \$(ss -Hant '( dport = :3306 )' | grep -c TIME-WAIT) >> /var/log/myapp/db_connection_ss.log"
```

#### 2-1-2. ② 프로덕션 권장형 백그라운드 쉘 스크립트 (DBCP 고갈 및 경고 발생 시 자동 기록)

`watch` 대신 톰캣/WAS 서버 내부 세션 백그라운드에서 상시 구동하며, 커넥션 수가 임계치(예: 메인 DB `maxTotal=100` 중 80개 이상 점유 시)를 초과할 때 스레드 덤프까지 연동하여 로깅하는 아키텍처 관점의 고도화 스크립트입니다.

Bash

```
#!/bin/bash
LOG_FILE="/var/log/myapp/db_conn_breach.log"
THRESHOLD=80 # 임계치 설정 (DBCP maxTotal의 80%)
PID=$(pgrep -f "wildfly|tomcat|spring") # WAS 프로세스 ID 자동 추출

while true; do
    CURRENT_CONN=$(ss -Hant '( dport = :3306 )' | grep -c ESTAB)
    if [ "$CURRENT_CONN" -ge "$THRESHOLD" ]; then
        TIMESTAMP=$(date +'%Y-%m-%d %H:%M:%S')
        echo "[$TIMESTAMP] ALERT: DB Connection Count ($CURRENT_CONN) exceeded threshold ($THRESHOLD)!" >> "$LOG_FILE"
        # 즉시 스레드 덤프를 수행하여 어떤 소스코드(OkHttp3 등)가 범인인지 연계 기록
        if [ -n "$PID" ]; then
            echo "--- Thread Dump at $TIMESTAMP ---" >> "$LOG_FILE"
            jstack -l "$PID" | head -n 100 >> "$LOG_FILE" # 상위 100줄 요약 기록
        fi
    fi
    sleep 1 # 1초 주기 검사
done
```

### 2-2. 모니터링 방법 고도화 비교 테이블

OS 레벨 쉘 스크립트 방식과 아키텍처 표준 개선안의 다각도 비교입니다.

| **비교 항목**            | **기존 방식 (watch + ss 쉘 스크립트)**          | **개선 방식 (Spring Actuator + Prometheus + Grafana)** |
| -------------------- | -------------------------------------- | -------------------------------------------------- |
| **모니터링 레이어**         | OS 네트워크 소켓 계층 (L4)                     | Application 내부 DBCP 상태 및 DB 엔진 계층 통합               |
| **데이터 영속성**          | 로컬 텍스트 파일 (디스크 Full 위험 및 검색 불편)        | 시시계열 데이터베이스(TSDB) 저장 (장기 트렌드 분석 가능)                |
| **시스템 오버헤드**         | 주기적 프로세스 분기(`ss`, `grep`)로 인한 CPU 스파이크 | 풀 메모리 메트릭 단순 노출로 가볍고 안정적임                          |
| **알림 연동 (Alerting)** | 이메일/슬랙 연동을 위해 별도 쉘 구현 필요               | Grafana Webhook을 통해 슬랙, PagerDuty 등 즉시 연동          |
| **탐지 정밀도**           | 밀리세컨드(ms) 단위의 순간 폭증 누락 가능              | 수집 주기 최적화 및 프로메테우스 카운터로 정밀 추적                      |

### 2-3. 개선 된 모니터링 아키텍처 제시 (Spring 5.3 + Grafana 표준)

가장 추천하는 개선 안은 **Micrometer 라이브러리**를 이용하여 Commons DBCP2 또는 HikariCP 내부 메트릭을 시스템 성능 저하 없이 Grafana 인프라로 가시화하는 전략입니다.

#### 2-3-1. ① Spring 애플리케이션 설정 (DBCP 메트릭 노출)

Spring 5.3 환경의 `pom.xml` 또는 `build.gradle`에 `micrometer-registry-prometheus` 의존성을 주입하고 Actuator를 활성화합니다.

XML

```
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <version>1.9.x</version> 
</dependency>
```

이후 Grafana 대시보드에서 `db_cp_active_connections` 및 `db_cp_pending_threads`(커넥션을 얻기 위해 대기 중인 스레드 수) 메트릭을 실시간 그래프로 구성합니다. 대기 스레드가 1개 이상 발생하는 즉시 슬랙(Slack) 경고 채널로 발송하도록 알람 룰을 구성하여 장애에 선제 대응합니다.

### 2-4. [추가 정리] 기술 심화: 나이퀴스트-섀넌(Nyquist-Shannon) 정리 기반 폴링 공백 리스크

`watch -n 1` 또는 쉘 스크립트의 `sleep 1` 방식은 1초 간격으로 상태를 조회하는 **폴링(Polling) 방식**입니다.

- 신호 처리 이론 및 모니터링 아키텍처 관점에서, 1초 미만(예: 200ms 단위)으로 순간 발생했다가 사라지는 **DBCP 커넥션 풀의 고갈 스파이크 현상**은 1초 주기 폴링 시스템에서 완전히 누락되는 현상이 발생합니다.

- 이를 물리적으로 해결하기 위해 최신 2026년 시스템 아키텍처 표준에서는 이벤트 주도(Event-driven) 방식으로 소켓 오프닝을 감지하는 **eBPF(Extended Berkeley Packet Filter)** 기술 기반의 에이전트(예: Grafana Beyla, Datadog)를 커널에 이식하여, 애플리케이션 수정과 폴링 오버헤드 없이 모든 3306 포트 트래픽 결함을 0ms 오차로 추적하는 추세입니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
