# 🛡️ Docker 환경의 데이터 보호와 고가용성 구축 방법

이 저장소는 Docker 환경에서 MySQL 데이터베이스의 안정적인 백업 방법과 고가용성 구성에 대한 가이드를 제공합니다.

## 📋 목차

- [개요](#개요)
- [시스템 요구사항](#시스템-요구사항)
- [기본 MySQL 및 2개 이상의 app 컨테이너 설정](#기본-MySQL-및-2개-이상의-app-컨테이너-설정)
- [데이터베이스 백업 전략](#데이터베이스-백업-전략)
  - [자동화된 백업 스크립트](#자동화된-백업-스크립트)
  - [크론 작업 설정](#크론-작업-설정)
- [고가용성 구성](#고가용성-구성)
  - [마스터-슬레이브 복제 설정](#마스터-슬레이브-복제-설정)
  - [자동 장애 복구(Failover)](#자동-장애-복구failover)
  - [로드 밸런싱 구성](#로드-밸런싱-구성)

## 개요

현대 애플리케이션에서 데이터베이스는 핵심 구성 요소이며, 특히 프로덕션 환경에서는 데이터 손실 방지와 서비스 연속성이 매우 중요합니다. 이 프로젝트는 Docker 환경에서 MySQL을 운영할 때 다음 두 가지 핵심 요소를 다룹니다:

1. **안정적인 백업 전략**: 데이터 손실을 방지하기 위한 자동화된 백업 솔루션
2. **고가용성 구성**: 서비스 중단을 최소화하기 위한 이중화 설정

## 시스템 요구사항

- Docker 및 Docker Compose가 설치된 Linux 서버
- 충분한 저장 공간 (데이터베이스 크기의 최소 3배)
- Ubuntu 24.02 (다른 Linux 배포판도 가능)

## 기본 MySQL 및 2개 이상의 app 컨테이너 설정

다음은 기본적인 MySQL 및 2개 이상의 app 컨테이너 설정입니다

```yaml
# docker-compose.yml
version: "3.8"

services:
  db:
    container_name: mysqldb
    image: mysql:8.0.41  # 명확한 버전 지정
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: fisa
      MYSQL_USER: user01
      MYSQL_PASSWORD: user01
    networks:
      - spring-mysql-net
    healthcheck:
      test: ['CMD-SHELL', 'mysqladmin ping -h 127.0.0.1 -u root --password=$${MYSQL_ROOT_PASSWORD} || exit 1']
      interval: 10s
      timeout: 2s
      retries: 100

  app1:
    container_name: springbootapp1
    build:
      context: .
      dockerfile: ./Dockerfile
    ports:
      - "8080:8080"
    environment:
      MYSQL_HOST: db
      MYSQL_PORT: 3306
      MYSQL_DATABASE: fisa
      MYSQL_USER: user01
      MYSQL_PASSWORD: user01
    depends_on:
      db:
        condition: service_healthy
    networks:
      - spring-mysql-net

  app2:
    container_name: springbootapp2
    build:
      context: .
      dockerfile: ./Dockerfile
    ports:
      - "8088:8080"
    environment:
      MYSQL_HOST: db
      MYSQL_PORT: 3306
      MYSQL_DATABASE: fisa
      MYSQL_USER: user01
      MYSQL_PASSWORD: user01
    depends_on:
      db:
        condition: service_healthy
    networks:
      - spring-mysql-net

networks:
  spring-mysql-net:
    driver: bridge

```

## Docker Compose 실행 스크립트

다음은 절대 경로를 사용한 Docker Compose 환경 시작 스크립트 설정입니다 
```bash
#!/bin/bash

# 애플리케이션 디렉토리 절대 경로 설정
APP_DIR="/home/ubuntu/08.dockerCompose"

# 디렉토리로 이동
cd "$APP_DIR"

echo "===== Docker Compose 환경 시작하기 ====="
echo "작업 디렉토리: $APP_DIR"
echo "파일 목록: $(ls -la)"
echo "========================================"

# Docker Compose 실행
echo "Docker Compose 환경을 시작합니다..."
docker-compose -f "$APP_DIR/docker-compose.yml" up -d

# 컨테이너 상태 확인
echo "컨테이너 상태 확인:"
docker-compose -f "$APP_DIR/docker-compose.yml" ps

echo "========================================"
echo "서비스가 백그라운드에서 실행 중입니다."
echo "로그를 확인하려면: docker-compose -f $APP_DIR/docker-compose.yml logs -f"
echo "환경을 중지하려면: docker-compose -f $APP_DIR/docker-compose.yml down"
echo "========================================"
```

실행 방법
```bash
$ chmod +x start-docker-env.sh

$ ./start-docker-env.sh
```

## 데이터베이스 백업 전략

### 자동화된 백업 스크립트

다음은 MySQL 백업을 자동화하는 쉘 스크립트입니다

```bash
#!/bin/bash
# mysql-backup.sh

# 백업 설정
LOG_DIR="/home/ubuntu/mysql-backups-logs"
BACKUP_DIR="/home/ubuntu/mysql-backups"
DATE=$(date +%Y%m%d_%H%M%S)
CONTAINER_NAME="mysqldb"
DATABASE_NAME="fisa"
MYSQL_USER="user01"
MYSQL_PASSWORD="user01"

# 백업 디렉토리 생성 (없는 경우)
mkdir -p $BACKUP_DIR

# 백업 로그 디렉토리 생성(없는 경우)
mkdir -p $LOG_DIR

# MySQL 백업 실행
echo "$(date): 백업 시작" >> $LOG_DIR/mysql-backup.log

docker exec $CONTAINER_NAME sh -c 'exec mysqldump -u'$MYSQL_USER' -p'$MYSQL_PASSWORD' '$DATABASE_NAME'' > $BACKUP_DIR/$DATABASE_NAME-$DATE.sql

echo "$(date): 백업 완료" >> $LOG_DIR/mysql-backup.log

# 30일 이상 된 백업 파일 삭제 (선택사항)
find $BACKUP_DIR -name "*.sql" -type f -mtime +30 -delete

```

스크립트 파일 권한 설정
```bash
chmod +x mysql-backup.sh
```

### 크론 작업 설정

30분마다 백업을 실행하도록 크론 작업을 설정합니다:

```bash
# 크론탭 편집
crontab -e

# 다음 라인 추가 (30분마다 실행)
*/10 * * * * /home/ubuntu/mysql-backup.sh >> /home/ubuntu/logs/cron-mysql-backup.log 2>&1
```



## 고가용성 구성

고가용성 구성에 대한 내용은 준비 중입니다. 곧 추가될 예정입니다.

마스터-슬레이브 복제, 자동 장애 복구, 로드 밸런싱 등의 고급 구성 방법이 포함될 예정입니다.

---
