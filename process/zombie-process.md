# 🧟 Zombie Process (좀비 프로세스)

## 🔥 문제 요약 (Symptoms)
- `ps`, `top`, `htop` 등에서 상태가 `Z`(zombie)로 표시되는 프로세스가 보인다.
- 프로세스는 종료되었지만 프로세스 테이블(PID)이 해제되지 않음.
- CPU 사용량은 크게 오르지 않으나 PID/프로세스 테이블이 누적되면 시스템 자원 고갈(새 프로세스 생성 실패) 가능.
- 많은 좀비가 쌓이면 서비스 재시작 또는 시스템 재부팅 필요.

---

## 🧩 좀비 프로세스가 생기는 이유 (Root Causes)
- 자식 프로세스가 종료(`exit`)했지만 부모가 `wait()` 또는 `waitpid()` 로 자식의 종료 상태를 수거(reap)하지 않음.
- 부모가 의도적으로(버그로) 자식 종료를 방치하거나, 부모가 죽을 때까지 기다리는 구조.
- 데몬 / 백그라운드 프로그램이 자식 관리를 잘못 구현함 (SIGCHLD 처리 누락).
- 잘못된 fork/exec 패턴 (예: 데몬화 시 double-fork 패턴 누락)

**핵심:** 좀비는 메모리·CPU를 대량 소모하진 않지만, PID 슬롯을 점유해 시스템에 문제가 됨.

---

## 🔎 진단(How to detect)

### 1) 전체 프로세스 중 Zombie 확인
```bash
ps aux | awk '$8=="Z"{print $0}'
# 또는
ps -eo pid,ppid,stat,cmd | grep ' Z'
```
### 2) top / htop 에서 Z 상태 보기
- top : S 열 또는 STAT 열에 Z 로 표시됨
- htop : 상태 컬럼에 Z 표시 가능

### 3) 좀비 개수 확인 (빠르게)
```bash
ps -eo stat | grep -c Z
```

### 4) 좀비의 부모 프로세스(PPID) 확인
```bash
ps -eo pid,ppid,stat,cmd | awk '$3=="Z"{print "zombie-pid="$1" ppid="$2" cmd="$4}'
# 또는 zombie PID가 1234라면
ps -o pid,ppid,stat,cmd -p 1234
```

---
## 🛠️ 즉시 해결법
### 1. 부모 프로세스가 정상적으로 자식을 처리하도록 변경
- 코드 수정 : 부모에서 wait()/waitpid() 호출 또는 SIGCHLD 처리기 추가.
### 2. 부모 프로세스 재시작(또는 종료)
- 부모를 kill(graceful) → 부모가 종료되면 자식(좀비)은 init(PID 1)로 입양되고 init이 즉시 재수거(reap)함.
```bash
# 부모 프로세스 PID가 2222 라면
sudo kill -TERM 2222
# 또는 강제
sudo kill -9 2222
```
- 주의: 부모를 강제 종료하면 서비스 영향이 있을 수 있음. 가능한 graceful restart 권장.
### 3. 코드에서 SIGCHLD 무시 설정 (임시/특정 상황)
- 부모 프로세스에서 SIGCHLD를 SIG_IGN로 설정하면 많은 플랫폼에서 자동으로 자식 자동 재수거 동작이 됨(플랫폼 종속적이므로 주의).
- 권장하지 않음(디테일한 관리 필요).

---
## ✅ 재발 방지 (Prevention / Best practices)

- 부모 프로세스에서 모든 자식 프로세스에 대해 waitpid() 호출(논블로킹 WNOHANG과 함께 사용 가능).

- 데몬화 시 double-fork 패턴 또는 적절한 부모-자식 리퍼런스 관리 적용.

- 자식 프로세스 생성을 관리하는 라이브러리(예: Java의 ProcessBuilder)를 사용하면 자동적으로 wait/cleanup 처리.

- SIGCHLD 핸들러를 적절히 등록해 비동기적으로 자식 종료 처리.

- 모니터링으로 좀비 수 임계값 설정(예: > 10 → 경고), 자동 알람/대응 스크립트 마련.

---

