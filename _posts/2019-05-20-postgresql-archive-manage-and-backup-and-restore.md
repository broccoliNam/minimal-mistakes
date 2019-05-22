---
title: "Postgresql WAL(Write-Ahead Logging) 아카이브 및 풀 백업과 복구"
categories:
    - Database
last_modified_at: 2019-05-22T15:47:00+09:00
excerpt: "Postgresql WAL(Write-Ahead Logging) 아카이브 및 풀 백업과 복구"
author_profile: true
tags:
    - Database
    - Postgresql
--- 
데이터베이스는 하나의 서비스에 꼭 필요하며, 무결성과 지속 가능성이 깨지지 않도록 하는 게 원칙입니다. 그에 따라 철저한 백업 관리 및 정책이 당연시 요구됩니다. 이번 포스팅에서는 Postgresql 환경에서 WAL 파일을 아카이빙하고 해당 파일을 이용해서 백업, 마지막으로 복구까지 해보도록 하겠습니다.

## WAL(Write-Ahead Logging)파일이란?
> Write-Ahead Logging (WAL) is a standard method for ensuring data integrity. Briefly, WAL's central concept is that changes to data files (where tables and indexes reside) must be written only after those changes have been logged, that is, after log records describing the changes have been flushed to permanent storage.

공식 문서의 내용을 확인해보면, WAL 파일은 데이터 무결성을 보장하는 표준 방법인데 어떤 데이터베이스에 쿼리를 날려 변경 이벤트를 실행할 때, 데이터를 변경하기 전에 해당 변경 내용을 로그에 미리 담아두고 이후에 변경한다는 개념이다. 이렇게 되면, 어떠한 문제(데이터 충돌, 파괴 등...)이 발생했을 때 WAL 파일에 로깅된 내용을 보고 복구가 가능할 수 있기 때문에 무결성을 보장해 줄 수 있다.

## 작업 환경
모든 작업 환경은 하나의 독립된 DB 서버(*저는 AWS EC2 사용)에서 진행됩니다.
- 운영체제 : CentOS 7
- Postgresql 버전 : 11.2

