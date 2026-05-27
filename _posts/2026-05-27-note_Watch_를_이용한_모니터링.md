---
layout: single
title: "Watch_를_이용한_모니터링"
excerpt: "Watch_를_이용한_모니터링"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-05-27"
last_modified_at: "2026-05-27 14:12:01 +0900"
---
## 1. Watch 사용 모니터링 커멘드 분석
#### 1-1-1. Watch 두 커맨드 비교

##### 1-1-1-1. A안: 이전에 만든 커맨드

```bash
watch -n 3 "ss -Hant '( dport = :3306 )' | awk '{ state=\$1; peer=\$5; sub(/:[0-9]+$/, \"\", peer); print peer, state; }' | sort | uniq -c | sort -nr"
```

##### 1-1-1-2. B안: 추가로 제시한 커맨드

```bash
watch -n 3 "echo \$(date +'%Y-%m-%d %H:%M:%S') '[ESTABLISHED]:' \$(ss -Hant '( dport = :3306 )' | grep -c ESTAB) '[TIME_WAIT]:' \$(ss -Hant '( dport = :3306 )' | grep -c TIME-WAIT) "
```

#### 1-1-2. 장단점 비교

|구분|A안|B안|
|---|---|---|
|목적|DB 서버 IP별 + TCP 상태별 집계|전체 ESTAB/TIME-WAIT 개수 요약|
|장점|어느 DB 서버에 어떤 상태가 많은지 확인 가능|한눈에 현재 연결 수 확인 가능|
|단점|시간이 표시되지 않음|DB 서버별 구분 불가|
|`ss` 실행 횟수|1회|2회|
|성능|상대적으로 효율적|같은 조회를 두 번 수행|
|상태 범위|`ESTAB`, `TIME-WAIT`, `CLOSE-WAIT` 등 전체 상태 확인 가능|`ESTAB`, `TIME-WAIT`만 확인|
|장애 분석|특정 DB 서버 또는 특정 상태 편중 확인 가능|전체 Connection 증가/감소 추이만 확인 가능|
|실무 적합성|상세 분석에 유리|단순 감시에 유리|

#### 1-1-3. 핵심 차이

B안은 다음처럼 `ss`를 두 번 실행합니다.

```bash
ss -Hant '( dport = :3306 )' | grep -c ESTAB
ss -Hant '( dport = :3306 )' | grep -c TIME-WAIT
```

이 방식은 간단하지만, 3초마다 반복 실행할 경우 불필요하게 같은 TCP 목록을 두 번 조회합니다. 반면 A안은 `ss`를 한 번만 실행한 뒤 `awk`에서 상태와 DB 서버를 동시에 집계하므로 더 효율적입니다.

DB Connection 모니터링 목적이라면 단순히 `ESTAB` 총합만 보는 것보다 아래 항목을 같이 봐야 합니다.

```text
1. DB 서버별 ESTAB 수
2. DB 서버별 TIME-WAIT 수
3. DB 서버별 CLOSE-WAIT 수
4. 전체 ESTAB 합계
5. 측정 시각
```

특히 `CLOSE-WAIT`는 애플리케이션 또는 드라이버 쪽에서 소켓 정리가 지연될 때 의미가 있을 수 있으므로, `ESTAB`와 `TIME-WAIT`만 보면 장애 징후를 놓칠 수 있습니다.

#### 1-1-4. 최선의 권장 커맨드

아래 커맨드를 권장합니다.

```bash
watch -n 3 "date '+%Y-%m-%d %H:%M:%S'; ss -Hant '( dport = :3306 )' | awk '{ state=\$1; peer=\$5; sub(/:[0-9]+$/, \"\", peer); cnt[peer, state]++; total[peer]++; stateTotal[state]++; grand++; } END { printf \"%-18s %8s %10s %12s %10s %10s\\n\", \"DB_SERVER\", \"ESTAB\", \"TIME_WAIT\", \"CLOSE_WAIT\", \"SYN_SENT\", \"TOTAL\"; for (p in total) { printf \"%-18s %8d %10d %12d %10d %10d\\n\", p, cnt[p, \"ESTAB\"], cnt[p, \"TIME-WAIT\"], cnt[p, \"CLOSE-WAIT\"], cnt[p, \"SYN-SENT\"], total[p]; } printf \"%-18s %8d %10d %12d %10d %10d\\n\", \"TOTAL\", stateTotal[\"ESTAB\"], stateTotal[\"TIME-WAIT\"], stateTotal[\"CLOSE-WAIT\"], stateTotal[\"SYN-SENT\"], grand; }'"
```

#### 1-1-5. 출력 예시

```text
Every 3.0s: date '+%Y-%m-%d %H:%M:%S'; ss ...

2026-05-27 14:30:12
DB_SERVER             ESTAB  TIME_WAIT   CLOSE_WAIT   SYN_SENT      TOTAL
192.168.100.130          15          2            0          0         17
192.168.100.131           5          0            1          0          6
TOTAL                    20          2            1          0         23
```

#### 1-1-6. 이 커맨드가 더 나은 이유

|항목|개선 내용|
|---|---|
|조회 효율|`ss`를 1번만 실행|
|시간 확인|매 측정 시각 출력|
|DB별 확인|DB 서버 IP별 연결 상태 확인 가능|
|상태별 확인|`ESTAB`, `TIME-WAIT`, `CLOSE-WAIT`, `SYN-SENT` 동시 확인|
|전체 합계|마지막 줄에 전체 합계 출력|
|장애 분석|특정 DB 서버에 연결이 몰리는지 확인 가능|
|운영 활용|WAS에서 DB 연결 수 추이를 실시간으로 보기 적합|

#### 1-1-7. 상태 해석 기준

