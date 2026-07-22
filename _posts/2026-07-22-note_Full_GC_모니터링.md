---
layout: single
title: "Full_GC_모니터링"
excerpt: "Full_GC_모니터링"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-07-22"
last_modified_at: "2026-07-22 09:49:24 +0900"
mermaid: false
---
### 1. 결론

JBoss 7에서 Full GC를 확인하는 방법은 크게 두 가지입니다.

| 목적                         | 방법                     |         과거 확인 가능 여부 |
| -------------------------- | ---------------------- | ------------------: |
| 현재 JVM의 Full GC 횟수·누적시간 확인 | `jstat`, `jcmd`        |   JVM 기동 이후 누적값만 가능 |
| Full GC 발생 시각·원인·정지시간 확인   | GC 로그 파일               | 로그가 사전에 활성화된 경우만 가능 |
| 과거 특정 일자의 시스템 영향 확인        | GC 로그＋서버 로그＋OS 모니터링 로그 |      자료가 남아 있을 때 가능 |

**중요한 점은 GC 로그를 당시 저장하지 않았다면, 과거 특정 시각의 Full GC 발생 여부와 정지시간을 정확하게 복원할 수 없다는 것입니다.** `jstat`은 현재 실행 중인 JVM의 누적 통계만 보여 줍니다.

JDK 11부터 GC 로그는 기존 `-XX:+PrintGCDetails`, `-Xloggc`보다 통합 로깅 옵션인 `-Xlog` 사용이 권장됩니다. ([Oracle Documentation][1])

### 2. JBoss JVM 프로세스 확인

먼저 JBoss 프로세스와 실제 사용 중인 Java 버전을 확인합니다.

```bash
ps -ef | grep -E '[j]boss|[s]tandalone|[d]omain'
```

보다 정확하게 확인하려면:

```bash
pgrep -af 'org.jboss.as.standalone|org.jboss.as.host-controller|jboss-modules.jar'
```

예시:

```text
jboss  12345  ... /usr/lib/jvm/java-11/bin/java ... org.jboss.as.standalone
```

PID를 변수로 지정합니다.

```bash
JBOSS_PID=$(pgrep -f 'org.jboss.as.standalone' | head -1)
echo "$JBOSS_PID"
```

프로세스가 여러 개인 서버에서는 반드시 명령행을 확인한 후 PID를 직접 지정하는 것이 안전합니다.

```bash
JBOSS_PID=12345
```

실행 중인 JVM 버전 확인:

```bash
readlink -f /proc/$JBOSS_PID/exe
```

```bash
sudo -u jboss /proc/$JBOSS_PID/exe -version
```

JVM 옵션 확인:

```bash
tr '\0' ' ' < /proc/$JBOSS_PID/cmdline
```

또는:

```bash
jcmd $JBOSS_PID VM.command_line
jcmd $JBOSS_PID VM.flags
```

> `jstat`, `jcmd`, `jstack`은 가급적 JBoss 프로세스를 실행한 동일한 OS 계정으로 실행해야 합니다.

```bash
sudo -u jboss jcmd $JBOSS_PID VM.flags
```

### 3. 현재 Full GC 발생 횟수와 누적시간 확인

가장 간단한 방법은 `jstat -gcutil`입니다. `jstat`은 실행 중인 HotSpot JVM의 GC 통계를 표시합니다. ([Oracle Documentation][2])

```bash
sudo -u jboss jstat -gcutil $JBOSS_PID 1000 10
```

* `1000`: 1초 간격
* `10`: 10회 출력

예시:

```text
  S0    S1     E      O      M    CCS    YGC   YGCT   FGC   FGCT    CGC   CGCT    GCT
  0.0  25.0   71.3   82.4   95.1  92.3  1421  31.2    7    12.8     0    0.0    44.0
```

주요 항목:

