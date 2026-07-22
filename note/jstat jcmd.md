### jstat이란?

`jstat`은 **현재 실행 중인 Java 프로세스의 JVM 상태를 조회하는 명령어**입니다.

주로 다음을 확인할 때 사용합니다.

| 확인 항목     | 설명                           |
| --------- | ---------------------------- |
| Heap 사용량  | Eden, Survivor, Old 영역 사용 상태 |
| GC 횟수     | Young GC, Full GC 발생 횟수      |
| GC 시간     | GC로 인해 사용된 누적 시간             |
| Metaspace | 클래스 정보 저장 영역 사용량             |
| GC 원인     | 마지막 GC가 발생한 원인               |

`jstat`은 로그 파일을 읽는 명령이 아니라, **현재 실행 중인 JVM 내부 통계값을 실시간으로 조회하는 명령**입니다.

따라서 다음 제약이 있습니다.

* JVM을 재시작하면 누적값이 초기화됩니다.
* 과거 특정 시각의 GC 발생 여부는 알 수 없습니다.
* 명령 실행 시점의 JVM 상태와 누적 통계만 확인할 수 있습니다.

### 1. Java 프로세스 PID 확인

`jstat`을 사용하려면 먼저 대상 Java 프로세스의 PID를 알아야 합니다.

#### jps 사용

```bash
jps -lv
```

예시:

```text
12345 /app/jboss/jboss-modules.jar -Djboss.home.dir=/app/jboss
24870 sun.tools.jps.Jps
```

여기서 JBoss PID는 `12345`입니다.

#### ps 사용

```bash
ps -ef | grep '[j]ava'
```

JBoss만 검색:

```bash
ps -ef | grep '[j]boss'
```

JBoss standalone 프로세스 검색:

```bash
pgrep -af 'org.jboss.as.standalone|jboss-modules.jar'
```

PID를 변수로 저장하면 이후 명령을 편하게 사용할 수 있습니다.

```bash
JBOSS_PID=12345
```

자동으로 찾는 예:

```bash
JBOSS_PID=$(pgrep -f 'org.jboss.as.standalone' | head -1)
echo "$JBOSS_PID"
```

> 한 서버에 Java 프로세스가 여러 개 있으면 자동 검색 결과를 그대로 사용하지 말고 반드시 프로세스 명령행을 확인해야 합니다.

### 2. 가장 많이 사용하는 `jstat -gcutil`

```bash
jstat -gcutil <PID> <조회간격ms> <조회횟수>
```

예:

```bash
jstat -gcutil 12345 1000 10
```

의미:

| 값       | 의미               |
| ------- | ---------------- |
| `12345` | 대상 JVM PID       |
| `1000`  | 1,000ms, 즉 1초 간격 |
| `10`    | 총 10회 조회         |

출력 예:

```text
  S0    S1     E      O      M     CCS    YGC   YGCT   FGC   FGCT    CGC   CGCT    GCT
  0.0  12.5   65.3   78.4   91.2   87.6   520   8.31     3   4.82      0   0.00   13.13
  0.0  12.5   72.1   78.4   91.2   87.6   520   8.31     3   4.82      0   0.00   13.13
```

### 3. `jstat -gcutil` 항목 설명

| 항목     | 의미                         | 단위 |
| ------ | -------------------------- | -: |
| `S0`   | Survivor 0 영역 사용률          |  % |
| `S1`   | Survivor 1 영역 사용률          |  % |
| `E`    | Eden 영역 사용률                |  % |
| `O`    | Old 영역 사용률                 |  % |
| `M`    | Metaspace 사용률              |  % |
| `CCS`  | Compressed Class Space 사용률 |  % |
| `YGC`  | Young GC 누적 횟수             |  회 |
| `YGCT` | Young GC 누적 시간             |  초 |
| `FGC`  | Full GC 누적 횟수              |  회 |
| `FGCT` | Full GC 누적 시간              |  초 |
| `CGC`  | Concurrent GC 누적 횟수        |  회 |
| `CGCT` | Concurrent GC 누적 시간        |  초 |
| `GCT`  | 전체 GC 누적 시간                |  초 |

#### 가장 중요하게 볼 항목

```text
E    O    YGC    YGCT    FGC    FGCT    GCT
```

| 항목     | 확인 포인트                           |
| ------ | -------------------------------- |
| `E`    | 빠르게 상승하다가 Young GC 후 떨어지는 것은 일반적 |
| `O`    | 계속 상승하고 GC 후에도 줄지 않으면 주의         |
| `YGC`  | Young GC 횟수 증가 속도 확인             |
| `YGCT` | Young GC 누적 정지시간                 |
| `FGC`  | Full GC 발생 여부                    |
| `FGCT` | Full GC로 소요된 총 시간                |
| `GCT`  | 전체 GC 누적 시간                      |