|상태|의미|DB Connection 관점|
|---|---|---|
|`ESTAB`|연결이 성립되어 있음|WAS ↔ DB 간 현재 유지 중인 TCP 연결|
|`TIME-WAIT`|연결 종료 후 대기|짧은 연결이 자주 생성/종료될 때 증가 가능|
|`CLOSE-WAIT`|상대가 종료했지만 로컬이 아직 닫지 않음|애플리케이션 또는 드라이버의 close 지연 의심 가능|
|`SYN-SENT`|연결 시도 중|DB 접속 지연, 네트워크 문제, DB 포트 응답 지연 가능|

#### 1-1-8. DBCP2 모니터링 시 주의

이 커맨드는 **OS TCP 연결 상태**를 보는 명령입니다. DBCP2 내부의 `active`, `idle`과 완전히 같은 값은 아닙니다.

```text
ESTAB = WAS와 DB 사이에 살아 있는 TCP 연결
DBCP2 Active = 애플리케이션 코드가 Pool에서 빌려 사용 중인 Connection
DBCP2 Idle = Pool 안에 반납되어 대기 중인 Connection
```

따라서 `ESTAB` 수는 보통 `DBCP2 active + idle`에 가까운 값으로 볼 수 있지만, 항상 1:1로 매핑되지는 않습니다.

#### 1-1-9. 실무 최종 권장

DB Connection 모니터링용 Linux 커맨드는 아래 1개를 기준으로 사용하면 됩니다.

```bash
watch -n 3 "date '+%Y-%m-%d %H:%M:%S'; ss -Hant '( dport = :3306 )' | awk '{ state=\$1; peer=\$5; sub(/:[0-9]+$/, \"\", peer); cnt[peer, state]++; total[peer]++; stateTotal[state]++; grand++; } END { printf \"%-18s %8s %10s %12s %10s %10s\\n\", \"DB_SERVER\", \"ESTAB\", \"TIME_WAIT\", \"CLOSE_WAIT\", \"SYN_SENT\", \"TOTAL\"; for (p in total) { printf \"%-18s %8d %10d %12d %10d %10d\\n\", p, cnt[p, \"ESTAB\"], cnt[p, \"TIME-WAIT\"], cnt[p, \"CLOSE-WAIT\"], cnt[p, \"SYN-SENT\"], total[p]; } printf \"%-18s %8d %10d %12d %10d %10d\\n\", \"TOTAL\", stateTotal[\"ESTAB\"], stateTotal[\"TIME-WAIT\"], stateTotal[\"CLOSE-WAIT\"], stateTotal[\"SYN-SENT\"], grand; }'"
```

운영 분석에서는 이 커맨드로 **WAS → DB TCP 연결 상태**를 보고, 동시에 DBCP2의 `numActive`, `numIdle`, `maxTotal`을 JMX 또는 Micrometer로 함께 확인하는 방식이 가장 정확합니다.

### 1-2. 참고

#### 1-2-1. IPv4 주소로 정규화한 수정 커맨드

```bash
watch -n 3 "date '+%Y-%m-%d %H:%M:%S'; ss -Hant '( dport = :3306 )' | awk '{ state=\$1; peer=\$5; sub(/:[0-9]+$/, \"\", peer); gsub(/^\\[::ffff:/, \"\", peer); gsub(/^\\[/, \"\", peer); gsub(/\\]$/, \"\", peer); cnt[peer, state]++; total[peer]++; stateTotal[state]++; grand++; } END { printf \"%-18s %8s %10s %12s %10s %10s\\n\", \"DB_SERVER\", \"ESTAB\", \"TIME_WAIT\", \"CLOSE_WAIT\", \"SYN_SENT\", \"TOTAL\"; for (p in total) { printf \"%-18s %8d %10d %12d %10d %10d\\n\", p, cnt[p, \"ESTAB\"], cnt[p, \"TIME-WAIT\"], cnt[p, \"CLOSE-WAIT\"], cnt[p, \"SYN-SENT\"], total[p]; } printf \"%-18s %8d %10d %12d %10d %10d\\n\", \"TOTAL\", stateTotal[\"ESTAB\"], stateTotal[\"TIME-WAIT\"], stateTotal[\"CLOSE-WAIT\"], stateTotal[\"SYN-SENT\"], grand; }'"
```

#### 1-2-2. 변경된 부분

기존 명령의 `awk` 내부에 아래 3줄을 추가한 것입니다.

```awk
gsub(/^\[::ffff:/, "", peer);
gsub(/^\[/, "", peer);
gsub(/\]$/, "", peer);
```

#### 1-2-3. 변환 결과

```text
[::ffff:192.168.111.222] → 192.168.111.222
```

#### 1-2-4. 처리 순서

|순서|처리|예시 결과|
|---|---|---|
|1|포트 제거|`[::ffff:192.168.111.222]:3306` → `[::ffff:192.168.111.222]`|
|2|IPv4-mapped IPv6 prefix 제거|`[::ffff:192.168.111.222]` → `192.168.111.222]`|
|3|끝 대괄호 제거|`192.168.111.222]` → `192.168.111.222`|

#### 1-2-5. 출력 예시

```text
2026-05-27 15:30:00
DB_SERVER             ESTAB  TIME_WAIT   CLOSE_WAIT   SYN_SENT      TOTAL
192.168.111.222          10          2            0          0         12
192.168.111.223           5          0            0          0          5
TOTAL                    15          2            0          0         17
```

#### 1-2-6. 참고

`ss -4`를 쓰면 IPv4만 조회할 수는 있지만, 현재처럼 `[::ffff:192.168.111.222]` 형태로 표시되는 연결이 누락될 수 있습니다. 따라서 DB Connection 모니터링 용도에서는 **`ss` 결과를 awk에서 IPv4 형식으로 정규화하는 현재 방식이 더 안전**합니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