| 항목     | 의미                  |
| ------ | ------------------- |
| `E`    | Eden 영역 사용률         |
| `O`    | Old 영역 사용률          |
| `M`    | Metaspace 사용률       |
| `YGC`  | Young GC 누적 횟수      |
| `YGCT` | Young GC 누적 소요시간(초) |
| `FGC`  | Full GC 누적 횟수       |
| `FGCT` | Full GC 누적 소요시간(초)  |
| `GCT`  | 전체 GC 누적 소요시간(초)    |

예를 들어:

```text
FGC = 7
FGCT = 12.8
```

이라면 JVM 기동 이후 Full GC가 7회 발생했고, Full GC에 총 12.8초가 소요됐다는 의미입니다.

평균 Full GC 시간은 대략 다음과 같습니다.

```text
12.8초 ÷ 7회 ≒ 1.83초/회
```

단, `jstat` 값만으로는 각 Full GC가 정확히 언제 발생했는지 알 수 없습니다.

### 4. 실시간으로 Full GC 증가 여부 감시

#### 4-1. 5초 간격 모니터링

```bash
sudo -u jboss jstat -gcutil $JBOSS_PID 5000
```

중지:

```text
Ctrl+C
```

`FGC` 값이 증가하면 해당 구간에 Full GC가 발생한 것입니다.

#### 4-2. Full GC 관련 항목만 보기

```bash
sudo -u jboss jstat -gcutil $JBOSS_PID 5000 |
awk '
NR == 1 {
    print $0
    next
}
{
    print strftime("%Y-%m-%d %H:%M:%S"), $0
}'
```

다만 헤더와 데이터 정렬이 다소 불편할 수 있으므로 다음 스크립트가 실무적으로 더 명확합니다.

```bash
#!/bin/bash

PID="$1"

if [ -z "$PID" ]; then
    echo "Usage: $0 <JBOSS_PID>"
    exit 1
fi

PREV_FGC=""

while true
do
    VALUE=$(jstat -gcutil "$PID" 2>/dev/null | tail -1)

    if [ -z "$VALUE" ]; then
        echo "$(date '+%F %T') JVM 조회 실패: PID=$PID"
        exit 1
    fi

    YGC=$(echo "$VALUE" | awk '{print $7}')
    YGCT=$(echo "$VALUE" | awk '{print $8}')
    FGC=$(echo "$VALUE" | awk '{print $9}')
    FGCT=$(echo "$VALUE" | awk '{print $10}')
    GCT=$(echo "$VALUE" | awk '{print $13}')

    if [ -n "$PREV_FGC" ] && [ "$FGC" -gt "$PREV_FGC" ]; then
        echo "$(date '+%F %T') [FULL GC 증가] FGC=$FGC FGCT=${FGCT}s GCT=${GCT}s"
    else
        echo "$(date '+%F %T') YGC=$YGC YGCT=${YGCT}s FGC=$FGC FGCT=${FGCT}s GCT=${GCT}s"
    fi

    PREV_FGC="$FGC"
    sleep 5
done
```

저장:

```bash
vi /opt/scripts/monitor_jboss_gc.sh
chmod 750 /opt/scripts/monitor_jboss_gc.sh
```

실행:

```bash
sudo -u jboss /opt/scripts/monitor_jboss_gc.sh $JBOSS_PID
```

파일로 저장:

```bash
sudo -u jboss nohup /opt/scripts/monitor_jboss_gc.sh $JBOSS_PID \
  >> /var/log/jboss/gc-monitor.log 2>&1 &
```

이 방식은 모니터링 스크립트를 실행한 시점부터 기록합니다.

### 5. 현재 JVM의 GC 원인 확인

```bash
sudo -u jboss jstat -gccause $JBOSS_PID 1000 10
```

예시:

```text
  S0   S1    E     O     M    CCS   YGC  YGCT  FGC  FGCT  CGC  CGCT  GCT  LGCC      GCC
  0.0  0.0  45.0  78.1  94.0 91.2  520  12.1    3   4.5    0   0.0 16.6 Allocation Failure No GC
```

