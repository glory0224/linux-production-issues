# 🚨 Inode 부족으로 인한 디스크 사용 불가 문제
## 📌 문제 요약
서버의 디스크 용량(df-h) 은 충분해 보이는데도 다음과 같은 오류가 발생한다.
- 파일 생성 불가: No space left on device
- 웹 서버/애플리케이션 정상 동작 중단
- 로그 기록 불가로 인해 서비스 장애
- touch test.txt 조차 실패 (간단한 파일 생성 오류)

이는 디스크 공간 부족이 아니라, <b>inode 부족으로 인해 파일을 더 이상 생성할 수 없는 상황</b>이다. 

## 🧩 inode 부족이란?
inode 는 <b>파일 개수</b> 를 추적하는 구조체이다.
- 파일 하나 = inode 하나
- 디스크 용량이 남아 있어도 inode가 모두 소진되면 "파일을 더 이상 만들 수 없음"
  
### 특히 다음 상황에서 자주 발생한다.
- 작은 파일이 수십만 ~ 수백만 개 생성될 때
- 로그 시스템 오작동으로 초단위 파일이 무한 생성될 때
- 캐시 폴더가 비정상적으로 쌓일 때

## 🔎 실제 실무에서의 발생 사례
- cronjob 오작동으로 1초마다 작은 tmp 파일 생성
- 애플리케이션 버그로 빈 파일을 무한 생성
- 특정 디렉토리에 .sess_abc123 세션 파일 수십만 개 누적
- nginx/varnish cache 가 청소되지 않음
- mail queue 에 작은 이메일 파일이 쌓임
  
### 결과적으로 다음 명령에서 의심 가능
```bash
df -h                # 디스크 용량은 정상
df -i                # inode 사용량 100% (문제 원인)
```

## 🛠️ 해결 절차 개요
### 1) inode 사용량 확인
```bash
df -i
```
### 2) inode 를 많이 사용하는 디렉토리 찾기
```bash
for i in /*; do echo $i; find $i | wc -l; done
```

또는 더 빠르게
```bash
sudo du --inodes -d 3 /var | sort -n
```

### 3) 문제 디렉토리 내 작은 파일 삭제
```bash
rm -f /var/tmp/app/cache/*
```
or
```bash
find /var/tmp/app -type f -delete
```

### 4) 서비스 정상화 확인

## 🔁 재발 방지
- 로그/캐시 청소 cron 추가
- logrotate 정상 동작 확인
- tmp 파일 자동 정리 정책 적용
- 애플리케이션 파일 생성 방식 개선
- 캐시 TTL 설정 강화

## 🧪 모의적으로 inode 부족 상황 만들어보기
### 실제 운영 서버에서는 절대 하면 안되며, 테스트 서버 혹은 VM 환경에서만 실행

아래 과정은 아주 작은 파일을 수십만 개 생성하여 inode를 강제로 소모하는 방법

### 1) 테스트용 디렉토리 생성
```bash
mkdir ~/inode-test
cd ~/inode-test
```

### 2) 작은 파일 무한 생성 (inode 빠르게 소모)
<b>방법 A) touch 를 반복해 수십만 파일 생성</b>
```bash
for i in $(seq 1 500000); do
  touch file_$i
done
```
<b>방법 B) dd로 0바이트 파일을 대량 생성</b>
```bash
for i in {1..500000}; do
  dd if=/dev/null of=file$i 2>/dev/null
done
```
<b>방법 C) 더 빠른 방식 (parallel)</b>
```bash
seq 1 500000 | xargs -n 1 -P 8 touch
```

### 3) inode 소모 확인
```bash
df -i       # 특정 파티션의 IUse % 가 100% 근처로 올라간다.
```

### 4) 실패 테스트
```bash
# inode 가 부족하면 아래 모든 명령이 실패하게 된다.
touch test.txt # → No space left on device
mkdir newfolder # → No space left on device
```
### 5) 복구 (파일 삭제)
```bash
rm -f ~/inode-test/*
# 삭제 후 inode 사용률 확인
df -i
```