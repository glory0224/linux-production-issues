# sudoers — "no tty" 문제 (no tty present / requiretty 관련)

## 🔥 문제 요약 (Symptoms)
- 원격에서 스크립트/cron/자동화 툴로 `sudo` 실행 시 아래와 같은 에러가 발생
```bash
sudo: no tty present and no askpass program specified
# 또는
sudo: sorry, you must have a tty to run sudo
```
- 수동으로 로그인(대화형 TTY) 후 `sudo` 명령은 정상 동작
- 자동화(ssh without -t, cron, systemd service, ansible ad-hoc 등)에서 실패

---

## 🧩 원인 (Root causes)
1. **sudoers에 `requiretty` 옵션이 설정된 경우**  
 - (보통 RHEL 계열에서 기본 사용) `Defaults requiretty` — TTY(터미널)가 없으면 sudo 불허
2. **자동화 환경(비대화형)에서 사용자에게 패스워드 입력을 요구하는데, TTY/askpass가 없어 입력 불가**
 - 에러 메시지에 `no askpass program specified` 가 같이 나오는 경우
3. **sudoers 설정이 잘못되어 특정 사용자/그룹에 대해 TTY를 요구하는 경우**
4. **systemd 서비스 / cron / ssh 비대화형 세션에서 TTY가 할당되지 않아 발생**

---

## 🔍 진단 (How to check)
- 에러 로그 확인
```bash
journalctl -u your-service
tail -n 200 /var/log/auth.log    # Ubuntu
tail -n 200 /var/log/secure      # RHEL
```

- sudoers 전체에서 requiretty 검색
```bash
sudo grep -n requiretty /etc/sudoers /etc/sudoers.d/* || true
```
- 자동화 명령(비대화형)에서 어떤 환경인지 확인
```bash
ssh user@host 'tty || echo NO_TTY'
```
→ 결과가 NO_TTY면 비대화형(tty 없음)
- 실험: 대화형 vs 비대화형
```bash
# 비대화형 (expect "no tty" 형태)
ssh user@host "sudo -l"        # without -t may fail differently
# 강제 TTY 할당
ssh -t user@host "sudo -l"     # with -t should succeed if TTY is required
```
## 🛠 해결 방법 (Fixes / Options)
### 1) (권장) 특정 사용자/그룹에 대해 !requiretty 설정
```bash
# 예: user2에 대해 TTY 요구 비활성화
Defaults:user2 !requiretty
# 또는 그룹 단위
Defaults:%automation !requiretty
```
> 중요: 절대 직접 `/etc/sudoers`를 `vi`로 편집하지 마세요 — 항상 visudo 사용 (문법 검사 및 잠금). 파일 만들 때는 `visudo -f /etc/sudoers.d/90-no-requiretty.`

### 2) 자동화에서 TTY를 강제로 할당

- SSH에서 -t 옵션 사용
```bash
ssh -t user@host "sudo your-command"
```
**그러나 자동화 프레임워크(Ansible 등)에서는 별도 설정 필요.**
### 3) 비대화형에서 패스워드 요구 문제 해결
- NOPASSWD로 sudo를 허용(안전성 고려 필요)
```bash
user2 ALL=(ALL) NOPASSWD: /path/to/allowed-cmd
```
- 또는 askpass 프로그램 설정(보안상 잘 쓰이지 않음)
### 4) 전체 requiretty 비활성화 (운영 정책에 따라)
- `/etc/sudoers` 또는 `/etc/sudoers.d/*`에서 `Defaults requiretty` 줄을 제거하거나 주석 처리

- 주의: 보안정책 상 허용 여부를 반드시 검토
### ⚠ 안전 권고 (Security note)
- `NOPASSWD`와 `!requiretty`는 편리하지만 권한 범위를 최소화 하세요 (특정 명령만 허용).
- 가능하면 특정 그룹/계정에 한정해서 적용하고, 감사/로깅은 활성화하세요.

## 🧪 로컬(UTM Ubuntu)에서 안전하게 재현해보기
> 목표: sudo: no tty present and no askpass program specified 또는 you must have a tty 를 재현하고, 해결(해당 사용자에 대해 !requiretty 적용 또는 ssh -t로 회피)을 실습