주요 항목:

| 항목                        | 의미                      |
| ------------------------- | ----------------------- |
| `LGCC`                    | 마지막 GC 원인               |
| `GCC`                     | 현재 GC 원인                |
| `Allocation Failure`      | 객체를 할당할 공간이 부족해 GC 실행   |
| `Metadata GC Threshold`   | Metaspace 임계치 도달        |
| `System.gc()`             | 코드 또는 라이브러리에서 명시적 GC 요청 |
| `G1 Evacuation Pause`     | G1 GC 영역 회수             |
| `G1 Humongous Allocation` | 대용량 객체 할당으로 GC 유발       |

`jstat -gccause`도 발생 시각이나 상세 pause time은 제공하지 않으므로 GC 로그와 함께 봐야 합니다.

### 6. 현재 Heap 상태 확인

```bash
sudo -u jboss jcmd $JBOSS_PID GC.heap_info
```

예시:

```text
garbage-first heap
 total 4194304K, used 3512000K
 region size 2048K
 Metaspace used 412000K, committed 430000K, reserved 1500000K
```

클래스 히스토그램:

```bash
sudo -u jboss jcmd $JBOSS_PID GC.class_histogram
```

> `GC.class_histogram`은 운영 JVM에 부하를 줄 수 있으므로 장애 중 반복 실행하지 않는 것이 좋습니다.

JVM 가동 시간:

```bash
sudo -u jboss jcmd $JBOSS_PID VM.uptime
```

`jstat`의 누적 Full GC 수치는 이 JVM 가동 시간 동안의 값입니다. JBoss가 재시작되면 다시 0부터 계산됩니다.

### 7. 현재 GC 로그 설정 여부 확인

JVM 명령행에서 GC 로그 옵션을 확인합니다.

```bash
tr '\0' '\n' < /proc/$JBOSS_PID/cmdline |
grep -E 'Xlog|Xloggc|PrintGC|UseGCLogFileRotation'
```

또는:

```bash
sudo -u jboss jcmd $JBOSS_PID VM.command_line |
grep -E 'Xlog|Xloggc|PrintGC'
```

JDK 11 설정 예:

```text
-Xlog:gc*:file=/app/jboss/standalone/log/gc.log:time,uptime,level,tags:filecount=10,filesize=100M
```

기존 JDK 8 형식 예:

```text
-Xloggc:/app/jboss/standalone/log/gc.log
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
```

JDK 11에서는 기존 GC 옵션 일부가 deprecated되었으므로 `-Xlog` 형식을 사용하는 것이 적절합니다. ([Oracle Documentation][3])

### 8. JDK 11 권장 GC 로그 설정

#### 8-1. Standalone 방식

설정 파일:

```text
$JBOSS_HOME/bin/standalone.conf
```

기존 `JAVA_OPTS` 뒤에 다음을 추가합니다.

```bash
GC_LOG_DIR="/app/jboss/standalone/log"

JAVA_OPTS="$JAVA_OPTS \
-Xlog:gc*,safepoint:file=${GC_LOG_DIR}/gc.log:time,uptime,level,tags:filecount=10,filesize=100M"
```

한 줄 형식:

```bash
JAVA_OPTS="$JAVA_OPTS -Xlog:gc*,safepoint:file=/app/jboss/standalone/log/gc.log:time,uptime,level,tags:filecount=10,filesize=100M"
```

옵션 의미:

| 옵션              | 의미                 |
| --------------- | ------------------ |
| `gc*`           | GC 관련 전체 하위 태그 포함  |
| `safepoint`     | JVM 정지 지점과 정지시간 기록 |
| `file=...`      | 로그 파일 경로           |
| `time`          | 실제 날짜·시각           |
| `uptime`        | JVM 시작 후 경과시간      |
| `level`         | 로그 레벨              |
| `tags`          | GC 로그 태그           |
| `filecount=10`  | 최대 10개 순환 파일       |
| `filesize=100M` | 파일당 최대 100MB       |

