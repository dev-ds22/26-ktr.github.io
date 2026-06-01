
#### `ss -tanp | grep '192.168.100.50:8080'` 상태별 모니터링 커맨드 3가지

##### 1. 5초마다 대상 서버 연결 상태별 개수 확인

```bash
watch -n 5 "ss -tanp | grep '192.168.100.50:8080' | awk '{state[\$1]++} END {for (s in state) print s, state[s]}' | sort"
```

설명:

- `192.168.100.50:8080`으로 연결된 TCP Socket을 상태별로 집계합니다.
    
- `ESTAB`, `TIME-WAIT`, `CLOSE-WAIT`, `SYN-SENT` 등이 몇 개인지 확인할 수 있습니다.
    
- 장애 상황에서 가장 먼저 쓰기 좋은 커맨드입니다.  
    예상 출력:
    

```bash
CLOSE-WAIT 2
ESTAB 15
TIME-WAIT 40
```

##### 2. 5초마다 전체 목록과 상태별 개수를 같이 확인

```bash
watch -n 5 "echo '===== DETAIL ====='; ss -tanp | grep '192.168.100.50:8080'; echo; echo '===== COUNT BY STATE ====='; ss -tanp | grep '192.168.100.50:8080' | awk '{state[\$1]++} END {for (s in state) print s, state[s]}' | sort"
```

설명:

- 실제 연결 목록과 상태별 집계를 같이 봅니다.
    
- 특정 WAS 프로세스가 어떤 상태의 연결을 많이 가지고 있는지 확인할 때 유용합니다.
    
- `-p` 옵션은 프로세스 정보를 보여주지만, 일반 계정에서는 일부 PID/프로세스명이 안 보일 수 있습니다. 필요하면 `sudo` 권한이 필요합니다.
    

##### 3. 5초마다 주요 상태만 개별 카운트

```bash
watch -n 5 "echo -n 'ESTAB='; ss -tanp state established | grep '192.168.100.50:8080' | wc -l; echo -n 'CLOSE-WAIT='; ss -tanp state close-wait | grep '192.168.100.50:8080' | wc -l; echo -n 'TIME-WAIT='; ss -tanp state time-wait | grep '192.168.100.50:8080' | wc -l; echo -n 'SYN-SENT='; ss -tanp state syn-sent | grep '192.168.100.50:8080' | wc -l"
```

설명:

- 운영 중 자주 보는 상태만 분리해서 확인합니다.
    
- `ESTAB`: 정상 연결 중
    
- `CLOSE-WAIT`: 상대 서버가 닫았지만 WAS 쪽 정리가 완료되지 않은 상태
    
- `TIME-WAIT`: 짧은 연결이 많이 생성될 때 증가 가능
    
- `SYN-SENT`: 대상 서버 연결 수립 지연 또는 네트워크 문제 가능
    

#### 실무 권장

장애 분석 중에는 1번으로 상태 변화를 먼저 보고, `CLOSE-WAIT`나 `TIME-WAIT`가 많으면 2번으로 상세 목록을 확인하는 방식이 좋습니다. `Socket closed` 원인 분석에서는 특히 `CLOSE-WAIT`, `TIME-WAIT`, `SYN-SENT` 증가 여부를 중점적으로 봐야 합니다.

## LAST-ACK

#### `LAST-ACK`가 많을 경우 문제점

