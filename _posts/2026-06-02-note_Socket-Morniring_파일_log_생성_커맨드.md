---
layout: single
title: "Socket-Morniring_파일_log_생성_커맨드"
excerpt: "Socket-Morniring_파일_log_생성_커맨드"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-06-02"
last_modified_at: "2026-06-02 17:55:54 +0900"
mermaid: false
---
## 1. watch 모니터링을 log 파일 생성으로 변경
```bash
watch -n 3 "date '+%Y-%m-%d %H:%M:%S'; ss -Hant '( dport = :3306 )' | awk '{ state=\$1; peer=\$5; sub(/:[0-9]+$/, \"\", peer); gsub(/^\\[::ffff:/, \"\", peer); gsub(/^\\[/, \"\", peer); gsub(/\\]$/, \"\", peer); cnt[peer, state]++; total[peer]++; stateTotal[state]++; grand++; } END { printf \"%-18s %8s %10s %12s %10s %10s %10s\\n\", \"DB_SERVER\", \"ESTAB\", \"TIME_WAIT\", \"CLOSE_WAIT\", \"LAST_ACK\", \"SYN_SENT\", \"TOTAL\"; for (p in total) { printf \"%-18s %8d %10d %12d %10d %10d %10d\\n\", p, cnt[p, \"ESTAB\"], cnt[p, \"TIME-WAIT\"], cnt[p, \"CLOSE-WAIT\"], cnt[p, \"LAST-ACK\"], cnt[p, \"SYN-SENT\"], total[p]; } printf \"%-18s %8d %10d %12d %10d %10d %10d\\n\", \"TOTAL\", stateTotal[\"ESTAB\"], stateTotal[\"TIME-WAIT\"], stateTotal[\"CLOSE-WAIT\"], stateTotal[\"LAST-ACK\"], stateTotal[\"SYN-SENT\"], grand; }'"
```
### 1-1. 핵심 요점 및 실행 결론

기존 대화형 모니터링 도구인 `watch` 명령어는 터미널 화면 제어를 위한 ANSI Escape 문자(제어 코드)를 출력하므로 파일로 직접 리다이렉션(`>>`)할 경우 로그 파일이 깨지는 현상이 발생합니다.

따라서 이를 백그라운드 환경에서 안정적으로 동작하도록 무한 루프(`while true`) 기반의 데몬(Daemon) 스크립트 형태로 변환해야 하며, 요구사항에 맞춰 `awk` 내의 모든 `TOTAL` 집계 로직을 제거한 코드를 제시합니다.

### 1-2. 쉘 스크립트 기반 모니터링 적용 방안

아래의 코드를 `socket_monitor.sh` 파일로 작성하여 실행 권한(`chmod +x`)을 부여한 후 백그라운드에서 실행하는 것을 권장합니다.

{% raw %}

```bash
#!/bin/bash

# 모니터링 무한 루프 실행
while true; do
    # 1. 동적 로그 파일명 지정 (자정 경과 시 새로운 일자 파일로 자동 분리)
    LOG_FILE="socket-mornitoring-$(date +%Y%m%d).log"

    # 2. 소켓 상태 추출 및 파일 Append
    {
        date '+%Y-%m-%d %H:%M:%S'
        ss -Hant '( dport = :3306 )' | awk '{ 
            state=$1; 
            peer=$5; 
            
            # IPv4/IPv6 포트 및 특수문자 정제
            sub(/:[0-9]+$/, "", peer); 
            gsub(/^\[::ffff:/, "", peer); 
            gsub(/^\[/, "", peer); 
            gsub(/\]$/, "", peer); 
            
            # 상태 카운트 증가 및 고유 IP 기록
            cnt[peer, state]++; 
            peers[peer]=1; 
        } END { 
            # 헤더 출력 (TOTAL 제외)
            printf "%-18s %8s %10s %12s %10s %10s\n", "DB_SERVER", "ESTAB", "TIME_WAIT", "CLOSE_WAIT", "LAST_ACK", "SYN_SENT"; 
            
            # 데이터 출력 (TOTAL 컬럼 제외)
            for (p in peers) { 
                printf "%-18s %8d %10d %12d %10d %10d\n", p, cnt[p, "ESTAB"], cnt[p, "TIME-WAIT"], cnt[p, "CLOSE-WAIT"], cnt[p, "LAST-ACK"], cnt[p, "SYN-SENT"]; 
            } 
        }'
        echo "--------------------------------------------------------"
    } >> "${LOG_FILE}"

    # 3. 3초 대기 후 반복
    sleep 3
done
```

{% endraw %}