Oracle 문서도 `-Xlog:gc*:file=...:filecount=...,filesize=...` 형태의 파일 순환 설정을 지원합니다. ([Oracle Documentation][1])

Red Hat 문서에 따르면 standalone 서버의 JVM 설정은 `standalone.conf`의 `JAVA_OPTS`에 추가할 수 있습니다. ([Red Hat Customer Portal][4])

#### 8-2. 권장 디렉터리 생성

```bash
mkdir -p /app/jboss/standalone/log
chown jboss:jboss /app/jboss/standalone/log
chmod 750 /app/jboss/standalone/log
```

#### 8-3. 설정 전 백업

```bash
cp -p "$JBOSS_HOME/bin/standalone.conf" \
      "$JBOSS_HOME/bin/standalone.conf.$(date +%Y%m%d_%H%M%S).bak"
```

#### 8-4. 적용 시 주의

`standalone.conf` 변경은 일반적으로 JBoss 재기동 후 적용됩니다.

재기동 전 옵션 문법 확인:

```bash
$JAVA_HOME/bin/java \
'-Xlog:gc*,safepoint:file=/tmp/gc-test.log:time,uptime,level,tags:filecount=2,filesize=10M' \
-version
```

정상이라면 `/tmp/gc-test.log`가 생성됩니다.

### 9. Domain 방식 설정 위치

JBoss가 Managed Domain 방식이면 각 서버 JVM 설정은 일반적으로 다음에서 관리합니다.

```text
$JBOSS_HOME/domain/configuration/domain.xml
$JBOSS_HOME/domain/configuration/host.xml
```

Red Hat 문서 기준으로 Managed Domain에서는 host, server group 또는 개별 server 수준에 JVM 설정을 선언할 수 있습니다. ([Red Hat Customer Portal][4])

CLI 예시는 환경별 서버 그룹과 JVM 이름이 다르므로 먼저 조회합니다.

```bash
$JBOSS_HOME/bin/jboss-cli.sh --connect
```

```text
/server-group=*:read-resource
/server-group=<SERVER_GROUP>/jvm=*:read-resource
```

운영 환경에서는 `domain.xml`을 직접 수정하기보다 관리 CLI 또는 관리 콘솔을 통한 변경이 안전합니다.

### 10. 재기동 없이 지금부터 GC 로그 시작

JDK 11의 통합 로깅은 `jcmd VM.log`를 통해 실행 중 변경이 가능한 경우가 있습니다.

지원 여부 확인:

```bash
sudo -u jboss jcmd $JBOSS_PID help VM.log
```

현재부터 GC 로그 활성화 예:

```bash
sudo -u jboss jcmd $JBOSS_PID VM.log \
  output=/app/jboss/standalone/log/gc-runtime.log \
  what='gc*=info,safepoint=info' \
  decorators='time,uptime,level,tags'
```

확인:

```bash
tail -f /app/jboss/standalone/log/gc-runtime.log
```

다만 이 방식은 다음 제약이 있습니다.

* 명령 실행 이후의 로그만 기록
* JBoss 재기동 시 유지되지 않을 수 있음
* 영구 설정은 `standalone.conf` 등에 적용해야 함
* JVM 배포판에 따라 `VM.log` 세부 지원이 다를 수 있음

따라서 운영 상시 모니터링은 시작 옵션에 `-Xlog`를 넣는 방식이 적절합니다.

### 11. 이전 일자의 GC 로그 찾기

#### 11-1. JBoss 기본 로그 디렉터리 검색

```bash
find "$JBOSS_HOME" -type f \
  \( -iname '*gc*.log*' -o -iname 'gc.log*' \) \
  -ls 2>/dev/null
```