## 구조
![사진1](https://user-images.githubusercontent.com/35317926/58011653-aac08800-7b2d-11e9-941e-09faebb9f654.PNG)

구조는 간단합니다. 하나의 인스턴스 서버에 Postgresql가 띄워져 있고, 실제 모든 데이터는 Data 디스크에서 받아옵니다. 그리고 Data 디스크에선 Cluster 디스크로 데이터 파일을 백업하고 있습니다. 마지막으로 백업한 내용을 복원할 때는 Cluster 디스크에서 데이터 파일을 가져와 복원을 진행합니다.

## 실습
설정 파일 혹은 스크립트에 나와 있는 경로들은 직접 커스터마이징 하셔도 됩니다.
### WAL 파일 설정
***postgresql.conf***
```
wal_level = archive # 복구에 필요한 모든 WAL 파일 아카이빙에 필요한 정보를 로깅

# - Archiving -
archive_mode = on
archive_command = 'test ! -f {CLUSTER_DISK_PATH}/archive/%f && cp %p {CLUSTER_DISK_PATH}/archive/%f'
archive_timeout = 60
```
`archive_mode`에서 아카이브 모드를 실행시켜주고, `archive_command`에서 명령어를 통해 WAL 파일을 아카이브 하게 되는데 이미 있는 파일을 덮어쓰게 될 경우 복원 작업 시 의도치 않은 오류가 발생될 수 있기 때문에 `test` 명령어를 통해 중복된 파일이 있는지 없는지 검사한 뒤, WAL 파일을 아카이브 하게 됩니다. 그리고 `%p`는 WAL 로그파일 절대 경로, `%f`는 보관할 로그 파일 이름을 의미합니다. `archive_timeout`은 WAL 파일을 저장하는 주기를 지정하는 타임아웃 설정값입니다. 보통 60 ~ 120초가 적절한 값이 되겠습니다. 참고로 `test` 명령어는 헷갈리게도, 해당 검사가 맞는다면 0를, 틀리다면 1을 리턴하게 돼서 앞에 '!'을 통해 반대되는 결과 값을 리턴하게 해줍니다.

### 백업
***backup.sh***
```
#!/bin/bash
SRC=/var/lib/pgsql/11/data
DST={CLUSTER_DISK_PATH}/cluster
ARC={CLUSTER_DISK_PATH}/archive

TODAY=`date +%F`
ARCHIVE=data.tgz

function backup_pg() {
   ARCHIVE=$1
   echo Backing up database ...
   psql -U postgres -h localhost -c "SELECT pg_start_backup('$ARCHIVE');" postgres
   tar czf $ARCHIVE $SRC
   psql -U postgres -h localhost -c "SELECT pg_stop_backup();" postgres
   echo Completed!
}

mkdir -p $DST/$TODAY
cd $DST/$TODAY

backup_pg $ARCHIVE

cd $ARC
tar czpf archive.tgz ./
mv archive.tgz $DST/$TODAY
```
- $SRC : Postgresql Data 디렉터리 
- $DST : 백업된 파일이 들어갈 디렉터리
- $ARC : WAL 파일이 아카이브 됐던 디렉터리

를 각각 변수에 지정을 해주고, 날짜 별로 관리를 하기 위해 `$DST/$TODAY` 디렉터리를 생성합니다. 그리고 해당 경로로 이동해서 `backup_pg` 함수를 호출하게 됩니다. 함수 안에서는 `pg_start_backup('{LABEL}')` 명령어를 이용해서 백업을 진행합니다. 그리고 백업된 파일 `$SRC`를 `$ARCHIVE` 이름으로 압축하고 `pg_stop_backup()` 명령어를 이용해서 백업을 완료합니다. 완료하게 되면 실제로 Postgresql의 데이터 디렉터리가 압축된 파일(data.tgz)이 생성이 됩니다. 마지막으로 해당 백업 시점의 WAL 파일의 보관을 위해 `$ARC` 경로로 이동하여 WAL 파일을 압축하게 되고 백업 후 데이터 디렉터리가 압축되어 있던 경로인 `$DST/$TODAY`에 옮겨 담게 됩니다.

***스케줄링***

백업은 매번 수동으로 진행하게 되면 번거로워질 수 있습니다. 따라서 크론 작업에 다음과 같이 등록하도록 합니다. 물론 해당 내용도 원하는 시간에 맞춰서 변경해주시면 됩니다. 저는 매주 월요일 새벽 5시에 백업을 진행하는 크론 작업을 등록하도록 하겠습니다.
```
0 5 * * 1 su postgres -c '{BACKUP_SCRIPT_PATH}/backup.sh'
```
`su postgres -c 'COMMAND'`명령어는 postgres 사용자로 커맨드를 실행하겠다는 의미입니다. Postgresql의 백업 파일, 데이터 파일, 설정 파일 등은 일반적인 유저가 접근을 하면 안 되기 때문에 특정 유저를 이용해서 작업을 하도록 되어 있습니다. 실제로 Postgresql을 처음 설치했을 때 데이터 파일이나 설정 파일 같은 경우에는 권한이 `drwx------`로 되어 있고 소유자 또한 `postgres`로 되어 있습니다.

### 복구
복구는 Postgresql 서버를 중지시켜주고, 데이터 디렉터리를 삭제해주도록 합니다. 그리고 위에서 `$DST/%TODAY`에 백업해두었던 데이터 디렉터리가 압축되어 담긴 `data.tgz` 파일의 압축을 해제하고 기존의 데이터 디렉터리 위치에 옮겨줍니다.

***recovery.conf***
```
restore_command 'cp {CLUSTER_DISK_PATH}/archive/%f %p'
archive_cleanup_command '/usr/pgsql-11/bin/pg_archivecleanup {CLUSTER_DISK_PATH}/archive/ %r
```
그리고 데이터 디렉터리 안에 `recovery.conf` 파일을 위처럼 작업한 다음에 옮겨둡니다. `restore_command`에서는 명령어를 통해 백업되었던 WAL 파일을 WAL 로그 파일 절대 경로인 `%p`에 다시 복사하게 되어 복원해주고 `archive_cleanup_command`에서는 `pg_archivecleanup` 명령어를 통해 이제 복원에 사용된 WAL 파일을 다 지우는 작업을 하게 됩니다. 하지만, 우리는 이전에 만약의 상황을 대비하여 `$DST/$TODAY` 경로에 WAL 파일을 백업해두었습니다. 마지막으로, 다시 postgresql 서버를 시작시켜줍니다.

### 결과
WAL 파일을 저장해두었던 클러스터 디스크에는 WAL 파일이 삭제되고, 데이터 디렉터리를 확인해보면 `recovery.conf`가 `recovery.done`으로 변경되게 됩니다. 그리고 `backup_label.old`라는 파일이 새로 생성되는데 해당 파일은 복원에 대한 로그입니다.
***backup_label.old***
```
START WAL LOCATION: 0/44000028 (file 000000060000000000000044)
CHECKPOINT LOCATION: 0/44000060
BACKUP METHOD: pg_start_backup
BACKUP FROM: master
START TIME: 2019-05-20 07:15:43 UTC
LABEL: data.tgz
START TIMELINE: 6
```
내용을 확인해보면 어느 시점의 WAL 파일부터 복원을 진행했고 어디에서 끝났는지 등등 복원에 대한 정보가 나와있습니다. 그리고 실제로 데이터베이스에 접속하여 데이터를 확인해봤을 때 기존에 백업해두었던 내용과 동일하게 적용돼 보입니다.

### 이슈
***1. `pg_archivecleanup` 명령어를 찾지 못했을 때***

설치 시에, `pg_archivecleanup`가 없는 상태로 Postgresql만 설치되는 경우가 많습니다. 다음과 같은 경우에는 다음 명령어를 통해 라이브러리를 설치해줍니다.
```
yum install postgresql11-contrib
```

### 참고
- https://www.postgresql.org/docs/11/index.html
- https://stackabuse.com/backing-up-and-restoring-postgresql-databases/