### 준비 (Ubuntu VM)
```bash
# 1. 새로운 테스트 사용자 생성
sudo adduser --disabled-password --gecos "" testuser

# 2. (옵션) sudo 그룹에 추가하지 않음 → sudoers 파일로 접근 권한 별도 설정할 예정
#    또는 sudo 그룹 추가하고 sudoers 덮어쓰기 테스트 가능
sudo usermod -aG sudo testuser
# 3. 비밀번호 설정 - 비밀번호를 설정하지 않으면 다른 클라이언트 환경에서 SSH 접근 시도 시 비밀번호를 요구하는데 계정에 비밀번호가 없어서 permission denied 에러만 계속 발생
sudo passwd testuser
```

### A. (RHEL-style) requiretty를 강제하는 방식 재현 (Ubuntu에서는 기본 불포함 — 아래처럼 직접 추가)

> **Ubuntu에 기본 `requiretty`가 없으므로 `sudoers.d`파일을 만들어 테스트합니다. 절대 직접 `/etc/sudoers` 편집 금지 — `visudo` 사용.**

```bash
# 1) visudo로 새 파일 생성 (안전)
sudo visudo -f /etc/sudoers.d/99-requiretty-test

# 파일 내용 (visudo 편집기 안에 붙여넣기):
# -------------------------
Defaults requiretty
# 또는 특정 사용자에 대해 강제
# Defaults:testuser requiretty
# -------------------------
# 저장 후 종료
```
- 비대화형으로 로컬(또는 Mac에서 ssh)에서 시도
```bash
ssh testuser@(가상환경IP) "sudo whoami"
# 출력 예: 
# sudo: a password is required (or) sudo: no tty present and no askpass program specified
# sudo: sorry, you must have a tty to run sudo
```
- 대화형으로 강제 ssh 접근 시도
```bash
ssh -t testuser@(가상환경IP) "sudo whoami"
# 출력 root
```
### B. 자동화(크론)에서 재현

### 1. testuser로 스크립트 작성 (vi 사용)
```bash
su - testuser
vi /home/testuser/test-sudo.sh
```
vi 안에 아래 내용 작성
```bash
#!/bin/bash
sudo whoami > /home/testuser/sudo.log 2>&1
# 저장 후 나가기
wd
```
### 2. 실행 권한 부여
```bash
chmod +x /home/testuser/test-sudo.sh
```
### 3. 수동 실행 테스트
크론 걸기 전에 먼저 확인
```bash
/home/testuser/test-sudo.sh
```
터미널에서 test-sudo.sh 로 접근
```bash
 ssh testuser@(가상환경 IP) "/home/testuser/test-sudo.sh"
```
우분투 서버에서 sudo.log 확인
```bash
cat /home/testuser/sudo.log
# sudo: sorry, you must have a tty to run sudo 가 로그 찍히면 정상
```

### 4. 크론 등록 (vi로 직접 편집)

testuser 상태에서
```bash
crontab -e
```
맨 아래 줄에 추가
```bash
* * * * * /home/testuser/test-sudo.sh
```
저장한 뒤 1분 정도 기다리고

`cat /home/testuser/sudo.log` 실행

`sudo: sorry, you must have a tty to run sudo`

→ 여기 나오는 에러가 바로 TTY 없는 환경에서 sudo 실패하는 로그다.


즉 이게 진짜 “크론 + requiretty” 시나리오다.

## 🧾 복구 / 해결 실습 (안전한 방법)
### 1) 특정 사용자에 대해 requiretty 비활성화 (권장)
```bash
# 안전하게 /etc/sudoers.d 에 설정
sudo visudo -f /etc/sudoers.d/90-no-requiretty

# 파일 내용 예시:
Defaults:testuser !requiretty
# 또는 그룹 단위
# Defaults:%automation !requiretty
```
### 2) 또는 자동화 측에서 TTY 강제
- SSH: `ssh -t user@host 'sudo ...'`
- Ansible 등: `ansible -m shell -a "sudo ..." --ask-become-pass `혹은 `become: true` 및 `become_method: sudo` 사용 (Ansible 쪽 설정 필요)
  
### 3) NOPASSWD로 특정 명령만 허용 (더 안전)

```bash
# /etc/sudoers.d/10-testuser
testuser ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp, /usr/bin/journalctl -u myapp
# visudo -f /etc/sudoers.d/10-testuser 로 생성
```