자주 사용하는 위치:

```text
$JBOSS_HOME/standalone/log/
$JBOSS_HOME/domain/servers/<SERVER_NAME>/log/
$JBOSS_HOME/log/
```

일부 JBoss EAP 구성에서는 기본 GC 로그가 다음과 같은 순환 파일 이름을 사용할 수 있습니다. ([Red Hat Customer Portal][5])

```text
gc.log
gc.log.0
gc.log.1
gc.log.0.current
gc.log.<숫자>.current
```

전체 서버에서 검색:

```bash
find /app /opt /var/log -type f \
  \( -iname 'gc.log*' -o -iname '*gc*.log*' \) \
  2>/dev/null
```

#### 11-2. 파일 수정일 기준 검색

2026년 7월 16일 관련 파일:

```bash
find "$JBOSS_HOME" -type f \
  \( -iname 'gc.log*' -o -iname '*gc*.log*' \) \
  -newermt '2026-07-16 00:00:00' \
  ! -newermt '2026-07-17 00:00:00' \
  -printf '%TY-%Tm-%Td %TH:%TM:%TS %p\n'
```

주의할 점은 GC 로그 순환 파일의 수정일이 로그 내부 이벤트 날짜와 정확히 일치하지 않을 수 있다는 것입니다. 최종 판단은 파일 내용의 timestamp로 해야 합니다.

### 12. 이전 일자의 Full GC 검색

#### 12-1. 일반 검색

```bash
grep -iE 'Full GC|Pause Full|System\.gc|Metadata GC Threshold' \
  "$JBOSS_HOME"/standalone/log/gc.log*
```

JDK 11 G1 GC의 대표적인 Full GC 로그:

```text
[2026-07-16T03:18:44.120+0900][123456.789s][info][gc,start] GC(731) Pause Full (G1 Compaction Pause)
[2026-07-16T03:18:46.842+0900][123459.511s][info][gc] GC(731) Pause Full (G1 Compaction Pause) 4092M->2510M(4096M) 2722.531ms
```

여기서 확인할 수 있는 내용:

| 값              | 의미               |
| -------------- | ---------------- |
| `03:18:44.120` | Full GC 시작시각     |
| `03:18:46.842` | 종료 부근 시각         |
| `4092M->2510M` | GC 전후 Heap 사용량   |
| `(4096M)`      | 전체 Heap 크기       |
| `2722.531ms`   | JVM 정지시간 약 2.72초 |

#### 12-2. 특정 일자 검색

```bash
grep '2026-07-16' "$JBOSS_HOME"/standalone/log/gc.log* |
grep -iE 'Pause Full|Full GC|System\.gc|Metadata GC'
```

#### 12-3. 특정 장애 시각 전후 검색

ZooKeeper 오류 시각이 `2026-07-16 03:18:45`였으므로 전후 구간을 확인합니다.

```bash
grep '2026-07-16T03:1[7-9]:' \
  "$JBOSS_HOME"/standalone/log/gc.log*
```

압축 파일까지 검색:

```bash
zgrep -iE \
'2026-07-16T03:1[7-9]:.*(Pause Full|Full GC|safepoint|System\.gc)' \
"$JBOSS_HOME"/standalone/log/gc.log*.gz
```

현재 파일과 압축 파일을 함께 검색:

```bash
find "$JBOSS_HOME/standalone/log" -type f -name 'gc.log*' -print0 |
while IFS= read -r -d '' file
do
    case "$file" in
        *.gz)
            zgrep -H -iE \
            '2026-07-16T03:1[7-9]:.*(Pause Full|Full GC|safepoint)' "$file"
            ;;
        *)
            grep -H -iE \
            '2026-07-16T03:1[7-9]:.*(Pause Full|Full GC|safepoint)' "$file"
            ;;
    esac
done
```

### 13. 로그에 실제 날짜가 없는 경우

