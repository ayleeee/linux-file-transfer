# linux-file-transfer
이 레포지토리는 개발 서버와 운영 서버 간의 파일 전송 방법에 대해 다룹니다.
## Linux 서버 간 파일 공유 방법

### 1단계: SSH 설정

서버 간 안전한 파일 전송을 위해 먼저 로컬 서버와 원격 서버 간의 SSH 접속을 설정해야 합니다.

1. **로컬 서버에서 SSH 키 생성** (파일 전송을 시작하는 서버):
   ```bash
   ssh-keygen -t rsa -b 4096
   ```
2. **생성한 SSH 키를 원격 서버에 복사**:
   ```bash
   ssh-copy-id [사용자명]@[원격_IP]
   ```
   이 명령어는 로컬 서버의 공개 키를 원격 서버로 복사하여 암호 없이 SSH 접속이 가능하게 만듭니다.

3. **원격 서버로 SSH 접속 확인**:
   ```bash
   ssh [사용자명]@[원격_IP]
   ```
   성공하면 비밀번호 입력 없이 원격 서버에 접속할 수 있습니다.

4. **원격 서버에서 SSH 키가 제대로 추가되었는지 확인**:
   ```bash
   cat ~/.ssh/authorized_keys
   ```
   공개 키가 이 파일에 추가되어 있어야 합니다.

### 2단계: `scp` 명령어로 파일 전송

SSH 설정이 완료되면, `scp`(secure copy) 명령어를 사용하여 서버 간에 파일을 전송할 수 있습니다.

#### 단일 파일을 원격 서버로 전송하기

- 로컬 서버에서 원격 서버로 단일 파일을 보내려면 다음 명령어를 사용합니다:
   ```bash
   scp [파일명] [사용자명]@[원격_IP]:[목적지_경로]
   ```
   - `[파일명]`은 전송하려는 파일의 이름입니다.
   - `[사용자명]`은 원격 서버의 사용자 이름입니다.
   - `[원격_IP]`는 원격 서버의 IP 주소입니다.
   - `[목적지_경로]`는 파일을 저장할 원격 서버의 경로입니다.

# 프로젝트 워크플로: 개발 서버와 운영 서버 간 JAR 파일 자동 전송 및 업데이트

이 프로젝트에서는 GitHub 리포지토리의 변동 사항을 자동으로 감지하여 개발 서버와 운영 서버 간에 JAR 파일을 전송하고, 운영 서버에서 애플리케이션을 업데이트하는 시스템을 구축합니다.

## 1. 시스템 개요

1. **개발 서버**와 **운영 서버**가 존재합니다.
2. GitHub에 코드의 변동 사항이 생기면 **Jenkins 파이프라인**을 통해 개발 서버에 있는 `SpringApp-0.0.1-SNAPSHOT.jar` 파일이 자동으로 업데이트됩니다.
3. **개발 서버**에서 `SpringApp-0.0.1-SNAPSHOT.jar` 파일의 변동 사항을 감지하는 **쉘 스크립트**가 실행됩니다.
4. 이 스크립트는 수정된 JAR 파일을 **운영 서버**로 전송합니다.
5. **운영 서버**에서는 기존에 실행 중이던 `SpringApp-0.0.1` 파일이 덮어씌워지고, 새로운 JAR 파일이 실행됩니다.

## 2. 파일 감지 및 전송 스크립트

개발 서버에서 JAR 파일이 수정되었을 때 이를 감지하고 운영 서버로 전송하는 스크립트입니다.

### 1. `autorunning.sh` 스크립트
```bash
#!/bin/bash

JAR_FILE="./SpringApp-0.0.1-SNAPSHOT.jar"  
FILENAME=$(basename "$JAR_FILE")  # 파일 이름만 추출

# 원격 서버 정보 설정
REMOTE_USER="username"
REMOTE_SERVER="10.0.2.15"
REMOTE_PATH="/home/username/updateJAR"

# 파일 수정 감지 및 원격 서버로 전송
inotifywait -m -e close_write "$JAR_FILE" |
while read -r directory events; do
    CURRENT_TIME=$(date +%s)

    # 마지막 실행 후 지정된 시간이 지났는지 확인
    if (( CURRENT_TIME - LAST_RUN > COOLDOWN )); then
        echo "$(date): $FILENAME 파일이 수정되었습니다."  # 수정 시간 로그 추가

        # 수정된 JAR 파일을 원격 서버로 전송
        echo "$(date): $FILENAME 파일을 원격 서버로 전송 중..."
        scp "$JAR_FILE" "$REMOTE_USER@$REMOTE_SERVER:$REMOTE_PATH/$FILENAME"

        # 마지막 실행 시간 업데이트
        LAST_RUN=$CURRENT_TIME
    else
        echo "$(date): 쿨다운 기간 중입니다."
    fi
done
```

이 스크립트는 inotifywait를 사용하여 SpringApp-0.0.1-SNAPSHOT.jar 파일의 수정 여부를 감지합니다. 파일이 수정되면 지정된 원격 서버로 scp 명령어를 통해 파일이 전송되고, 쿨다운 시간(여기서는 10초)이 설정되어 반복 실행을 방지합니다.

### 2. 파일 실행 및 쿨다운 처리
이 스크립트는 JAR 파일이 감지되면 별도의 실행 스크립트 (autorunning.sh)를 호출하여 운영 서버에 파일을 전송하고, 해당 서버에서 애플리케이션을 업데이트 및 재시작합니다.

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
        echo "$(date): $filename"  
        bash "$SH_FILE"
        LAST_RUN=$CURRENT_TIME
    else
        echo "$(date)"
    fi
done

```
### 3. 결론
이 워크플로는 GitHub에서 코드 변동 사항을 감지하여 개발 서버와 운영 서버 간의 파일 동기화를 자동화합니다. 이로 인해 수동으로 JAR 파일을 전송하고 애플리케이션을 업데이트하는 번거로움을 줄이고, 운영 환경에서 최신 코드를 항상 유지할 수 있습니다.
