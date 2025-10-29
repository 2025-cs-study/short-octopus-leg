# 09 | 최소한 알고 있어야 할 서버 지식

## 1. 개발자와 서버

백엔드 개발자라고 해서 코딩만 잘하면 되는 것이 아님. 실제 서비스 운영 과정에서는 서버에서 발생하는 문제를 파악하고 해결해야 하는 상황이 자주 발생함.

- **장애 대응**: 서비스가 멈췄을 때 빠르게 원인을 찾을 수 있음.
- **디스크 관리**: 디스크가 가득 차서 서비스가 멈추는 것을 방지할 수 있음.
- **자동화**: 반복 작업을 자동으로 실행되게 할 수 있음.

## 2. SSH로 서버 접속하기

원격 서버에서 작업하려면 **SSH(Secure Shell)**로 접속해야 함. SSH는 안전하게 서버에 접속하고 명령어를 실행할 수 있게 해주는 프로토콜임.

### 기본 SSH 접속

```bash
# 기본 접속 (포트 22번 사용)
$ ssh username@server-ip

# 예시
$ ssh ubuntu@192.168.1.100

# 특정 포트로 접속
$ ssh -p 2222 username@server-ip

# 비밀번호 입력 후 접속 완료
```

### SSH 키 기반 접속

비밀번호 없이 안전하게 접속할 수 있음.

```bash
# 1. 로컬에서 SSH 키 생성 (처음 한 번만)
$ ssh-keygen -t rsa -b 4096

# 2. 공개키를 서버에 복사
$ ssh-copy-id username@server-ip

# 3. 이제 비밀번호 없이 접속 가능
$ ssh username@server-ip
```

### SSH 설정 파일로 편하게 접속하기

`~/.ssh/config` 파일에 서버 정보를 저장하면 간단하게 접속할 수 있음.

```bash
# ~/.ssh/config 파일 생성
$ vim ~/.ssh/config

# 아래 내용 추가
Host myserver
    HostName 192.168.1.100
    User ubuntu
    Port 22
    IdentityFile ~/.ssh/id_rsa

# 접속
$ ssh myserver
```

> SSH 키는 반드시 비밀번호로 보호하고, private key(id_rsa)는 절대 다른 사람과 공유하지 말아야 함.

## 3. OS 계정과 권한

리눅스 서버에는 여러 사용자가 작업할 수 있도록 **계정**을 제공함. 리눅스는 파일과 디렉터리에 대해 **읽기(r), 쓰기(w), 실행(x)** 권한을 제공함.

```bash
# 파일 권한 확인
$ ls -l myfile.txt
-rw-r--r-- 1 user group 1024 Oct 27 10:00 myfile.txt

# 권한 부여 (실행 권한 추가)
$ chmod +x myfile.txt

# 파일 소유자 변경
$ chown newuser myfile.txt
```

> root 계정은 모든 권한을 가지므로 일상적인 작업에는 일반 계정을 사용하고, 필요할 때만 `sudo` 명령어를 사용하는 것이 안전함.

## 4. 프로세스 확인하기

**프로세스**는 실행 중인 프로그램을 의미함. 서버에서는 웹 서버, DB, 애플리케이션 등 여러 프로세스가 동시에 실행됨.

```bash
# 실행 중인 프로세스 확인
$ ps aux

# 특정 프로세스 찾기
$ ps aux | grep java

# 실시간 모니터링
$ top
```

- **PID**: 프로세스를 식별하는 고유 번호
- **%CPU**: CPU 사용률
- **%MEM**: 메모리 사용률

```bash
# 정상 종료
$ kill PID

# 강제 종료
$ kill -9 PID
```

## 5. 백그라운드 프로세스

- **포그라운드**: 터미널에서 직접 실행하고 결과를 바로 확인함.
- **백그라운드**: 터미널과 독립적으로 실행함.

```bash
# 백그라운드로 실행 (&를 마지막에 추가)
$ java -jar myapp.jar &

# nohup으로 터미널 종료 후에도 계속 실행
$ nohup java -jar myapp.jar > app.log 2>&1 &

# 백그라운드 작업 목록 확인
$ jobs
```

> 실제 서비스 배포에는 systemd나 docker 같은 도구를 사용하는 것이 더 안정적임.

## 6. 로그 확인과 분석

서비스에 문제가 생겼을 때 가장 먼저 해야 할 일은 로그를 확인하는 것임.

### 로그 확인 명령어