### 4. 출력 결과 해석 예

```text
YGC=520
YGCT=8.31
FGC=3
FGCT=4.82
GCT=13.13
```

의미:

* JVM 시작 후 Young GC가 520회 발생
* Young GC에 총 8.31초 사용
* Full GC가 3회 발생
* Full GC에 총 4.82초 사용
* 전체 GC에 총 13.13초 사용

Full GC 평균 시간은 대략 다음과 같습니다.

```text
4.82초 ÷ 3회 = 약 1.61초
```

단, 평균값만으로는 각 Full GC가 1.61초였다는 뜻은 아닙니다.

예를 들어 실제로는 다음과 같을 수 있습니다.

```text
1회차: 0.5초
2회차: 0.7초
3회차: 3.62초
```

각 GC의 정확한 정지시간은 GC 로그를 확인해야 합니다.

### 5. 계속 실시간으로 모니터링

조회 횟수를 생략하면 계속 출력됩니다.

```bash
jstat -gcutil 12345 5000
```

의미:

* PID `12345`
* 5초 간격
* `Ctrl+C`로 중지할 때까지 계속 조회

운영에서는 보통 다음처럼 사용합니다.

```bash
jstat -gcutil $JBOSS_PID 5000
```

### 6. Full GC 발생 여부 쉽게 확인

첫 번째 출력:

```text
FGC=3
FGCT=4.82
```

5초 후 출력:

```text
FGC=4
FGCT=7.31
```

변화량:

```text
FGC: 3 → 4
FGCT: 4.82 → 7.31
```

따라서 해당 5초 동안:

* Full GC가 1회 발생
* Full GC 누적시간이 2.49초 증가

한 것으로 볼 수 있습니다.

```text
7.31 - 4.82 = 2.49초
```

### 7. `jstat -gc`

`-gcutil`은 사용률 위주이고, `-gc`는 실제 용량을 KB 단위로 표시합니다.

```bash
jstat -gc 12345 1000 5
```

출력 예:

```text
 S0C    S1C    S0U    S1U      EC       EU        OC        OU       MC       MU      YGC  YGCT  FGC  FGCT
 0.0   8192.0  0.0   4096.0  524288.0 320000.0 2097152.0 1650000.0 300000.0 280000.0 520 8.31 3 4.82
```

주요 항목:

| 접미사 | 의미                  |
| --- | ------------------- |
| `C` | Capacity, 확보된 전체 용량 |
| `U` | Used, 현재 사용 중인 용량   |

예:

| 항목   | 의미              |
| ---- | --------------- |
| `EC` | Eden 전체 용량      |
| `EU` | Eden 사용량        |
| `OC` | Old 전체 용량       |
| `OU` | Old 사용량         |
| `MC` | Metaspace 확보 용량 |
| `MU` | Metaspace 사용량   |

Old 사용률은 직접 계산할 수 있습니다.

```text
OU ÷ OC × 100
```

예:

```text
1,650,000 ÷ 2,097,152 × 100 ≒ 78.7%
```

일반적인 운영 확인에는 `-gcutil`이 더 보기 쉽고, 정확한 용량 분석에는 `-gc`가 적합합니다.

### 8. `jstat -gccause`

GC 발생 원인을 확인할 때 사용합니다.

```bash
jstat -gccause 12345 1000 10
```

출력 마지막 부분 예:

```text
LGCC                     GCC
Allocation Failure       No GC
```

| 항목     | 의미                       |
| ------ | ------------------------ |
| `LGCC` | Last GC Cause, 마지막 GC 원인 |
| `GCC`  | 현재 진행 중인 GC 원인           |

대표적인 GC 원인:

| 원인                        | 설명                          |
| ------------------------- | --------------------------- |
| `Allocation Failure`      | 새 객체를 할당할 공간이 부족함           |
| `Metadata GC Threshold`   | Metaspace 임계치에 도달           |
| `System.gc()`             | 애플리케이션이나 라이브러리가 명시적으로 GC 호출 |
| `G1 Evacuation Pause`     | G1 GC가 Region의 객체를 이동·회수    |
| `G1 Humongous Allocation` | 매우 큰 객체를 할당하면서 GC 발생        |
| `No GC`                   | 현재 GC가 수행 중이지 않음            |

`Allocation Failure`는 반드시 장애라는 의미는 아닙니다. Young 영역이 가득 차면서 정상적인 Young GC가 실행될 때도 나타날 수 있습니다.

### 9. `jstat -gcnew`

Young 영역을 집중적으로 확인합니다.

```bash
jstat -gcnew 12345 1000 5
```

주요 확인 대상:

* Eden
* Survivor
* Young GC 횟수
* 객체 승격 관련 상태

Young GC가 지나치게 자주 발생하는지 분석할 때 사용하지만, 일반적인 운영 점검에서는 `-gcutil`로도 충분합니다.

