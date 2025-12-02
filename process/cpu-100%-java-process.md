# CPU 100% - Java Process

## 📌 현상 (Symptoms)

- 서버 CPU 사용률이 100%에 근접하거나 고정됨
- `top` / `htop`에서 특정 Java 프로세스가 CPU를 과도하게 점유
- 앱 응답 지연 또는 서버 접속 불가 현상 발생
- 로드밸런서 헬스체크 실패 가능성

```bash
top -o %CPU
```
결과 예시
```bash
PID   USER   %CPU  COMMAND
2354  root   198.5  java
```
## 🔍 주요 원인 (Causes)

1. 무한 루프 (while(true))

2. 잘못된 재귀 호출

3. Thread 폭증

4. Deadlock 후 busy wait

5. 대용량 데이터 처리 / 정렬

6. GC 폭주

7. 잘못된 Batch 작업

## 🧪 문제 분석 절차 (Investigation)

### 1. 어떤 프로세스가 CPU를 먹고 있는지 확인

```bash
top
ps -ef | grep java
```

### 2. 어떤 스레드가 문제인지 확인

```bash
top -H -p <PID>
```
→ Thread ID 확인 후 16진수로 변환
```bash
printf "%x\n" <TID>
```
### 3. 스레드 덤프 획득
```bash
jstack <PID> > dump.txt
```
16진수 Thread ID 검색해서 어떤 메서드에서 멈췄는지 확인

### 🧯 즉시 조치 방법 (Immediate Action)
|방법|설명|
|---------------|----------------------|
|kill -3 < PID > |	스레드 덤프 확인
|kill -9 < PID > | 	강제 종료
|systemctl restart <서비스>	|서비스 재시작
|CPU 제한	|cgroups, cpulimit
```bash
cpulimit -p <PID> -l 50
```

### 🛠 재발 방지 방법 (Prevention)

#### ✅ 코드 개선
#### ✅ Thread Pool 제한
#### ✅ timeout 설정
#### ✅ while 문에 탈출 조건 추가
#### ✅ Batch 분할 처리
#### ✅ 로그와 모니터링 구축
#### ✅ CPU 알람 설정

권장 모니터링
- Prometheus
- Grafana
- Datadog

## ✅ Mac + UTM + Ubuntu에서 CPU 100% (Java) 모의 테스트

### 1️⃣ UTM 환경 준비 (Mac 기준)

UTM에서 Ubuntu VM 생성 시 추천 설정:

Memory: 2GB 이상

Cores: 2 이상 (테스트 확실히 하려면 4 추천)

Storage: 10GB 이상

Network: Shared Network

Ubuntu에 접속 후 먼저 확인
```bash
htop   # 없으면
sudo apt install htop -y
```

### 2️⃣ Java 설치

Ubuntu VM에서
```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
java -version
```

### 3️⃣ CPU 100% 만드는 Java 코드 생성

Ubuntu VM에서
```bash
mkdir cpu-test
cd cpu-test
nano CpuTest.java
```

재귀 코드로 cpu 점유율 100프로 만드는 코드 작성
```java
public class CpuTest {
    public static void main(String[] args) {
        System.out.println("CPU stress started...");
        while(true) {
        }
    }
}
```
컴파일 & 실행
```bash
javac CpuTest.java
java CpuTest
```

### 4️⃣ CPU 사용량 확인 (다른 터미널에서)

mac utm 으로 java 프로그램을 실행시킨 상태에서

mac 터미널(iterm2)을 통해 utm ubuntu 서버로 ssh 접근 후 아래 명령어 실행

```bash
top -o %CPU
```

결과

- java 프로세스가 100%에 고정

### 5️⃣ 실무처럼 진단 연습하기 (중요한 파트)
#### ✅ 1. PID 확인

```bash
ps -ef | grep CpuTest
```
#### ✅ 2. Thread 단위 확인
```bash
top -H -p PID
```

예: 2356

16진수로 변환
```java
printf "%x\n" 2356
```

예: 934

#### ✅ 3. Thread Dump 분석
 ```bash
 jstack 2354 > dump.txt
nano dump.txt
 ```

 그리고 0x934 검색
```bash
Ctrl + W → 0x934
```

결과 예시
```bash
"Thread-0" runnable
   at CpuTest.main(CpuTest.java:4)
```

## 6️⃣ 실제 장애 시나리오처럼 연습하는 팁

이렇게 해보면 최고야👇

### ✅ Ubuntu에 nginx 실행
### ✅ 한쪽에서 Java CPU 100% 유발
### ✅ curl로 웹 호출
### ✅ 응답 지연 확인

```bash
sudo apt install nginx -y
curl localhost
```

→ 그 상태에서 java 부하 걸면 응답 느려짐 확인 가능

## 🔎 jstack을 파일로 저장하는 이유

### ✅ 실시간으로 터미널에 안 보고 파일로 떨구는 이유

```bash
jstack <PID> > dump.txt
```

### 1. Thread Dump는 수천줄까지 나옴
- 터미널에서 보면 거의 읽을 수도 없고 기록도 못함
### 2. 장애 시점 증거 보존 
- 실제  장애  발생 시엔  1분 후에 증상이 사라 질 수도 있고
- 재현이 안될 수 도 있음
- 그래서 장애 당시에 파일로 남기는 것이 핵심
### 3. 팀원/개발자/리더와  공유용
- dump 파일은 그대로 전달해서 분석 가능
- "내 PC에서 봐야해요" 와 같은 문제 발생 x
### 💬 실무에서 dump 언급 방식
“장애 시점의 thread dump를 채취해서 분석했고…”
즉 dump.txt는 증거 자료야.
## 🔎 왜 16진수로 변환함?
top / htop 에서 보이는 건 → 10진수 TID
```bash
top -H -p 2354
# 결과
2356  99%  java # → 이건 10진수
```
### 하지만 jstack 에 나오는 건 → 16진수
```bash
"Thread-0" ... nid=0x934 runnable
```
여기서 0x934가 중요 → 이게 16진수
그래서 mapping 하려면
```bash
printf "%x\n" 2356
```
→ 결과: 934
### TIP)
#### ❌ 저 printf "%x\n" 2356 는 자바 문법이 아니다
#### ✅ 이건 리눅스 쉘(Bash)에 내장된 printf 명령어야
#### ✅ 즉, 우분투 서버 터미널에서 바로 실행하는 명령어다
#### ✅ 어떤 코드도 작성할 필요가 없다

즉, 서버에 접속해서
```bash
ssh user@server
```
그 다음 그냥 입력
```bash
printf "%x\n" 2356
```
끝이다.

## 정리
|방법|설명|
|---------------|----------------------|
top | 2356 (10진수)
jstack | 0x934 (16진수)
→ 둘은 같은 스레드

이걸 안 하면

#### ✅ 어떤 스레드가 CPU를 먹는지
#### ✅ 어떤 코드 줄이 문제인지

연결 시키기 어렵다. 그래서 이것은

"CPU 100% 분석의 핵심 연결 고리"
