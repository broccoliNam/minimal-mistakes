---
title: "패키지 관리자를 통해 NGINX 최신 버전(Mainline/Stable) 설치하기"
categories:
    - Server
last_modified_at: 2019-02-07T11:00:00+09:00
excerpt: "NGINX 최신 버전 설치하기"
toc: true
author_profile: true
read_time: true
tags:
    - Web server
    - nginx
--- 


## NGINX의 버전 관리
![사진1](https://www.nginx.com/wp-content/uploads/2014/04/branch-1024x395.png)
일반적으로, 많이 알려진 버전은 단 두 가지인데요. Mainline 버전과 Stable 버전이 있습니다. 새로운 특징, 기능, 버그 패치 등은 Mainline 버전에서 작업하고 그 이후에, 새로운 기능이 추가되지 않고 버그 패치만 하는 게 Stable 버전입니다. 버전 선택에 관련해서 NGINX의 공식 입장은 다음과 같습니다.

> We recommend that in general you deploy the NGINX mainline branch at all times. The main reason to use the stable branch is that you are concerned about possible impacts of new features, such as incompatibility with third-party modules or the inadvertent introduction of bugs in new features.

기본적으로 Mainline 버전을 택하고 지속적으로 업데이트 하기를 권장하지만 Stable 버전은 서드파티 모듈과의 호환성 문제 또는 새로운 기능이 도입됨에 따라, 생길 수 있는 문제 때문에 지속적으로 업데이트하지 못할 때 사용된다고 나와 있습니다.

따라서, 버전을 선택할 때 환경을 고려하여 신중하게 선택하는 것이 중요할 것 같습니다.

## 설치
### RHEL/CentOS 계열
의존성 설치: 
```
sudo yum install yum-utils
```

yum 저장소에 nginx 저장소를 추가:
```
sudo vi /etc/yum.repos.d/nginx.repo
```
```
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
```

(Optinal)mainline 버전을 설치하려면 다음 명령어를 수행:
```
sudo yum-config-manager --enable nginx-mainline
```

yum 저장소 갱신:
```
sudo yum update
```

NGINX 설치:
```
sudo yum install -y nginx
```

NGINX 버전 확인:
```
nginx -v
```
**Output**
```
nginx version: nginx/1.14.2
```

### Debian/Ubuntu 계열
의존성 설치: 
```
sudo apt install curl gnupg2 ca-certificates lsb-release
```

apt 저장소에 원하는 버전의 nginx 저장소 추가:
```
# Debian 계열
echo "deb http://nginx.org/packages/debian `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list

echo "deb http://nginx.org/packages/mainline/debian `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list

# Ubuntu 계열
echo "deb http://nginx.org/packages/ubuntu `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list

echo "deb http://nginx.org/packages/mainline/ubuntu `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list
```

패키지 신뢰성 확인:
```
curl -fsSL https://nginx.org/keys/nginx_signing.key | sudo apt-key add -
```

키 신뢰성 확인:
```
sudo apt-key fingerprint ABF5BD827BD9BF62
```

apt 저장소 갱신:
```
sudo apt update
```

NGINX 설치:
```
sudo apt-get install nginx
```

NGINX 버전 확인:
```
nginx -v
```
**Output**
```
nginx version: nginx/1.14.2
```

## 에러
### RHEL/CentOS 계열
1. yum 저장소에 NGINX 저장소를 등록하고, yum 관련 명령어 실행 시에 다음과 같은 에러 발생
```
failure: repodata/repomd.xml from nginx-stable: [Errno 256] No more mirrors to try.
http://nginx.org/packages/centos/7Server/x86_64/repodata/repomd.xml: [Errno 14] HTTP Error 404 - Not Found
```
저장소에 등록 내용 중 baseurl에서 $releasever 값이 잘못된 경우입니다. 이 경우에 자신의 운영체제 버전을 확인해서 넣어줍시다. <br/><br/>
**Example**
```
http://nginx.org/packages/centos/7/$basearch/
```