- _기술 설명_: 위 스크립트는 `nohup ./socket_monitor.sh > /dev/null 2>&1 &` 명령어를 통해 백그라운드 프로세스로 실행해야 사용자의 SSH 세션이 끊어져도 모니터링이 유지됩니다.

### 1-3. 아키텍처 관점의 변경 사항 매트릭스

|**변경 항목**|**기존 방식 (watch)**|**변경된 방식 (Shell Daemon)**|**기술적 사유**|
|---|---|---|---|
|**실행 주체**|`watch -n 3`|`while true; do ... sleep 3; done`|`watch`의 터미널 제어 문자 혼입 방지 및 순수 텍스트 로깅 확보|
|**로그 파일명**|지원 불가 (터미널 출력 전용)|`socket-mornitoring-$(date +%Y%m%d).log`|`date` 명령어를 루프 내에서 평가하여 자정 전환 시 자동 분리 (Daily Rotation)|
|**고유 IP 트래킹**|`total[peer]++` (합계 계산용)|`peers[peer]=1` (목록 저장용)|TOTAL 연산이 불필요해졌으므로, 배열의 키(Key) 값만 활용하여 메모리 점유 최소화|
|**출력 항목**|`grand`, `stateTotal` 연산 및 출력|해당 변수 및 관련 `printf` 구문 전면 삭제|사용자의 "TOTAL 제거" 요구사항 반영 및 `awk` 파싱 부하 감소|

### 1-4. 시스템 아키텍처 및 동작 구조 도식화

해당 스크립트가 시스템에서 동작하고 로그를 적재하는 라이프사이클을 구조화한 도식입니다.

코드 스니펫

```
flowchart TD
    A[Daemon Process\nwhile true] --> B(현재 System Time 확인\n%Y%m%d)
    B --> C{자정 변경 여부 판단}
    C -->|Yes| D[새로운 로그 파일 Target 지정\nsocket-mornitoring-새날짜.log]
    C -->|No| E[기존 로그 파일 Target 유지]
    D --> F[ss 커맨드 실행 및 awk 파싱\nTOTAL 항목이 제거된 파이프라인]
    E --> F
    F --> G[(로그 파일 Append I/O)]
    G --> H[sleep 3초 대기]
    H --> A
```

### 1-5. 실무 도입 시 임계 장애 포인트 및 회피 방안

_용어 설명 - 임계 장애 포인트(Critical Failure Point)_: 시스템의 가용 리소스가 한계에 달해 더 이상 정상적인 서비스를 제공할 수 없게 되는 치명적인 병목 지점이나 장애 유발 요소를 의미합니다.

- **임계 장애 포인트 1: Disk I/O 지속 발생에 따른 부하 발생**

    - 매 3초마다 디스크에 Write 연산을 수행하므로 하루에 28,800회의 I/O가 발생합니다. 서비스 트래픽이 높은 DB 서버의 경우 이로 인해 발생하는 디스크 경합이 메인 서비스의 레이턴시(Latency)를 증가시킬 위험이 있습니다.

    - _회피 방안_: 로그 파일의 적재 위치(`LOG_FILE`의 경로)를 OS 메인 디스크(`/`)나 DB 데이터 영역(`/var/lib/mysql`)이 아닌, 독립된 마운트 볼륨이나 `/tmp`와 같은 메모리 기반 파일 시스템(tmpfs)으로 분리하여 I/O 간섭을 물리적으로 차단해야 합니다.

- **임계 장애 포인트 2: 로그 파일 비대화로 인한 Disk Full 장애**

    - 날짜별로 파일이 분리되기는 하지만, 오래된 로그 파일이 지속적으로 누적되면 디스크 공간이 고갈되어 OS 패닉이나 DB 프로세스 중단을 유발할 수 있습니다.

    - _회피 방안_: 리눅스의 `logrotate` 유틸리티를 설정하거나, 스크립트 상단에 `find /로그경로 -name "socket-mornitoring-*.log" -mtime +7 -delete` 구문을 추가하여 7일 이상 경과한 로그 파일을 자동으로 파기(Purge)하는 생명주기(Lifecycle) 관리를 강제해야 합니다.

- **임계 장애 포인트 3: 프로세스 고아(Zombie)화 및 중복 실행**

    - 관리자의 실수로 해당 쉘 스크립트를 여러 번 백그라운드에서 실행하게 되면 중복 로깅이 발생하여 부하가 배수 단위로 폭증합니다.

    - _회피 방안_: `nohup`과 `&`을 이용한 백그라운드 실행 방식 대신 리눅스 `systemd` 서비스(Service) 파일로 등록하여 데몬을 관리하면 단일 프로세스 실행(Singleton) 보장 및 서버 재부팅 시 자동 재시작을 통제할 수 있습니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
