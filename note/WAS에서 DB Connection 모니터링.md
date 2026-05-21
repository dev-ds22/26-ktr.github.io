## 모니터링 전제

WAS 서버에 SSH로 접속한 상태에서 실행한다고 보면 됩니다. WAS → DB로 나가는 연결은 보통 `Peer Address:Port`가 `DB서버IP:3306` 형태로 표시됩니다.

## 1. 현재 WAS 서버에서 DB 서버별/상태별 연결 수 즉시 확인

```bash
ss -Hant '( dport = :3306 )' \
| awk '{
    state=$1;
    peer=$5;
    sub(/:[0-9]+$/, "", peer);
    print peer, state;
  }' \
| sort \
| uniq -c \
| sort -nr
```

출력 예:

```text
25 192.168.10.11 ESTAB
3  192.168.10.12 TIME-WAIT
1  192.168.10.13 CLOSE-WAIT
```

의미:

|항목|의미|
|---|---|
|첫 번째 숫자|연결 수|
|IP|대상 DB 서버|
|상태|TCP 연결 상태|

## 2. 특정 DB 서버들만 확인

예를 들어 DB 서버가 아래 3대라고 가정합니다.

```text
192.168.10.11
192.168.10.12
192.168.10.13
```

명령:

```bash
ss -Hant '( dport = :3306 )' \
| awk '{
    state=$1;
    peer=$5;
    sub(/:[0-9]+$/, "", peer);
    print peer, state;
  }' \
| grep -E '^(192\.168\.10\.11|192\.168\.10\.12|192\.168\.10\.13) ' \
| sort \
| uniq -c \
| sort -nr
```

## 3. 5초 간격으로 날짜가 붙은 파일에 저장하는 스크립트

### 3-1. 스크립트 생성

```bash
vi db-connection-monitoring.sh
```

### 3-2. 스크립트 내용

아래에서 `DB_SERVERS`만 실제 DB 서버 IP로 바꾸면 됩니다.

```bash
#!/bin/bash
DB_PORT=3306
DB_SERVERS="192.168.10.11 192.168.10.12 192.168.10.13"
OUT_FILE="./db-connection-monitoring_$(date '+%Y%m%d').log"
echo "execution_time,target_db_server,connection_state,connection_count" >> "$OUT_FILE"
while true
do
  NOW=$(date '+%Y-%m-%d %H:%M:%S')
  ss -Hant "( dport = :$DB_PORT )" \
  | awk -v now="$NOW" -v dbs="$DB_SERVERS" '
    BEGIN {
      db_count = split(dbs, db_arr, " ");
      state_order[1] = "ESTAB";
      state_order[2] = "SYN-SENT";
      state_order[3] = "TIME-WAIT";
      state_order[4] = "CLOSE-WAIT";
      state_order[5] = "FIN-WAIT-1";
      state_order[6] = "FIN-WAIT-2";
      state_order[7] = "LAST-ACK";
      state_order[8] = "CLOSING";
      for (i = 1; i <= db_count; i++) {
        target_db[db_arr[i]] = 1;
      }
    }
    {
      state = $1;
      peer = $5;
      sub(/^\[/, "", peer);
      sub(/\]:[0-9]+$/, "", peer);
      sub(/:[0-9]+$/, "", peer);
      if (peer in target_db) {
        conn_count[peer, state]++;
      }
    }
    END {
      for (i = 1; i <= db_count; i++) {
        db = db_arr[i];
        printed = 0;
        for (j = 1; j <= 8; j++) {
          state = state_order[j];
          count = conn_count[db, state] + 0;
          if (count > 0) {
            print now "," db "," state "," count;
            printed = 1;
          }
        }
        if (printed == 0) {
          print now "," db ",NO_CONNECTION,0";
        }
      }
    }
  ' >> "$OUT_FILE" 2>&1
  sleep 5
done
```

## 4. 실행 방법

```bash
chmod +x db-connection-monitoring.sh
./db-connection-monitoring.sh
```

이 방식은 터미널을 점유합니다. 중지는 `Ctrl + C`입니다.

## 5. 백그라운드 실행