## 🧪 로컬(모의) 테스트: UTM Ubuntu (Mac 호스트)에서 좀비 만들기
### 1. 방법 1 - C 코드 (create_zombie.c)
```c
// create_zombie.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    pid_t pid = fork();
    if (pid < 0) {
        perror("fork failed"); exit(1);
    } else if (pid == 0) {
        // child
        printf("Child (%d) exiting\n", getpid());
        _exit(0); // child exits immediately -> becomes zombie if parent doesn't wait
    } else {
        // parent
        printf("Parent (%d) sleeping. Child pid=%d\n", getpid(), pid);
        sleep(300); // 부모가 자식의 wait를 하지 않고 오래 잠자면 좀비가 됨
        printf("Parent exiting\n");
    }
    return 0;
}
```
### 2. 컴파일 & 실행 (Ubuntu VM)
```bash
gcc create_zombie.c -o create_zombie
./create_zombie &
# 부모는 sleep 상태, child는 exit -> child는 좀비가 됨
```
### 3. 좀비 확인
```bash
ps -ef | grep Z
ps -eo pid,ppid,stat,cmd | awk '$3=="Z"{print $0}'
# 또는
ps aux | awk '$8=="Z"{print $0}'
```

### 4. 좀비가 사라지게 하기 (정리)
- 부모 프로세스를 종료(또는 기다렸다가 부모가 exit 하면 init이 reaper가 되어 좀비를 정리)
```bash
ps -ef | grep create_zombie   # 부모 PID 확인
kill <parent-pid>
# kill 하면 init이 자식을 인계받아 zombie를 재수거 -> 좀비가 사라짐
```

---

### 1. 방법 2 — 파이썬으로 좀비 만들기 (빠른 방법)
(파이썬으로도 fork 가능)
```python
# create_zombie.py
import os
pid = os.fork()
if pid == 0:
    # child
    print("child exiting", os.getpid())
    os._exit(0)
else:
    # parent
    print("parent sleeping", os.getpid(), "child", pid)
    import time; time.sleep(300)
```
### 2. 실행
```bash
python3 create_zombie.py &
# 확인
ps -eo pid,ppid,stat,cmd | grep ' Z'
```
### 3. 부모 프로세스 찾아서 종료
---
### 1. 방법 3 - 대량 좀비를 만들어 PID 고갈 시뮬레이션 (주의: 시스템 영향)

교육 목적 — 정상 환경(작은 VM)에서만.
```c
// many_zombies.c  -- 많은 수의 좀비를 만드는 부모
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

int main() {
    for (int i = 0; i < 10000; i++) {
        pid_t pid = fork();
        if (pid == 0) {
            _exit(0); // child exits immediately => zombie
        } else if (pid > 0) {
            // 부모가 wait를 하지 않음 -> 다음 반복으로 계속 좀비 생성
            usleep(1000);
        } else {
            perror("fork"); exit(1);
        }
    }
    sleep(600);
    return 0;
}
```
- 이 코드는 테스트중인 VM의 PID/프로세스 테이블을 빠르게 채울 수 있음(주의).

- 소규모로 먼저 테스트(예: 100개)로 해보고 진행.

---
## 🔧 자주 쓰는 진단/정리 명령 모음
```bash
# 좀비만 목록
ps -eo pid,ppid,stat,cmd | awk '$3=="Z"{print}'

# 특정 부모 프로세스가 만든 좀비 찾기
PARENT=2222
ps -eo pid,ppid,stat,cmd | awk -v p=$PARENT '$2==p && $3=="Z"{print}'

# 간단히 좀비 수 카운트
ps -eo stat | grep -c Z

# 부모 PID 얻기
ps -o pid,ppid,stat,cmd -p <zombie-pid>

# 좀비가 소멸하지 않을 때 (임시 조치) : 부모 종료
kill -TERM <parent-pid>
# 또는 강제
kill -9 <parent-pid>

```
---
## 🧠 실무 팁 
- 좀비 자체는 즉시 치명적이지 않지만 누적되면 시스템 문제(새 PID 생성 실패 등)를 유발.

- 원인 대부분은 부모가 자식을 wait(수거)하지 않음 → 코드 수정을 통해 해결.

- 긴급 대응은 부모 프로세스 재시작(또는 종료). 재발 방지: SIGCHLD 핸들러/적절한 wait 정책/프로세스 관리 도구 사용.

- 데몬/백그라운드 로직을 작성할 때는 항상 자식 정리 패턴(예: double-fork, waitpid loop)을 확인.