## 🔁 체크 리스트 (운영 시)
- sudoers에 `Defaults requiretty` 존재 여부 확인
- 자동화 계정에 대해 `!requiretty` 적용 여부 검토
- cron/systemd 서비스는 TTY 없음 → 적절한 sudoers 설정 필요
- 범위 최소화: NOPASSWD 적용 시 허용 명령 최소화
- 변경 시 `visudo` 사용(문법 체크)
- 변경 시 로그/감사 활성화 확인

## 마지막으로 중요한 포인트 하나

만약 나중에 "자동화에서도 sudo 쓰게 하고 싶다"면
TTY가 아니라 **NOPASSWD + requiretty 해제** 쪽으로 가야 한다.

## 1️⃣ 자동화에서 sudo를 쓰려면 기본 개념

### 자동화(cron, ssh, Ansible, CI/CD 등)의 특징

- 비대화형(non-interactive)

- TTY가 없음

- 비밀번호 입력 불가

- 사용자 개입이 없어야 함

### 그래서 자동화에서 sudo를 쓰려면 반드시 이 2가지 조건이 필요

#### ✅ NOPASSWD 설정
#### ✅ requiretty 비활성화
> **"자동화에서 sudo를 안전하게 쓰려면
NOPASSWD + requiretty 해제"가 정석이다.**

## 2️⃣ 자동화 계정을 위한 정석 sudoers 설정

실무에서 가장 많이 쓰는 방식
```bash
sudo visudo -f /etc/sudoers.d/90-automation
```
이렇게 설정
```bash
Defaults:automation !requiretty
automation ALL=(ALL) NOPASSWD: /usr/bin/systemctl, /usr/bin/journalctl, /usr/bin/docker
```
|설정|의미|
|-----|------|
|Defaults:automation !requiretty|TTY 없어도 sudo 허용|
|NOPASSWD|비밀번호 요구 안 함|
|/usr/bin/systemctl 등|실행 가능한 명령어 제한|

**✅ "ALL" 대신 필요한 명령만 명시 ← 이게 핵심 보안 포인트**

절대 이런 건 권하지 않는다.
```bash
automation ALL=(ALL) NOPASSWD: ALL     ❌ 위험
```

## 3️⃣ cron + sudo + NOPASSWD 구성 예시
cron에서 root 권한 명령을 써야 할 경우

### ① 스크립트
```bash
#!/bin/bash
sudo systemctl restart nginx
```
### ② sudoers
```bash
automation ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx
```
### ③ crontab
```bash
* * * * * /opt/scripts/restart_nginx.sh
```
위 설정으로 아래 효과를 얻을 수 있다.

- TTY 없음 ✅

- 비밀번호 없음 ✅

- 필요한 명령만 실행 ✅

👉 이게 자동화 표준 구조

## 4️⃣ Ansible이 requiretty / sudo를 싫어하는 이유
### Ansible은 내부적으로 이런 식으로 실행됨
```bash
ssh user@host "sudo command"
```
### 문제는
|설정|결과|
|-----|------|
requiretty 활성화|❌ 실패 (must have tty)|
|패스워드 필요|❌ 실패 (자동화 불가)|
|NOPASSWD + !requiretty|✅ 정상|

### 그래서 Ansible 공식 문서에서도 이렇게 권장
```bash
Defaults:ansible !requiretty
ansible ALL=(ALL) NOPASSWD: ALL
```
(실무에서는 ALL 대신 명령어 제한)

👉 그래서 클라우드 / 데브옵스 쪽에서 requiretty는 기본 OFF 상태임

## 5️⃣ 실무 서버 구성 패턴

### 실제 회사에서 가장 많이 쓰는 패턴은 아래 3가지 중 하나

#### 🔹 1. 운영 서버

- requiretty 꺼짐

- NOPASSWD + 명령어 제한

- audit 필수

#### ✅ 가장 일반적
---
#### 🔹 2. 고보안 서버

- requiretty 켜짐

- 접속은 root 불가

- bastion 서버 통해 사람 접근

#### ✅ 금융권 / 군 / 정부
---
#### 🔹 3. MSA / 컨테이너 환경

- sudo 자체를 안 씀

- kube 권한 / IAM Role 에서 해결

#### ✅ 최신 트렌드