기존 GC 로그가 다음처럼 JVM uptime만 기록할 수 있습니다.

```text
123456.789: [Full GC ...]
```

이 경우 JVM 시작시각을 알아야 실제 시각을 환산할 수 있습니다.

JBoss 프로세스 시작시각:

```bash
ps -p $JBOSS_PID -o lstart=
```

또는:

```bash
stat -c '%y' /proc/$JBOSS_PID
```

하지만 과거 JVM이 이미 재시작된 경우 현재 `/proc` 정보로 이전 인스턴스의 시작시각을 확인할 수 없습니다. 다음 자료를 찾아야 합니다.

```bash
grep -iE 'JBoss.*started|started in|WFLYSRV0025|WFLYSRV0050' \
  "$JBOSS_HOME"/standalone/log/server.log*
```

systemd 환경:

```bash
journalctl -u jboss \
  --since '2026-07-15 00:00:00' \
  --until '2026-07-17 00:00:00'
```

JBoss 시작시각이 예를 들어 다음과 같다면:

```text
2026-07-14 16:00:00
```

GC 로그 uptime이 다음과 같을 때:

```text
126,000초
```

실제 시각은 대략:

```text
2026-07-16 03:00:00
```

입니다. 단, 수동 계산보다 실제 날짜가 포함되도록 `time` decorator를 설정하는 것이 훨씬 안전합니다.

### 14. GC 로그가 없을 때 과거 Full GC를 추정하는 방법

GC 로그가 활성화되지 않았다면 정확한 확인은 불가능하지만 다음 로그를 이용해 간접 추정할 수 있습니다.

#### 14-1. JBoss server.log

```bash
grep -iE \
'OutOfMemoryError|GC overhead|allocation failure|heartbeat|timeout|blocked|ConnectionLoss|SessionExpired' \
"$JBOSS_HOME"/standalone/log/server.log*
```

특정 시각:

```bash
grep -E '2026-07-16 03:1[7-9]:' \
"$JBOSS_HOME"/standalone/log/server.log*
```

다만 Full GC 동안 JVM 전체가 멈추면 그 시간에는 로그 자체가 출력되지 않을 수 있습니다. 따라서 아래처럼 로그 시간 공백을 확인해야 합니다.

```text
03:18:43.100 정상 로그
03:18:48.500 다음 로그
```

중간 5.4초 공백이 있다고 해서 반드시 Full GC인 것은 아니지만, CPU·GC·네트워크 로그와 함께 보면 단서가 됩니다.

#### 14-2. systemd journal

```bash
journalctl -u jboss \
  --since '2026-07-16 03:15:00' \
  --until '2026-07-16 03:25:00' \
  -o short-iso
```

#### 14-3. 커널 로그

OOM Killer 또는 시스템 메모리 압박:

```bash
journalctl -k \
  --since '2026-07-16 03:10:00' \
  --until '2026-07-16 03:30:00' |
grep -iE 'oom|out of memory|killed process|memory pressure'
```

또는:

```bash
grep -iE 'oom|out of memory|killed process' /var/log/messages*
```

#### 14-4. `sar` 과거 성능 로그

`sysstat`이 당시 활성화되어 있었다면 매우 유용합니다.

2026년 7월 16일:

```bash
sar -u -f /var/log/sa/sa16
```

03:10~03:30 구간 CPU:

```bash
sar -u \
  -s 03:10:00 \
  -e 03:30:00 \
  -f /var/log/sa/sa16
```

메모리:

```bash
sar -r \
  -s 03:10:00 \
  -e 03:30:00 \
  -f /var/log/sa/sa16
```

Swap:

```bash
sar -W \
  -s 03:10:00 \
  -e 03:30:00 \
  -f /var/log/sa/sa16
```

Run queue와 load:

```bash
sar -q \
  -s 03:10:00 \
  -e 03:30:00 \
  -f /var/log/sa/sa16
```

네트워크 오류:

