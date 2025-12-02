# Permission denied — Script 실행 불가 문제

## 🔥 문제 요약 (Symptoms)
`./myscript.sh` 실행 시
```bash
./myscript.sh: Permission denied
```

- 또는 서비스( systemd )에서 실행 시 권한 오류로 실패
- 스크립트 파일이 있는데 실행되지 않음(그러나 `cat` 등은 가능)
- 다른 계정(예: sudo가 아닌 계정)으로 실행 시 실패

---

## 🧩 대표 원인 (Causes)
1. **실행 권한 없음 (no +x)**  
2. **파일 소유자/그룹 문제** (권한이 소유자에게만 있음)  
3. **파일 시스템이 `noexec`로 마운트됨** (예: /tmp 또는 외부 파티션)  
4. **Shebang(#!)에 지정한 인터프리터가 없음** → `bad interpreter` 또는 permission-like 에러  
5. **CRLF (Windows 줄바꿈) 때문에 해석 불가** (보통 “bad interpreter”가 뜨지만 때때로 혼동됨)  
6. **ACL / SELinux / AppArmor 정책에 의해 차단**  
7. **디렉토리 권한 문제** (디렉토리에서 실행 권한 x가 없음 → 진입 불가)  
8. **스크립트를 실행하려는 계정에 실행 허가가 없음 (sudoers / restricted shell)**  
9. **NFS 공유에서 root_squash 또는 export 옵션**  
10. **실행 파일이 이진이지만 실행 권한 없음 / 아키텍처 불일치** 

---

## 🔍 진단(How to investigate)

### 1) 파일 권한과 소유자 확인
```bash
ls -l myscript.sh
stat myscript.sh
```
### 2) 파일 타입 확인 (스크립트인지 바이너리인지)
```bash
file myscript.sh
```
### 3) 디렉토리 권한 확인
```bash
ls -ld .
```
### 4) 마운트 옵션 확인 (noexec 여부)
```bash
mount | grep $(df --output=target myscript.sh | tail -1)
# 또는
findmnt -no OPTIONS $(dirname $(readlink -f myscript.sh))
```
### 5) ACL 확인
```bash
getfacl myscript.sh
```
### 6) SELinux / AppArmor 검사 (배포판에 따라)
- Ubuntu(AppArmor)
```bash
sudo aa-status
sudo journalctl -t apparmor
```
- SELinux (RHEL 계열)
```bash
sestatus
ausearch -m avc -ts recent
```
### 7) shebang(첫 줄) 확인
```bash
head -n 1 myscript.sh
```
예: #!/usr/bin/env bash 또는 #!/bin/bash 가 유효한지 확인
### 8) 시스템 로그 확인 (서비스 실패 시)
```bash
journalctl -u your-service -b --no-pager
```

---

## 🛠 해결 방법 (Fixes)
### A. 실행 권한 부여
```bash
chmod +x myscript.sh
./myscript.sh
```
### B. 소유권 변경 (필요 시)
```bash
sudo chown user:group myscript.sh
```
### C. 마운트 옵션 문제 해결 (noexec)
- 즉시 테스트(테스트 파티션에서 noexec 해제)
```bash
sudo mount -o remount,exec /path/to/mountpoint
```
- 영구적 수정: /etc/fstab에서 해당 파티션의 옵션에서 noexec 제거 후 재마운트

  > 주의: 보안 정책상 /tmp 등을 noexec로 유지하는 경우가 많음. 그럴 땐 bash myscript.sh 처럼 인터프리터를 명시해 실행
### D. shebang 문제 해결
- 유효한 shebang으로 수정
```bash
#!/usr/bin/env bash
```
- 직접 인터프리터로 실행
```bash
bash myscript.sh
python3 myscript.py
```
### E. CRLF 문제 (Windows EOL)
```bash
dos2unix myscript.sh
# 또는
sed -i 's/\r$//' myscript.sh
```
### F. ACL 수정
```bash
setfacl -m u:youruser:rx myscript.sh
getfacl myscript.sh
```
### G. AppArmor / SELinux 정책 문제
- AppArmor: 문제되는 프로파일을 수정하거나, 임시 비활성화로 원인 확인
```bash
sudo aa-complain /usr/bin/your-app
```
- SELinux: audit 로그 확인 후 적절한 boolean 또는 정책 수정

### H. systemd 서비스에서 실패 시
- 서비스 유닛 파일 내에서 ExecStart에 절대경로 + 실행 권한 확인
- User= 설정이 해당 계정에 권한이 있는지 확인
- ProtectSystem/ReadOnlyPaths 등 sandbox 옵션 확인

---

## 🧪 로컬(UTM Ubuntu)에서 문제 재현 및 테스트 (Step-by-step)
> **주의**: 테스트는 로컬 VM(UTM)에서만 하세요. 운영 서버에서 noexec 재마운트나 ACL 조작은 서비스 영향이 큽니다.

### 환경: UTM Ubuntu (접속은 Mac 터미널에서 ssh)

### 테스트 케이스 1 — 실행 비트 없음 (가장 흔함)
#### 1. 스크립트 생성
```bash
vi myscript.sh
# 내용 작성

#!/bin/bash
echo "hello"
```
#### 2. 의도적으로 실행 권한 제거
```bash
chmod 644 myscript.sh
ls -l myscript.sh
# -rw-r--r-- 1 user user ... myscript.sh
```
#### 3. 실행 시도
```bash
./myscript.sh
# 결과: -bash: ./myscript.sh: Permission denied
```
#### 4. 해결
```bash
chmod +x myscript.sh
./myscript.sh  # hello
```
### 테스트 케이스 2 — 인터프리터로 직접 실행(임시 우회 방법)
실행되는 파일의 비트가 없더라도
```bash
bash myscript.sh   # 정상 실행 (실행 비트를 안 줘도 작동)
```
→ 따라서 Permission denied는 ./script 형태로 실행하려고 할 때 주로 발생

### 테스트 케이스 3 — 파일 시스템이 noexec 로 마운트된 경우
#### 1. 임시 마운트 포인트 만들기
```bash
sudo mkdir -p /mnt/test_noexec
sudo mount -o loop,noexec /path/to/some.img /mnt/test_noexec  # 예시(이미지 필요)
# 간단히 재마운트 현재 폴더 (주의: 안전 테스트 환경에서만)
# 예: /home/user/mountpoint 이 있다면
sudo mount -o remount,noexec /home/youruser/mountpoint
```
(간단한 방법 — tmpfs 생성하여 noexec 마운트)
```bash
sudo mkdir /mnt/tmp_noexec
sudo mount -t tmpfs -o size=10M,noexec tmpfs /mnt/tmp_noexec
```
#### 2. 스크립트 복사
```bash
cp myscript.sh /mnt/tmp_noexec/
cd /mnt/tmp_noexec
chmod +x myscript.sh
./myscript.sh
# 결과: -bash: ./myscript.sh: Permission denied
```
#### 3. 우회/해결
```bash
bash myscript.sh   # 동작함
sudo mount -o remount,exec /mnt/tmp_noexec
./myscript.sh     # 이제 실행됨
```
### 테스트 케이스 4 — CRLF 문제로 실행 실패(보통 "bad interpreter" 메시지)
#### 1. Windows 스타일 줄바꿈으로 파일 만들기(모의)
```bash
printf '#!/bin/bash\r\n echo hi\r\n' > winscript.sh
chmod +x winscript.sh
./winscript.sh
# 결과: /bin/bash^M: bad interpreter: No such file or directory
```
보틍은 위와 같은 에러가 발생하나 아래와 같은 에러가 발생할 수 있다.
```bash
required file not found
```
- 왜 위와 같은 에러가 발생하나?
- 이게 뜬 이유는 거의 확실하게 이것 중 하나일 가능성이 있다.

1. winscript.sh 파일 자체가 깨진 상태

2. /bin/bash 경로가 깨졌거나 심볼릭 링크 문제

3. 실제 에러는 bad interpreter인데 터미널에 다르게 표시됨

가장 가능성 높은 건 → ✅ Hidden CR 문자 때문
```bash
cat -A winscript.sh
# 결과는 아래
#!/bin/bash^M$
 echo hi^M$
```
이 ^M 이 바로 윈도우 줄바꿈(CRLF = \r\n)이며
bash 는 실제로 이렇게 해석한다.
```bash
/bin/bash\r
```
**그런 경로는 없으니까 →"파일이 없네? → required file not found"**
#### 2. 해결 - dos2unix 사용
```bash
sudo apt update
sudo apt install dos2unix
dos2unix winscript.sh
# 다시 실행한다.
./winscript.sh
```
### 테스트 케이스 5 — 디렉토리 권한 문제
#### 1. 디렉토리에서 실행 권한 제거
```bash
mkdir testdir
chmod 700 testdir           # 소유자만 접근 가능
# 만약 다른 계정으로 접근 시도하면 실행 불가 -> Permission denied
```
#### 2. 다른 유저가 실행 시도 시 확인 (su 또는 ssh로 다른 계정에서)
```bash
# as other user
cd testdir
# 결과: Permission denied
```
### 3. 테스트 케이스 6 — ACL / getfacl 예시
#### 1. user1 의 home 디렉토리 하위에 myscript.sh 생성
#### 2. user2 계정 생성
```bash
sudo useradd young        # 홈 없음
sudo useradd -m young      # 홈까지 생성
```
#### 3. user1 에서 acl 및 권한 설정
```bash
# user1 기준

chmod 700 /home/user1
chmod 700 /home/user1/myscript.sh

# cd 로 해당 user1 홈 디렉토리에 접근하기 위해 acl 허용
setfacl -m u:user2:--x /home/user1
# myscript 내부에 #!/bin/bash 때문에 읽기 권한도 꼭 함께 주어야한다.
setfacl -m u:user2:rx /home/user1/myscript.sh
```
#### 4. user2 에서 접근
```bash
su - user2
/home/user1/myscript.sh   ✅ 실행됨
ls /home/user1            ❌ 안 보임
```

기본 권한만으로는 문제가 해결되지 않을 때,
ACL 설정으로 인해 접근이 차단되거나 허용되는 케이스가 있다.

1. 기본 권한 제거
chmod 700 myscript.sh

2. 특정 사용자에게 ACL 부여
setfacl -m u:otheruser:--x myscript.sh

3. ACL 확인
getfacl myscript.sh

4. ls -l 에 **+ 표시 확인**
-rwx------+

### 🔁 진단용 명령 모음
```bash
# 권한/소유자
ls -l myscript.sh
stat myscript.sh

# 파일 시스템 마운트 옵션
findmnt -no SOURCE,TARGET,FSTYPE,OPTIONS $(dirname $(readlink -f myscript.sh))

# ACL
getfacl myscript.sh

# apparmor 상태(Ubuntu)
sudo aa-status

# systemd 서비스 로그
sudo journalctl -u your-service -b

# 임시로 인터프리터로 실행(우회)
bash myscript.sh
sh myscript.sh
```

✅ 요약 체크리스트 (운영 시 빠르게 확인할 항목)

- ls -l로 실행권한(x) 확인

- file로 타입 확인(스크립트/바이너리)

- head -n1로 shebang 확인

- findmnt/mount로 noexec 확인

- getfacl로 ACL 확인

- AppArmor/SELinux 로그 확인

- systemd 유닛(실행 사용자) 권한 확인