### 10. `jstat -gcold`

Old 영역 상태를 확인합니다.

```bash
jstat -gcold 12345 1000 5
```

Old 영역과 Full GC를 집중적으로 확인할 때 사용합니다.

다만 실무에서는 다음 두 명령으로 대부분 확인 가능합니다.

```bash
jstat -gcutil $JBOSS_PID 5000
jstat -gccause $JBOSS_PID 5000
```

### 11. `jstat -class`

로드된 클래스 상태를 확인합니다.

```bash
jstat -class 12345 5000
```

출력 예:

```text
Loaded  Bytes  Unloaded  Bytes  Time
45820   89231      120    148.3  35.21
```

| 항목         | 의미             |
| ---------- | -------------- |
| `Loaded`   | 지금까지 로드된 클래스 수 |
| `Unloaded` | 언로드된 클래스 수     |
| `Bytes`    | 클래스 데이터 크기     |
| `Time`     | 클래스 로딩에 사용된 시간 |

클래스 수가 계속 증가하고 거의 줄지 않으면서 Metaspace도 상승한다면 클래스 로더 누수를 의심할 수 있습니다.

JBoss에서 애플리케이션 재배포를 반복할 때 특히 확인할 가치가 있습니다.

### 12. `jcmd`와의 차이

`jcmd`는 JVM에 진단 명령을 전달하는 도구입니다.

| 도구       | 주요 용도                                 |
| -------- | ------------------------------------- |
| `jstat`  | JVM 통계를 짧은 주기로 반복 조회                  |
| `jcmd`   | Heap, JVM 옵션, uptime, thread 등의 상세 진단 |
| `jstack` | 전체 Java Thread 상태 확인                  |
| `jmap`   | Heap 및 객체 분포 분석                       |
| `jps`    | Java 프로세스 PID 확인                      |

### 13. 자주 사용하는 `jcmd`

#### 지원 명령 목록

```bash
jcmd 12345 help
```

#### JVM 가동시간

```bash
jcmd 12345 VM.uptime
```

예:

```text
12345:
86400.532 s
```

JVM이 약 86,400초, 즉 약 1일 동안 실행됐다는 의미입니다.

`jstat`의 GC 횟수와 시간은 이 가동기간 동안 누적된 값입니다.

#### JVM 실행 옵션

```bash
jcmd 12345 VM.flags
```

초기 명령행 포함:

```bash
jcmd 12345 VM.command_line
```

확인 가능한 내용:

* `-Xms`
* `-Xmx`
* GC 종류
* GC 로그 설정
* 시스템 프로퍼티
* JBoss 실행 옵션

#### Heap 상태

```bash
jcmd 12345 GC.heap_info
```

예:

```text
garbage-first heap
 total 4194304K, used 3250000K
 region size 2048K
 Metaspace used 385000K
```

| 항목               | 의미               |
| ---------------- | ---------------- |
| `total`          | JVM이 현재 확보한 Heap |
| `used`           | 현재 Heap 사용량      |
| `region size`    | G1 Region 크기     |
| `Metaspace used` | Metaspace 사용량    |

#### 객체별 메모리 사용량

```bash
jcmd 12345 GC.class_histogram
```

출력 예:

```text
 num     #instances         #bytes  class name
------------------------------------------------
   1:       1520000      182400000  [B
   2:        850000       68000000  java.lang.String
   3:        420000       33600000  java.util.HashMap$Node
```

주요 의미:

| 항목           | 의미                |
| ------------ | ----------------- |
| `#instances` | 해당 객체 개수          |
| `#bytes`     | 해당 객체가 차지하는 총 메모리 |
| `class name` | 클래스 이름            |

> 이 명령은 객체 수가 많은 운영 JVM에서 일시적인 부하를 줄 수 있으므로 반복 실행하지 않는 것이 좋습니다.

### 14. `jstack`

현재 JVM의 전체 Thread 상태를 출력합니다.

```bash
jstack -l 12345
```

파일 저장:

```bash
jstack -l 12345 > /tmp/jboss_thread_$(date +%Y%m%d_%H%M%S).txt
```

확인 가능한 상태:

| 상태              | 의미                      |
| --------------- | ----------------------- |
| `RUNNABLE`      | 실행 중이거나 CPU 할당 대기       |
| `BLOCKED`       | synchronized lock 획득 대기 |
| `WAITING`       | 무기한 대기                  |
| `TIMED_WAITING` | 제한시간을 두고 대기             |

다음 상황에서 사용합니다.

* WAS 응답 정지
* DB Connection 대기 증가
* 스레드 풀 고갈
* Deadlock 의심
* 외부 API 응답 지연

Thread dump는 보통 5~10초 간격으로 3회 정도 수집해야 흐름을 파악하기 쉽습니다.

