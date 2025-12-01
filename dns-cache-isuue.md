# DNS Cache Issue (DNS 캐시 문제)

## 1. 문제 상황 (Symptom)

특정 서버 또는 도메인에 접속이 안 되거나,
이미 IP가 변경된 도메인인데 **예전 IP로 계속 접속을 시도하는 현상** 발생

주요 증상:
- 브라우저: `This site can’t be reached`
- ping 결과가 실제 서버 IP와 다름
- 서버 IP를 변경했는데도 예전 서버로 접속됨
- 다른 PC에서는 접속되는데 내 컴퓨터에서만 안 됨

즉,

> "도메인의 IP가 바뀌었는데도 클라이언트가 과거 IP를 계속 사용함"

---

## 2. 원인 (Root Cause)

DNS는 성능 향상을 위해 결과를 **캐시에 저장**한다.

이 캐시는 다음 위치들에 저장될 수 있음:

- OS 내부 DNS 캐시
- 브라우저 DNS 캐시
- 라우터 캐시
- ISP DNS 서버 캐시
- `/etc/hosts` 파일

하지만 도메인의 실제 IP가 변경되었는데 캐시가 갱신되지 않으면
**잘못된 IP로 계속 접속 시도**하게 된다.

대표 상황:

- 서버를 재배포했거나, IP가 변경된 경우
- Load Balancer 구조 변경
- 도메인을 다른 서버로 이전한 경우
- /etc/hosts 에 과거 IP가 남아 있는 경우

---

## 3. 확인 방법 (How to Check)

### ✅ 현재 DNS가 어느 IP를 가리키는지 확인

#### Linux / Mac
```bash
nslookup example.com
dig example.com
ping example.com
```

### ✅ /etc/hosts 확인
```bash
cat /etc/hosts
```
다음과 같은 라인이 있으면 문제 원인이 될 수 있음
```bash
192.168.0.10   example.com
```
이 줄이 있으면 DNS와 상관 없이 이 IP로 강제 연결됨.

## 4. 해결 방법
### ✅ 1) DNS 캐시 초기화
Mac
```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```
Ubuntu (systemd 사용 시)
```bash
sudo systemctl restart systemd-resolved
# 또는
sudo resolvectl flush-caches
```
Windows
```bash
ipconfig /flushdns
```
### ✅ 2) 브라우저 캐시 삭제
가장 빠른 테스트 방법
- 크롬: Ctrl + Shift + R (강력 새로고침)
- 또는 시크릿 모드로 접속
  
### ✅ 3) /etc/hosts 수정
잘못된 내용이 있다면 해당 줄 삭제 후
```bash
sudo systemctl restart networking
```
## 5. 모의 실험 방법 (Mock DNS Cache Issue Test)

로컬(Ubuntu or Mac + UTM)에서 직접 테스트하는 방법

🎯 목표

"도메인이 다른 IP를 가리키게 만든 뒤, 캐시 때문에 이전 IP로 접속되는 상황을 만들어보기"

### ✅ 방법 ① /etc/hosts를 이용한 강제 DNS 설정

1단계 — hosts 수정 (Ubuntu or Mac)
```bash
sudo nano /etc/hosts
```
아래 추가
```bash
1.1.1.1   fake.com
```
2단계 - 테스트
```bash
ping fake.com
# 또는
ssh fake.com
```
3단계 - hosts 수정 변경
```bash
8.8.8.8   fake.com
```
그런데도 접속이 계속 1.1.1.1로 된다면?
→ DNS 캐시 또는 local cache 영향

### ✅ 방법 ② 실제 서버 IP 변경 실험

### 1) UTM 우분투에 웹서버 실행
Ubuntu 에서 아래 명령어 실행
```bash
sudo apt install apache2 -y
sudo systemctl start apache2
```
브라우저에서 접속
```bash
http://<ubuntu-ip>
```
정상작동 확인
### 2) Mac에서 hosts 편집
Mac 터미널
```bash
sudo nano /etc/hosts
<ubuntu-ip> testserver.com
```
### 3) 브라우저에서 접속
```bash
http://testserver.com
```
→ 정상
### 4) UTM Ubuntu IP 변경 후, 다시 접속

그런데도 이전 IP로 접속 시도되면?
→ DNS 캐싱 문제 발생 상황

## 6. 실무에서 매우 자주 발생하는 상황

이런 상황에서 특히 많이 발생함

- 도메인을 새 서버로 옮겼을 때
- 서버 점검 후 복구했는데 접속 안 되는 경우
- CDN/로드밸런서 설정 변경 후
- VPN 사용 후
- 특정 PC에서만 접속 안 될 때

그래서 실무에서는
> "서버 문제인가?" 하기 전에 항상 DNS 캐시부터 의심한다.

### ✅ 주의사항 / 실무 팁 (Important Notes)

### 1. /etc/hosts 수정만으로도 즉시 복구되는 경우가 있다.

/etc/hosts에서 잘못된 IP 매핑을 제거하면
DNS flush를 하지 않아도 바로 원래 IP로 복구되는 것처럼 보이는 경우가 많다.

예시 상황:
```bash
/etc/hosts

127.0.0.1   www.naver.com   ← 임시 테스트
(이후 삭제)
```

이후 곧바로
```bash
ping www.naver.com
```

을 했을 때 정상 IP로 돌아오는 현상을 확인할 수 있다.


👉 이는 시스템 환경, TTL, 최근 DNS 요청 이력 등에 따라 자동으로 정상 복구되는 경우가 있기 때문이다.

그러나 이것이 항상 보장되지는 않는다.

### 2. 그럼에도 DNS 를 초기화 해야하는 이유

운영 환경에서는 다음과 같은 이유로
/etc/hosts 를 수정했음에도 잘못된 IP가 계속 유지되는 상황이 발생할 수 있다.

캐시가 존재하는 위치:
- OS 레벨 DNS 캐시
- mDNSResponder
- 브라우저 DNS 캐시 (Chrome, Safari 등)
- VPN / 보안 프로그램의 내부 캐시
- 애플리케이션 자체 캐시 (Java, Node, Docker 등)
  
이 때문에 다음과 같은 상황이 생길 수 있다.
- 터미널에서는 정상인데 브라우저는 계속 잘못된 사이트 접속
- ping 은 정상인데 서비스 접속 실패
- 특정 프로그램만 옛 IP를 사용

그래서 실무에서는 각 OS 에 맞는 dns 캐시 초기화 방법을 사용하여
혹시 남아 있을지 모를 캐시를 완전히 제거하는 것이 안전한다.

### 3. nslookup 결과가 다를 수 있는 이유

nslookup 명령은 /etc/hosts가 아니라
외부 DNS 서버에 직접 질의하는 경우가 많다.

그래서 다음과 같은 상황이 발생할 수 있다.
```bash
ping fake.com          -> 1.1.1.1 (hosts 기준)
nslookup fake.com      -> 77.68.x.x (DNS 서버 기준)
```

이것은 문제가 아니라,
정상적인 작동 방식의 차이이다.

따라서 /etc/hosts 테스트 시에는 다음 명령이 더 정확하다
```bash
# hosts 적용 확인용
dscacheutil -q host -a name fake.com
# 또는
ping fake.com
```

### 4. 실무 대응 기준 정리

실무에서는 다음과 같이 대응하는 것이 가장 안전하다.

1. /etc/hosts 수정 또는 삭제

2. DNS 캐시 초기화

3. ping, curl, 브라우저 접속 확인

> /etc/hosts 수정만으로 해결되는 경우도 있지만,
DNS 캐시 전체를 초기화하는 것이 가장 확실하고 재발을 방지한다.