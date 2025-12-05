# SSH KEY 기반 로그인 방식

## 1. 개념 요약

SSH Key 기반 로그인이란 비밀번호 대신 공개키(Public Key) + 개인키(Private Key) 를 사용하여 서버에 접속하는 방식이다.

### 핵심 포인트

- 공개키는 서버에 저장
- 개인키는 클라이언트(Mac, 개발자 PC)에만 존재
- 서버는 개인키를 요구하지 않음
- 개인키가 없으면 접근 불가

## 2. 작동 원리

접속 시 내부 흐름

1. 서버는 “이 문자열에 서명해보라”는 요청을 보낸다

2. 클라이언트는 개인키로 해당 문자열에 전자서명

3. 서버는 저장된 공개키로 서명을 검증

4. 일치하면 인증 성공

중요한 특징

- 개인키는 절대 서버로 전송되지 않는다

- 중간에 패킷을 가로채도 인증 불가

- 비밀번호를 브루트포스로 시도하는 공격 구조 자체가 성립하지 않음

## 3. 계정 구조와 키 저장 위치

SSH 키는 서버 전체가 아니라 특정 사용자 계정에 귀속된다.
```bash
/home/사용자명/.ssh/authorized_keys
```
예시
```bash
ssh-copy-id dev@192.168.0.10
```
→ dev 계정의
```bash
/home/dev/.ssh/authorized_keys
```
에 공개키 추가됨

**실무에서 일반적인 계정 구조**
|계정|용도|
|---------------|----------------------|
|ubuntu / ec2-user|	기본 계정 (sudo 가능)
|deploy | 	배포전용|
|dev1 / dev2	|개발자 개인 계정|
|root	|직접 로그인 금지|

권장 원칙

- root 직접 로그인 금지

- 일반 계정 + sudo 사용

- 사용자별 개별 키 등록

### 4. 패스워드 로그인 방식과 비교
#### 길이 차이
|방식|복잡도|
|---------------|----------------------|
|비밀번호| 8 ~ 16자|
|SSH key|2048~4096비트|

#### 전송 방식
|방식|네트워크 전송|
|---------------|----------------------|
|Password |	비밀번호 전송|
|Key | 	서명값만 전송(키 전송 x)|

→ 본질적으로 키 방식이 안전

### 5. key 기반 로그인 (Mac)
### 5.1 Mac에서 SSH 키 생성
```bash
ssh-keygen -t rsa -b 4096 -C "mac-key"
```
#### 생성 파일
|파일|설명|
|---------------|----------------------|
|~/.ssh/id_rsa |	개인키|
|~/.ssh/id_rsa.pub | 	공개키|

### 5.2 서버에 공개키 등록
#### 자동방식
```bash
ssh-copy-id user@192.168.xx.xx
```
#### 수동방식
```bash
cat ~/.ssh/id_rsa.pub | ssh user@192.168.xx.xx \
"mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```
### 5.3 비밀번호 로그인 비활성화
```bash
sudo nano /etc/ssh/sshd_config

# 설정 변경 
PasswordAuthentication no
PubkeyAuthentication yes
# 적용
sudo systemctl restart ssh
# 클라이언트에서 비밀번호 없이 접속 시도
ssh user@192.168.xx.xx
```

### 6. 실무 운영 보안 설정

권장 구성
```bash
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers dev deploy
```
추가 보안 강화

- fail2ban 설치

- SSH 포트 변경 또는 IP 제한

- bastion 서버 사용

- 개인키에 passphrase 설정

- 키 유출 시 즉시 폐기