```bash
nohup ./db-connection-monitoring.sh > /dev/null 2>&1 &
```

실행 확인:

```bash
ps -ef | grep db-connection-monitoring.sh | grep -v grep
```

중지:

```bash
ps -ef | grep db-connection-monitoring.sh | grep -v grep
kill [PID]
```

## 6. 결과 파일 확인

파일명 예:

```text
db-connection-monitoring_20260521_163000.log
```

실시간 확인:

```bash
tail -F db-connection-monitoring_*.log
```

출력 예:

```csv
execution_time,target_db_server,connection_state,connection_count
2026-05-21 16:30:00,192.168.10.11,ESTAB,25
2026-05-21 16:30:00,192.168.10.12,TIME-WAIT,3
2026-05-21 16:30:00,192.168.10.13,NO_CONNECTION,0
2026-05-21 16:30:05,192.168.10.11,ESTAB,26
2026-05-21 16:30:05,192.168.10.12,ESTAB,2
```

## 7. 단순 버전

DB 서버 목록을 고정하지 않고, 현재 연결된 모든 `:3306` 대상만 저장하려면 아래가 더 간단합니다.

```bash
OUT_FILE="./db-connection-monitoring_$(date '+%Y%m%d_%H%M%S').log"
echo "execution_time,target_db_server,connection_state,connection_count" >> "$OUT_FILE"
while true
do
  NOW=$(date '+%Y-%m-%d %H:%M:%S')
  ss -Hant '( dport = :3306 )' \
  | awk -v now="$NOW" '{
      state=$1;
      peer=$5;
      sub(/:[0-9]+$/, "", peer);
      print peer, state;
    }' \
  | sort \
  | uniq -c \
  | awk -v now="$NOW" '{print now "," $2 "," $3 "," $1}' \
  >> "$OUT_FILE" 2>&1
  sleep 5
done
```

정확도: 97%

## 전제

여기서 말하는 `DB connection 상태`는 **DBCP/Hikari 같은 JDBC Connection Pool의 Active/Idle 상태가 아니라**, WAS 서버에서 `ss` 명령으로 확인되는 **TCP 연결 상태**입니다.  
즉, WAS → DB서버 `:3306` 으로 맺어진 **네트워크 소켓 상태**입니다.

## 상태별 의미

|상태|의미|운영 관점 판단|
|---|---|---|
|`ESTAB`|연결이 정상적으로 수립된 상태|정상 상태. WAS와 DB 사이 TCP 연결이 살아 있음|
|`SYN-SENT`|WAS가 DB로 연결 요청을 보냈지만 아직 연결 완료 전|짧게 보이면 정상. 지속되면 DB 접속 지연, 방화벽, 네트워크, DB 포트 문제 의심|
|`TIME-WAIT`|연결 종료 후 일정 시간 동안 남아 있는 상태|일반적으로 정상. 너무 많으면 DB 연결 생성/종료가 과도한 구조일 수 있음|
|`CLOSE-WAIT`|상대방이 연결 종료를 요청했고, 로컬 애플리케이션이 아직 close하지 않은 상태|주의. 계속 증가하면 WAS 애플리케이션의 Connection/Socket/Response close 누락 의심|
|`FIN-WAIT-1`|WAS가 연결 종료 요청을 보낸 직후 상태|짧게 보이면 정상. 오래 지속되면 상대 응답 지연 가능|
|`FIN-WAIT-2`|WAS의 종료 요청에 대해 상대가 ACK 했고, 상대의 종료 FIN을 기다리는 상태|짧게 보이면 정상. 오래 지속되면 상대방 종료 지연 가능|
|`LAST-ACK`|상대가 먼저 종료했고, 로컬도 종료 응답 후 마지막 ACK를 기다리는 상태|짧게 보이면 정상. 지속되면 종료 흐름 문제 가능|
|`CLOSING`|양쪽이 거의 동시에 종료 요청을 보낸 상태|드문 상태. 일시적이면 문제 아님|

## 상태별 상세 설명

### 1. `ESTAB`