`LAST-ACK`는 TCP 연결 종료 과정에서 **로컬 서버가 FIN을 보낸 뒤, 상대방의 마지막 ACK를 기다리는 상태**입니다. RFC 9293의 TCP 상태 모델 기준으로 `LAST-ACK`는 “보낸 연결 종료 요청에 대한 ACK를 기다리는 상태”에 해당합니다. `ss`는 Linux에서 Socket 상태를 확인하는 도구이며 `netstat`보다 더 많은 TCP 상태 정보를 표시할 수 있습니다. ([IETF Datatracker](https://datatracker.ietf.org/doc/html/rfc9293?utm_source=chatgpt.com "RFC 9293 - Transmission Control Protocol (TCP)"))

##### 1. 상태 의미

```text
상대방 FIN 수신
→ 로컬 ACK 응답
→ 애플리케이션 close 처리
→ 로컬 FIN 전송
→ LAST-ACK 상태
→ 상대방 ACK 수신
→ CLOSED
```

즉, `LAST-ACK`는 연결 종료의 마지막 단계입니다. 짧게 존재하는 것은 정상입니다. 그러나 **많이 쌓이거나 오래 유지되면 비정상 가능성**이 있습니다.

#### 2. `LAST-ACK`가 많을 때 의심할 수 있는 문제

|구분|가능 원인|설명|
|--:|---|---|
|1|상대방 ACK 미수신|로컬이 FIN을 보냈지만 상대 서버/클라이언트의 마지막 ACK가 오지 않음|
|2|네트워크 패킷 손실|FIN/ACK 패킷이 중간에서 유실되거나 지연됨|
|3|방화벽/L4 세션 정리|중간 장비가 종료 단계 패킷을 비정상적으로 차단/정리|
|4|상대 서버 비정상|검색 API 서버, DB 서버, Proxy가 종료 ACK를 정상 반환하지 못함|
|5|로컬 서버 고부하|WAS CPU/Thread/네트워크 큐 지연으로 종료 처리가 늦어짐|
|6|대량 단기 연결|짧은 HTTP 연결이 대량 생성되어 종료 상태가 순간적으로 쌓임|
|7|Keep-Alive 불일치|WAS/OkHttp/LB/검색 API 서버의 keep-alive timeout 정책 불일치|
|8|NAT/LB 경유 문제|NAT 테이블, L4 세션 테이블, 방화벽 idle timeout과 충돌|

#### 3. WAS/OkHttp 관점에서의 의미

`OkHttp → 검색 API 서버` 호출 중 `LAST-ACK`가 많다면, WAS 쪽에서 연결 종료를 진행했지만 검색 API 서버 또는 중간 장비로부터 마지막 ACK를 제대로 못 받고 있을 가능성이 있습니다.

|관점|해석|
|--:|---|
|`TIME-WAIT` 많음|짧은 연결이 많고 로컬이 정상 종료 완료 후 대기 중일 가능성|
|`CLOSE-WAIT` 많음|상대가 닫았는데 WAS 애플리케이션이 close를 늦게 하거나 누락 가능성|
|`LAST-ACK` 많음|WAS가 닫으려고 FIN을 보냈지만 상대 ACK가 안 돌아오는 상태|
|`SYN-SENT` 많음|연결 수립 자체가 지연되거나 대상 서버 접근 문제|
|`ESTAB` 많음|실제 연결이 많이 유지 중|

#### 4. 장애 영향

|영향|설명|
|--:|---|
|Socket/FD 점유|`LAST-ACK` 상태도 Socket 리소스를 점유함|
|신규 연결 지연|대량 누적 시 FD, ephemeral port, TCP 메모리 사용 증가|
|응답 지연 증가|종료 지연이 많으면 외부 API 호출 지연과 함께 나타날 수 있음|
|네트워크 장애 징후|패킷 손실, 방화벽/LB 문제, 상대 서버 비정상 종료 가능성|
|`Socket closed` 동반 가능|종료 중이거나 닫힌 Socket 접근 시 `SocketException` 로그와 함께 나타날 수 있음|

#### 5. 확인 커맨드

##### 5-1. 대상 검색 API의 `LAST-ACK` 개수 확인

```bash
ss -tan state last-ack | grep '192.168.100.50:8080' | wc -l
```

##### 5-2. 5초마다 `LAST-ACK` 변화 확인

```bash
watch -n 5 "ss -tan state last-ack | grep '192.168.100.50:8080' | wc -l"
```

##### 5-3. 대상 서버 상태별 개수 비교

```bash
watch -n 5 "ss -tan | grep '192.168.100.50:8080' | awk '{state[\$1]++} END {for (s in state) print s, state[s]}' | sort"
```

##### 5-4. 대상 연결 상세 확인

```bash
ss -tanp state last-ack | grep '192.168.100.50:8080'
```

##### 5-5. WAS 프로세스 기준 FD 사용량 확인

```bash
PID=<WAS_PID>
ls /proc/"$PID"/fd | wc -l
```

##### 5-6. WAS 프로세스의 TCP 연결 확인

```bash
PID=<WAS_PID>
lsof -Pan -p "$PID" -iTCP | grep '192.168.100.50:8080'
```

#### 6. 판단 기준

|현상|판단|
|--:|---|
|`LAST-ACK`가 순간적으로 보였다가 사라짐|정상 종료 과정일 가능성 큼|
|`LAST-ACK`가 계속 증가|상대 ACK 미수신, 네트워크/LB/상대 서버 문제 의심|
|`LAST-ACK`와 `Socket closed` 동시 증가|종료 중인 연결 접근, 검색 API/LB 연결 종료 문제 가능|
|`LAST-ACK`와 retransmission 증가|패킷 손실 또는 네트워크 품질 문제 가능|
|`LAST-ACK`와 FD 사용량 증가|Socket 리소스 점유로 WAS 안정성 저하 가능|

#### 7. 추가 확인 커맨드

```bash
# TCP 재전송/Reset/Timeout 통계 확인
netstat -s | grep -iE 'reset|retrans|timeout|failed'
```

```bash
# sysstat 설치 서버에서 TCP 통계 1초 간격 5회 확인
sar -n TCP,ETCP 1 5
```

```bash
# 대상 서버와의 패킷 흐름 확인, 운영에서는 짧게 수행
tcpdump -nn -i any host 192.168.100.50 and port 8080
```

#### 8. 실무 조치 방향

|우선|조치|설명|
|--:|---|---|
|1|대상 서버 기준 상태별 Socket 수 확인|`LAST-ACK`만 보지 말고 `ESTAB`, `TIME-WAIT`, `CLOSE-WAIT`와 비교|
|2|발생 시간대 로그 대조|`Socket closed`, `Connection reset`, `Read timed out`와 같은 시간대인지 확인|
|3|검색 API 서버 상태 확인|재기동, 배포, GC pause, Thread 고갈, Access log 지연 확인|
|4|L4/LB/방화벽 timeout 확인|Keep-Alive/idle timeout이 WAS·OkHttp timeout과 맞는지 확인|
|5|OkHttpClient 재사용|매 요청마다 Client 생성하지 않고 Connection Pool 재사용|
|6|Response close 보장|`try-with-resources`로 `Response` 정리|
|7|Timeout 명시|connect/read/write timeout을 업무 SLA 기준으로 설정|
|8|네트워크 품질 확인|retransmission, packet loss, ACK 지연 확인|

#### 9. 결론

`LAST-ACK`는 연결 종료의 마지막 ACK를 기다리는 상태이므로 **일시적으로 보이는 것은 정상**입니다. 하지만 특정 검색 API 서버 `192.168.100.50:8080` 대상으로 `LAST-ACK`가 많이 쌓이면, WAS가 연결을 닫으려 했지만 상대방 ACK가 오지 않는 상태이므로 **검색 API 서버, L4/LB, 방화벽, 네트워크 패킷 손실, keep-alive timeout 불일치**를 우선 의심해야 합니다. 이 상태가 지속되면 Socket/FD를 점유해 WAS의 신규 연결 처리와 외부 API 호출 안정성에 영향을 줄 수 있습니다.

