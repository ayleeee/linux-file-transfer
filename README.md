

# Linux 파일 전송

이 레포지토리는 개발 서버와 운영 서버 간의 파일 전송 방법에 대해 다룹니다.

## 목차
1. [Linux 서버 파일 공유 방법](#linux-서버-파일-공유-방법)
2. [프로젝트 워크플로: 개발 서버와 운영 서버 간 JAR 파일 자동 전송 및 업데이트](#프로젝트-워크플로-개발-서버와-운영-서버-간-jar-파일-자동-전송-및-업데이트)
    - [시스템 개요](#1-시스템-개요)
    - [파일 감지 및 전송 스크립트](#2-파일-감지-및-전송-스크립트)
        1. [send.sh 스크립트](#1-sendsh-스크립트)
        2. [change.sh 스크립트](#2-changesh-스크립트)
        3. [autorunning.sh 스크립트](#3-autorunningsh-스크립트)
3. [결론](#3-결론)

---

## Linux 서버 파일 공유 방법

### 1단계: SSH 설정

서버 간 안전한 파일 전송을 위해 먼저 로컬 서버와 원격 서버 간의 SSH 접속을 설정해야 합니다.

1. **로컬 서버에서 SSH 키 생성**:
   ```bash
   ssh-keygen -t rsa -b 4096
   ```
2. **생성한 SSH 키를 원격 서버에 복사**:
   ```bash
   ssh-copy-id [사용자명]@[원격_IP]
   ```
   이 명령어는 공개 키를 원격 서버로 복사하여 암호 없이 SSH 접속을 가능하게 만듭니다.

3. **원격 서버로 SSH 접속 확인**:
   ```bash
   ssh [사용자명]@[원격_IP]
   ```
   비밀번호 입력 없이 원격 서버에 접속할 수 있으면 성공입니다.

4. **SSH 키가 제대로 추가되었는지 확인**:
   ```bash
   cat ~/.ssh/authorized_keys
   ```
   공개 키가 이 파일에 나와 있어야 합니다.

### 2단계: `scp` 명령어로 파일 전송

SSH 설정이 완료되면, `scp` (secure copy) 명령어를 사용해 서버 간 파일을 전송할 수 있습니다.

#### 단일 파일을 원격 서버로 전송하기

- 로컬 서버에서 원격 서버로 파일을 전송하려면 아래 명령어를 사용합니다:
   ```bash
   scp [파일명] [사용자명]@[원격_IP]:[저장_경로]
   ```
   - `[파일명]`: 전송할 파일의 이름
   - `[사용자명]`: 원격 서버의 사용자명
   - `[원격_IP]`: 원격 서버의 IP 주소
   - `[저장_경로]`: 파일을 저장할 원격 서버의 경로

---

## 프로젝트 워크플로: 개발 서버와 운영 서버 간 JAR 파일 자동 전송 및 업데이트

이 프로젝트는 GitHub 리포지토리에서 발생한 변동 사항을 감지하여 개발 서버와 운영 서버 간에 JAR 파일을 자동으로 전송 및 업데이트하는 시스템을 구축합니다.

### 1. 시스템 개요

1. **개발 서버**와 **운영 서버**가 존재합니다.
2. GitHub에 변경 사항이 발생하면 **Jenkins 파이프라인**이 자동으로 개발 서버의 `SpringApp-0.0.1-SNAPSHOT.jar` 파일을 업데이트합니다.
3. **개발 서버**에서 JAR 파일의 변동 사항을 감지하는 **쉘 스크립트**가 실행됩니다.
4. 스크립트는 수정된 JAR 파일을 **운영 서버**로 전송합니다.
5. **운영 서버**는 기존에 실행 중이던 `SpringApp-0.0.1` 파일을 새로운 파일로 덮어쓰고, 애플리케이션을 재실행합니다.

---

### 2. 파일 감지 및 전송 스크립트

[개발 서버]JAR 파일이 수정되었을 때 이를 감지하고 운영 서버로 전송하는 스크립트입니다.

#### 1. `send.sh` 스크립트

```bash
#!/bin/bash

JAR_FILE="./SpringApp-0.0.1-SNAPSHOT.jar"
FILENAME=$(basename "$JAR_FILE")

# 원격 서버 정보
REMOTE_USER=""
REMOTE_SERVER=""
REMOTE_PATH="/home/username/updateJAR"

# 변경 사항 감지 및 파일 전송
inotifywait -m -e close_write "$JAR_FILE" |
while read -r directory events; do
    CURRENT_TIME=$(date +%s)

    if (( CURRENT_TIME - LAST_RUN > COOLDOWN )); then
        echo "$(date): $FILENAME 파일이 수정되었습니다."

        # 수정된 JAR 파일을 원격 서버로 전송
        echo "$(date): $FILENAME 파일을 원격 서버로 전송 중..."
        scp "$JAR_FILE" "$REMOTE_USER@$REMOTE_SERVER:$REMOTE_PATH/$FILENAME"

        LAST_RUN=$CURRENT_TIME
    else
        echo "$(date): 쿨다운 중입니다."
    fi
done
```

이 스크립트는 `inotifywait`를 사용하여 `SpringApp-0.0.1-SNAPSHOT.jar` 파일의 변경 사항을 감지합니다. 파일이 수정되면 이를 원격 서버로 전송하고, 쿨다운 시간(10초)이 설정되어 반복 실행을 방지합니다.

#### 2. `change.sh` 스크립트

[운영 서버] 이 스크립트는 파일 변경을 감지한 후, 별도의 스크립트(`autorunning.sh`)를 호출하여 파일을 운영 서버로 전송하고 실행합니다.

```bash
#!/bin/bash

JAR_FILE="./SpringApp-0.0.1-SNAPSHOT.jar"
SH_FILE="./autorunning.sh"
COOLDOWN=10
LAST_RUN=0

inotifywait -m -e close_write "$JAR_FILE" |
while read -r directory events filename; do
    CURRENT_TIME=$(date +%s)
    if (( CURRENT_TIME - LAST_RUN > COOLDOWN )); then
        echo "$(date): $filename 파일이 감지되었습니다."
        bash "$SH_FILE"
        LAST_RUN=$CURRENT_TIME
    else
        echo "$(date): 쿨다운 중입니다."
    fi
done
```
#### 2. `autorunning.sh` 스크립트
```bash
#!/bin/bash

if  lsof -i :8999 > /dev/null; then
  kill -9 $(lsof -t -i:8999)
  echo '정상적으로 종료되었습니다.'
fi

nohup java -jar SpringApp-0.0.1-SNAPSHOT.jar > app.log 2>&1 &
echo "배포완료 및 재 실행됩니다."
```
---

### 3. 결론

이 워크플로는 GitHub에서 발생한 변경 사항을 감지하여 개발 서버와 운영 서버 간의 파일 동기화를 자동화합니다. 이를 통해 JAR 파일 전송 및 애플리케이션 업데이트를 수동으로 처리할 필요가 없어지며, 운영 환경에서 최신 코드를 항상 유지할 수 있습니다.