`ESTAB`는 `ESTABLISHED`의 축약 표현입니다. TCP 연결이 정상적으로 맺어진 상태입니다.  
WAS에서 DB `:3306`으로 열린 커넥션이 `ESTAB`이면 네트워크 레벨에서는 연결이 살아 있는 것입니다.  
DBCP 같은 커넥션 풀에서 **Idle 상태인 JDBC Connection도 TCP 기준으로는 `ESTAB`**로 보일 수 있습니다.

```text
WAS Connection Pool Idle Connection
→ TCP 기준: ESTAB
→ DB SHOW PROCESSLIST 기준: Sleep
```

따라서 `ESTAB` 수가 많다고 해서 모두 실행 중인 SQL은 아닙니다.

## 2. `SYN-SENT`

WAS가 DB 서버에 TCP 연결을 시도한 상태입니다.

```text
WAS → DB: 연결 요청 SYN 전송
DB 응답 대기 중
```

짧게 지나가는 것은 정상입니다. 하지만 계속 쌓이면 문제가 될 수 있습니다.  
주요 원인:

```text
DB 서버 다운
DB 3306 포트 미기동
방화벽 차단
네트워크 지연
L4/보안장비 문제
DB max_connections 초과로 연결 지연
```

확인 명령:

```bash
ss -Hant state syn-sent '( dport = :3306 )'
```

## 3. `TIME-WAIT`

이미 종료된 연결이 OS에 잠시 남아 있는 상태입니다.

```text
연결은 종료됨
단, 지연 패킷 처리를 위해 OS가 일정 시간 보관
```

일반적으로 장애는 아닙니다. 다만 `TIME-WAIT`가 매우 많다면 DB 연결을 자주 만들고 닫는 구조일 수 있습니다.  
의심 상황:

```text
Connection Pool 미사용
매 요청마다 DB 연결 생성/종료
Pool 설정이 너무 작거나 유휴 커넥션을 너무 자주 제거
짧은 시간에 배치/대량 요청 발생
```

확인 명령:

```bash
ss -Hant state time-wait '( dport = :3306 )' | wc -l
```

## 4. `CLOSE-WAIT`

상대방, 즉 DB 서버 또는 중간 네트워크 장비가 먼저 연결 종료를 요청했는데, WAS 애플리케이션이 아직 해당 소켓을 닫지 않은 상태입니다.

```text
DB/L4/네트워크 장비 → 종료 요청
WAS OS는 받음
하지만 Java 애플리케이션이 close 처리 미완료
```

이 상태가 계속 증가하면 중요합니다.  
주요 원인:

```text
JDBC Connection close 누락
ResultSet/Statement/Connection 정리 누락
HTTP Client Response close 누락
Socket/Stream close 누락
커넥션 풀에서 비정상 커넥션 회수 실패
```

확인 명령:

```bash
ss -Hantp state close-wait '( dport = :3306 )'
```

주의:

```text
CLOSE-WAIT는 TIME-WAIT보다 훨씬 위험 신호입니다.
일시적이면 괜찮지만, 줄지 않고 계속 증가하면 애플리케이션 리소스 누수 가능성이 큽니다.
```

## 5. `FIN-WAIT-1`

WAS 쪽에서 먼저 연결 종료를 요청한 직후 상태입니다.

```text
WAS → DB: FIN 전송
DB의 ACK 대기 중
```

짧게 보이는 것은 정상입니다.  
지속되면 다음을 의심할 수 있습니다.

```text
DB 서버 응답 지연
네트워크 지연
패킷 손실
중간 방화벽/L4 장비 문제
```

## 6. `FIN-WAIT-2`

WAS가 종료 요청을 보냈고, DB가 그 요청에 대한 ACK는 보낸 상태입니다. 이제 WAS는 DB 쪽의 최종 종료 요청을 기다립니다.

```text
WAS → DB: FIN
DB → WAS: ACK
WAS는 DB의 FIN 대기 중
```

짧게 보이면 정상입니다.  
오래 지속되면 상대방이 연결 종료를 완전히 마무리하지 않는 상황일 수 있습니다.

## 7. `LAST-ACK`

상대방이 먼저 종료 요청을 했고, 로컬도 종료 처리를 한 뒤 마지막 ACK를 기다리는 상태입니다.