```bash
sar -n DEV,EDEV \
  -s 03:10:00 \
  -e 03:30:00 \
  -f /var/log/sa/sa16
```

Full GC와 함께 다음 패턴이 나타날 수 있습니다.

| 지표                      | 의심 패턴                |
| ----------------------- | -------------------- |
| `%user`, `%system`      | GC 직전 또는 중간 CPU 급증   |
| `%iowait`               | Swap 또는 디스크 병목 동반    |
| `kbmemavailable`        | 여유 메모리 급감            |
| `pswpin/s`, `pswpout/s` | Swap 발생              |
| `runq-sz`               | 실행 대기 프로세스 급증        |
| 시스템 로그                  | JVM 응답 공백·timeout 발생 |

### 15. ZooKeeper ConnectionLoss 오류와 시간 비교

이전 오류 시각:

```text
2026-07-16 03:18:45.205
```

다음 순서로 확인하는 것이 가장 정확합니다.

```bash
GC_DIR="$JBOSS_HOME/standalone/log"
```

#### 15-1. 1단계: Full GC 검색

```bash
grep -H -iE \
'2026-07-16T03:1[7-9]:.*(Pause Full|Full GC)' \
"$GC_DIR"/gc.log*
```

#### 15-2. 2단계: Safepoint 정지시간 검색

```bash
grep -H -iE \
'2026-07-16T03:1[7-9]:.*safepoint' \
"$GC_DIR"/gc.log*
```

#### 15-3. 3단계: JBoss 로그 공백과 오류 검색

```bash
grep -H -E \
'2026-07-16 03:1[7-9]:' \
"$GC_DIR"/server.log*
```

#### 15-4. 4단계: ZooKeeper 관련 오류 검색

```bash
grep -H -iE \
'2026-07-16 03:1[7-9]:.*(ConnectionLoss|SessionExpired|KeeperException)' \
"$GC_DIR"/server.log*
```

판단 기준:

| 확인 결과                                    | 판단                                             |
| ---------------------------------------- | ---------------------------------------------- |
| 03:18:45 전후 긴 `Pause Full` 존재            | GC와 ZooKeeper 오류 연관성 높음                        |
| Full GC 없음, 긴 safepoint 존재               | Thread dump, class redefinition 등 다른 JVM 정지 가능 |
| GC·safepoint 없음, 다수 서버에서 동일 ZooKeeper 오류 | 네트워크 또는 ZooKeeper 자체 장애 가능성 높음                 |
| 해당 JVM에서만 오류＋CPU·Swap 이상                 | WAS 서버 자원 문제 가능성 높음                            |
| Full GC 후 Old 사용률이 거의 감소하지 않음            | 메모리 누수 또는 live object 과다 의심                    |
| Full GC 정지시간이 세션 timeout에 근접·초과          | ZooKeeper 세션 유실과 직접 연관 가능                      |

### 16. Full GC 위험 판단 기준

절대적인 단일 기준은 없지만 운영에서는 다음을 우선 경고 신호로 봅니다.

| 상태                               | 판단                                 |
| -------------------------------- | ---------------------------------- |
| Full GC 1회, 짧은 정지, Old 사용률 크게 감소 | 일시적 현상일 수 있음                       |
| Full GC 반복 간격이 점점 짧아짐            | Heap 압박 또는 누수 의심                   |
| Full GC 후 Old 사용률이 80~90% 이상 유지  | 강한 위험 신호                           |
| Full GC가 수초 이상 지속                | timeout·heartbeat 장애 가능            |
| Full GC 직후 `OutOfMemoryError`    | 치명적                                |
| `System.gc()`가 반복 발생             | 애플리케이션 또는 라이브러리 명시 호출 점검           |
| `Metadata GC Threshold` 반복       | Metaspace·클래스 로더 누수 점검             |
| `G1 Compaction Pause` 반복         | G1이 정상적인 동시 회수로 공간을 확보하지 못하는 상태 가능 |

