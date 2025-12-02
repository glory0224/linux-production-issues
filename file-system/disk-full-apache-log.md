# 🚨 [File System] Apache 로그로 인한 디스크 100% 사용 문제

## 📌 문제 요약
웹 서비스 응답이 갑자기 느려지다가 이후 접속 불가 상태가 되었으며, SSH는 로그인 가능했으나 웹 요청은 계속 timeout 발생.

## 🧩 발생 상황
- 웹 애플리케이션 정상 동작 중이었음
- `systemctl status httpd` 확인 결과 Apache는 구동 중
- 서버 CPU/메모리 사용량 평소와 큰 차이 없음
- `curl localhost` 요청 시 응답 없음

## 🔍 원인 분석
1️⃣ 디스크 사용량 점검  
```bash
df -h
```
→ /dev/sda1 100% 사용

2️⃣ 어떤 디렉토리가 용량을 많이 차지하는지 확인
``` bash
du -sh /* | sort -h
du -sh /var/* | sort - h
du -sh /var/log/* | sort -h
```
→ /var/log/httpd/access_log 파일 용량 50GB

3️⃣ logrotate 비정상 동작 확인
```bash
cat /etc/logrotate.d/httpd
```
→ rotate 설정은 있으나 실제 rotation이 동작하지 않은 상황 발견
원인: copytruncate 옵션 없이 postrotate 스크립트 fail 발생

## 🛠️ 해결절차
### 1) 서비스 중단을 최소화하기 위해 로그 파일 삭제 대신 truncate
```bash
cp /var/log/httpd/access_log /var/log/httpd/access_log.bak # 안전 백업
> /var/log/httpd/access_log # 파일 내용만 비우기
```
### 2) Apache 다운 여부 확인 및 재시작
```bash
systemctl restart httpd
systemctl status httpd
```
→ 정상 응답 복구됨 확인
### 3) 디스크 용량 재확인
```bash
df -h
```
## 🔁 재발 방지 대책
### ✅ logrotate 설정 수정
/etc/logrotate.d/httpd 수정
```bash

 /var/log/httpd/*log {
    daily
    rotate 14
    missingok
    notifempty
    compress
    delaycompress
    copytruncate
}
```
### ✅ logrotate 강제 테스트
```bash
logrotate -f /etc/logrotate.d/httpd
```
### ✅ 모니터링 및 알림 추가
Prometheus or CloudWatch / Grafana / Zabbix 등에서 

- 디스크 사용률 75% 경고
- 85% 치명적 경고
- Webhook + Slack / 이메일 알림
  
## 🛠️ Troubleshooting 시 유의사항

|방법|설명|
|---------------|----------------------|
|rm access_log |파일 삭제해도 프로세스가 핸들 잡고 있으면 공간 반납 x|
|truncate|가장 안전한 방법 (> 파일명)|
|kill -9|Apache 강제 종료 시 세션 유실 가능|
|systemctl restart httpd|서비스 중단 시간 짧음, 유지 보수 시 우선

추가 확인 추천:
```bash
lsof | grep deleted | grep log
```

→ 삭제된 로그 파일을 여전히 잡고 있는 프로세스 존재 여부 확인

## 🧠 배운 점
- "디스크 100% 사용"은 CPU/MEMORY 가 정상이어도 웹 서비스 다운으로 이어질 수 있다.
- 로그 파일은 삭제보다 "truncate"가 안전하고 빠르다.
- logrotate는 설정만 되어 있다고 끝이 아니라 실제 동작 확인이 필수이다.
- 디스크 용량 기반 알림은 가장 기본적이면서 가장 중요한 운영 안정성 요소이다.

## 🔁 참고 명령어 치트시트
|목적|명령어|
|---------------|----------------------|
|디스크 사용량 확인 |df -h|
|디렉토리 용량 분석|du -sh *|
|가장 큰 파일 찾기|find / -type f -size +1G|
|열린 파일 목록|lsof
|로테이션 테스트|logrotate -f < config >