```text
DB → WAS: FIN
WAS → DB: FIN
WAS는 마지막 ACK 대기
```

일시적으로 보이는 것은 정상입니다. 지속적으로 많이 쌓이면 네트워크 지연 또는 종료 처리 문제를 의심할 수 있습니다.

## 8. `CLOSING`

양쪽이 거의 동시에 종료 요청을 보낸 상태입니다.

```text
WAS와 DB가 동시에 FIN 전송
```

운영 중 흔하게 많이 보이는 상태는 아닙니다. 순간적으로 보이면 큰 문제는 아닙니다.

## 운영 판단 기준

|상태|정상 여부|조치 필요성|
|---|---|---|
|`ESTAB`|대체로 정상|개수가 Pool 설정과 맞는지 확인|
|`SYN-SENT`|일시적 정상|지속되면 DB 접속/네트워크 점검|
|`TIME-WAIT`|대체로 정상|과다하면 연결 생성/종료 빈도 점검|
|`CLOSE-WAIT`|주의|증가하면 close 누락/리소스 누수 점검|
|`FIN-WAIT-1`|일시적 정상|지속되면 네트워크/DB 응답 점검|
|`FIN-WAIT-2`|일시적 정상|지속되면 상대 종료 지연 점검|
|`LAST-ACK`|일시적 정상|지속되면 종료 흐름 점검|
|`CLOSING`|일시적 정상|많으면 네트워크 종료 흐름 점검|

## 실무에서 특히 봐야 하는 상태

### 가장 중요한 상태

```text
1. ESTAB
2. CLOSE-WAIT
3. SYN-SENT
4. TIME-WAIT
```

### 판단 요령

```text
ESTAB 많음
→ 실제 DB 커넥션 풀 크기, DB max_connections, SHOW PROCESSLIST Sleep 수와 비교
CLOSE-WAIT 증가
→ WAS 애플리케이션 close 누락 가능성 우선 점검
SYN-SENT 지속
→ DB 접속 자체가 안 되거나 지연되는 상황
TIME-WAIT 과다
→ DB 연결을 너무 자주 만들고 끊는 구조 의심
```

## DB Connection Pool 관점과 TCP 상태 차이

|구분|의미|확인 위치|
|---|---|---|
|TCP `ESTAB`|네트워크 연결이 살아 있음|WAS 서버 `ss`|
|DB `Sleep`|DB에 접속은 되어 있으나 쿼리 실행 중 아님|DB `SHOW PROCESSLIST`|
|Pool `Idle`|커넥션 풀 안에서 대기 중|WAS Pool 모니터링|
|Pool `Active`|애플리케이션이 사용 중|WAS Pool 모니터링|
|TCP `CLOSE-WAIT`|상대가 끊었지만 WAS가 아직 안 닫음|WAS 서버 `ss`|

## 권장 확인 명령

### DB 연결 상태별 전체 집계

```bash
ss -Hant '( dport = :3306 )' | awk '{print $1}' | sort | uniq -c | sort -nr
```

### DB 서버별/상태별 집계

```bash
ss -Hant '( dport = :3306 )' \
| awk '{
    state=$1;
    peer=$5;
    sub(/:[0-9]+$/, "", peer);
    print peer, state;
  }' \
| sort \
| uniq -c \
| sort -nr
```

### `CLOSE-WAIT` 상세 확인

```bash
ss -Hantp state close-wait '( dport = :3306 )'
```

### `SYN-SENT` 상세 확인

```bash
ss -Hantp state syn-sent '( dport = :3306 )'
```

## 결론

DB 연결 모니터링에서 가장 중요하게 봐야 할 것은 아래입니다.

```text
ESTAB      : 현재 살아 있는 DB TCP 연결 수
CLOSE-WAIT : 애플리케이션 close 누락 가능성
SYN-SENT   : DB 접속 지연/실패 가능성
TIME-WAIT  : 연결 생성/종료 과다 가능성
```

특히 운영 장애 분석에서는 `CLOSE-WAIT`가 지속 증가하는지, `SYN-SENT`가 남아 있는지, `ESTAB` 수가 커넥션 풀 설정과 맞는지를 우선 확인하는 것이 좋습니다.