```bash
# 로그 파일 전체 보기
$ cat /var/log/app/application.log

# 로그 파일 끝부분만 보기 (마지막 20줄)
$ tail -20 /var/log/app/application.log

# 실시간 로그 확인 (가장 많이 사용)
$ tail -f /var/log/app/application.log

# 로그 파일을 페이지 단위로 보기
$ less /var/log/app/application.log
```

### 로그에서 특정 내용 찾기

```bash
# 에러 로그만 찾기
$ grep "ERROR" /var/log/app/application.log

# 특정 시간대 로그 찾기
$ grep "2024-10-28 14:" /var/log/app/application.log

# 대소문자 구분 없이 찾기
$ grep -i "error" /var/log/app/application.log

# 특정 단어가 포함된 줄과 앞뒤 5줄 함께 보기
$ grep -A 5 -B 5 "NullPointerException" /var/log/app/application.log

# 여러 로그 파일에서 동시에 찾기
$ grep "ERROR" /var/log/app/*.log

# 에러 개수 세기
$ grep -c "ERROR" /var/log/app/application.log
```

> 로그 파일이 너무 크면 less 명령어로 열고, / 키를 눌러서 검색하면 편리함. n으로 다음 검색 결과, N으로 이전 검색 결과로 이동 가능함.

## 7. 디스크 용량 관리

서버에서 흔하게 발생하는 문제 중 하나가 **디스크 용량 부족**임. 디스크가 가득 차면 서비스 전체가 멈출 수 있음.

```bash
# 디스크 사용량 확인
$ df -h

# 특정 디렉터리 용량 확인
$ du -sh /var/log

# 가장 큰 디렉터리 찾기
$ du -h --max-depth=1 /var/log | sort -rh | head -5
```

**디스크 정리 방법**

```bash
# 오래된 로그 파일 삭제 (30일 이전)
$ find /var/log -type f -name "*.log" -mtime +30 -delete

# 로그 파일 압축
$ gzip /var/log/old.log
```

## 8. 시간 맞추기

서버 시간이 정확하지 않으면 로그 분석이 어렵고, 인증 토큰 검증에 문제가 생겨 크론 작업이 예상과 다른 시간에 실행됨.

```bash
# 현재 시간 확인
$ date

# 타임존 변경
$ timedatectl set-timezone Asia/Seoul

# NTP 서버와 동기화
$ timedatectl set-ntp true
```

> 모든 서버는 UTC 기준으로 시간을 관리하고, 애플리케이션에서 타임존을 변환하는 것이 일반적임.

## 9. alias 등록하기

**alias**는 자주 사용하는 긴 명령어를 짧은 명령어로 등록하여 사용하는 기능임. 반복 작업의 효율성을 크게 높일 수 있음.

```bash
# 현재 등록된 alias 확인
$ alias

# 임시 alias 등록
$ alias ll='ls -alF'
```

### 영구 alias 등록

`.bashrc` 파일에 등록하면 영구적으로 사용할 수 있음.

```bash
# ~/.bashrc 편집
$ vim ~/.bashrc

# 아래 내용 추가
alias ll='ls -alF'
alias ..='cd ..'
alias grep='grep --color=auto'

# 애플리케이션 관련
alias app-start='cd /home/app && ./start.sh'
alias app-log='tail -f /var/log/app/application.log'

# Docker 관련
alias dps='docker ps'
alias dlog='docker logs -f'

# 설정 적용
$ source ~/.bashrc
```

## 10. 네트워크 정보 확인

서버에서 발생하는 문제(외부 API 연동 실패, DB 연결 끊김, 포트 충돌 등)는 네트워크와 관련이 있음.

### 주요 네트워크 명령어

```bash
# 네트워크 인터페이스 정보 확인
$ ip addr show

# 특정 호스트 연결 확인
$ ping google.com

# 도메인의 IP 주소 확인
$ nslookup google.com

# 열려있는 포트 확인
$ netstat -tuln

# 특정 포트 사용 프로세스 확인
$ lsof -i :8080
```

### 연결 테스트

```bash
# TCP 포트 연결 테스트
$ telnet localhost 8080

# curl로 HTTP 연결 테스트
$ curl http://localhost:8080/health
```

## 참고 자료

1. [The Linux Command Line](https://linuxcommand.org/tlcl.php)
2. [생활코딩 - 리눅스](https://opentutorials.org/course/2598)
3. [grep 명령어 사용법](https://recipes4dev.tistory.com/157)
4. [Explain Shell - 명령어 설명](https://explainshell.com/)