### 17. 운영 서버 권장 최종 설정

JDK 11 JBoss 7 standalone 기준:

```bash
GC_LOG_DIR="$JBOSS_HOME/standalone/log"

JAVA_OPTS="$JAVA_OPTS \
-Xlog:gc*,safepoint:file=${GC_LOG_DIR}/gc.log:time,uptime,level,tags:filecount=20,filesize=100M \
-XX:+HeapDumpOnOutOfMemoryError \
-XX:HeapDumpPath=${GC_LOG_DIR}/heapdump.hprof"
```

예상 최대 GC 로그 용량:

```text
20개 × 100MB = 약 2GB
```

디스크 용량이 부족하면 다음 정도로 줄일 수 있습니다.

```bash
-Xlog:gc*,safepoint:file=${GC_LOG_DIR}/gc.log:time,uptime,level,tags:filecount=10,filesize=50M
```

최대 약 500MB입니다.

`HeapDumpOnOutOfMemoryError`는 OOM 원인 분석에 유용하지만 Heap dump 파일은 최대 Heap 크기와 비슷한 크기가 될 수 있으므로 저장 공간을 먼저 확보해야 합니다.

### 18. 즉시 실행할 점검 명령

```bash
# 1. JBoss PID
JBOSS_PID=$(pgrep -f 'org.jboss.as.standalone' | head -1)
echo "$JBOSS_PID"

# 2. JVM 가동시간
sudo -u jboss jcmd "$JBOSS_PID" VM.uptime

# 3. 현재 Full GC 누적 횟수·시간
sudo -u jboss jstat -gcutil "$JBOSS_PID" 1000 5

# 4. 마지막 GC 원인
sudo -u jboss jstat -gccause "$JBOSS_PID" 1000 5

# 5. Heap 상태
sudo -u jboss jcmd "$JBOSS_PID" GC.heap_info

# 6. GC 로그 옵션 확인
tr '\0' '\n' < "/proc/$JBOSS_PID/cmdline" |
grep -E 'Xlog|Xloggc|PrintGC'

# 7. 기존 GC 로그 탐색
find "$JBOSS_HOME" -type f \
  \( -iname 'gc.log*' -o -iname '*gc*.log*' \) \
  -ls 2>/dev/null

# 8. 2026-07-16 03:18 전후 Full GC 검색
grep -H -iE \
'2026-07-16T03:1[7-9]:.*(Pause Full|Full GC|safepoint)' \
"$JBOSS_HOME"/standalone/log/gc.log* 2>/dev/null
```

현재 단계에서 가장 중요한 것은 **JVM 실행 옵션에 GC 로그 설정이 이미 존재하는지 확인하는 것**입니다. 설정이 존재하고 2026년 7월 16일 로그 파일이 보존되어 있다면, `03:18:45` 전후의 Full GC·safepoint 정지시간을 ZooKeeper 오류와 직접 비교할 수 있습니다.

[1]: https://docs.oracle.com/en/java/javase/11/tools/java.html?utm_source=chatgpt.com "java"
[2]: https://docs.oracle.com/en/java/javase/11/tools/jstat.html?source=fb&utm_source=chatgpt.com "jstat"
[3]: https://docs.oracle.com/en/java/javase/11/migrate/?utm_source=chatgpt.com "Java Platform, Standard Edition Oracle JDK Migration Guide, Release 11"
[4]: https://access.redhat.com/documentation/mr-in/red_hat_jboss_enterprise_application_platform/7.4/pdf/configuration_guide/red_hat_jboss_enterprise_application_platform-7.4-configuration_guide-en-us.pdf?utm_source=chatgpt.com "Red Hat JBoss Enterprise Application"
[5]: https://access.redhat.com/node/2306591/index.html?utm_source=chatgpt.com "Configuration Guide - Red Hat Customer Portal"
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