```bash
for i in 1 2 3
do
    jstack -l 12345 > "/tmp/jstack_${i}_$(date +%H%M%S).txt"
    sleep 5
done
```

### 15. `jmap`

Heap 정보를 확인하는 도구입니다.

객체 통계:

```bash
jmap -histo 12345
```

Heap dump 생성:

```bash
jmap -dump:live,format=b,file=/tmp/jboss_heap.hprof 12345
```

> 운영 서버에서 `jmap -dump:live`는 Full GC와 JVM 정지를 유발할 수 있습니다. 장애 분석 목적으로 무분별하게 실행하면 안 됩니다.

Heap dump는 보통 다음 옵션으로 OOM 발생 시 자동 생성하는 것이 안전합니다.

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/app/jboss/standalone/log
```

### 16. `jps`

현재 실행 중인 Java 프로세스를 확인합니다.

```bash
jps
```

상세 옵션 포함:

```bash
jps -lv
```

| 옵션   | 의미                  |
| ---- | ------------------- |
| `-l` | 메인 클래스 또는 JAR 전체 이름 |
| `-v` | JVM 옵션 표시           |
| `-m` | main 메서드 인자 표시      |

운영 환경에서는 `jps`에 일부 프로세스가 보이지 않을 수 있으므로 `ps`나 `pgrep`도 함께 사용합니다.

### 17. JBoss 운영 점검 시 권장 순서

#### 1단계: PID 확인

```bash
pgrep -af 'org.jboss.as.standalone|jboss-modules.jar'
```

#### 2단계: JVM 가동시간 확인

```bash
jcmd $JBOSS_PID VM.uptime
```

#### 3단계: GC 상태 확인

```bash
jstat -gcutil $JBOSS_PID 5000 12
```

약 1분 동안 확인합니다.

#### 4단계: GC 원인 확인

```bash
jstat -gccause $JBOSS_PID 5000 12
```

#### 5단계: Heap 확인

```bash
jcmd $JBOSS_PID GC.heap_info
```

#### 6단계: JVM 옵션 확인

```bash
jcmd $JBOSS_PID VM.command_line
jcmd $JBOSS_PID VM.flags
```

#### 7단계: 응답 지연이 있으면 Thread dump

```bash
jstack -l $JBOSS_PID > /tmp/jboss_thread_$(date +%Y%m%d_%H%M%S).txt
```

### 18. 한 번에 확인하는 간단 명령

```bash
JBOSS_PID=$(pgrep -f 'org.jboss.as.standalone' | head -1)

echo "PID=$JBOSS_PID"

echo "===== JVM UPTIME ====="
jcmd "$JBOSS_PID" VM.uptime

echo "===== JVM FLAGS ====="
jcmd "$JBOSS_PID" VM.flags

echo "===== HEAP INFO ====="
jcmd "$JBOSS_PID" GC.heap_info

echo "===== GC STATUS ====="
jstat -gcutil "$JBOSS_PID" 1000 5

echo "===== GC CAUSE ====="
jstat -gccause "$JBOSS_PID" 1000 3
```

JBoss 실행 계정이 `jboss`라면:

```bash
sudo -u jboss jstat -gcutil "$JBOSS_PID" 1000 5
```

### 19. 주의사항

| 명령                   | 운영 영향 | 비고            |
| -------------------- | ----: | ------------- |
| `jstat -gcutil`      | 매우 낮음 | 지속 모니터링 가능    |
| `jstat -gccause`     | 매우 낮음 | GC 원인 확인      |
| `jcmd VM.uptime`     | 매우 낮음 | 안전            |
| `jcmd VM.flags`      | 매우 낮음 | 안전            |
| `jcmd GC.heap_info`  |    낮음 | 일반적으로 안전      |
| `jstack`             |    낮음 | 순간적인 부하 가능    |
| `GC.class_histogram` |    중간 | 대형 Heap에서 주의  |
| `jmap -histo`        | 중간~높음 | 운영 반복 실행 금지   |
| `jmap -dump:live`    |    높음 | Full GC·정지 가능 |

### 핵심 명령 5개

```bash
# Java 프로세스 확인
jps -lv

# GC 횟수와 사용률 확인
jstat -gcutil <PID> 5000

# GC 원인 확인
jstat -gccause <PID> 5000

# Heap 상태 확인
jcmd <PID> GC.heap_info

# JVM 정지·응답 지연 시 Thread 확인
jstack -l <PID> > /tmp/jstack.txt
```

운영에서 가장 먼저 사용할 명령은 다음입니다.

```bash
jstat -gcutil $JBOSS_PID 5000
```

여기서 `FGC`가 증가하는지, `FGCT`가 빠르게 늘어나는지, `O` 영역이 GC 후에도 계속 높은 상태인지 확인하면 됩니다.
