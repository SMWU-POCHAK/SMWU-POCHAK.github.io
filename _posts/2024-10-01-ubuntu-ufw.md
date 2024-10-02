---
title: Ubuntu UFW 방화벽 확인하기
date: 2024-10-01 01:30:00 +0900
categories: [Infra, Firewall]
tags: [backend, infra, firewall, error]     # TAG names should always be lowercase
author: lucy-oh
---

# 오류 발생 상황

## 수상한 점

포착은 배포를 위해 docker hub를 사용하고 있습니다! private registry에 빌드 후 푸시한 이미지를 각각 개발 서버와 운영 서버에서 pull을 받아와서 사용하고 있는데요!

앱을 이전하고 애플로그인 설정 변경을 위해 서버에 새로운 키 값을 넣어주고, 이를 운영 서버에 반영하고자 하였습니다.

하지만 이상하게도 분명 개발 서버와 운영 서버는 설정 값의 차이일 뿐 거의 동일한 버전의 이미지를 pull 받아서 사용하게 했는데.. 🤷‍♀️ 운영 서버에서만 jdbc connection error가 발생하였습니다.

> 심지어 docker hub에 올린 이미지를 로컬에서 실행시켰을 때도 에러가 발생하지 않았어요....;;😂
{: .prompt-danger }

![img](/assets/img/2024-10-01-ubuntu-ufw/img0.png)
_킹받게 잘돌아가는 개발 서버_

### 오류 메세지

제가 확인했던 오류 메세지입니다.

```
com.mysql.cj.jdbc.exceptions.CommunicationsException: Communications link failure

The last packet sent successfully to the server was 0 milliseconds ago. The driver has not received any packets from the server.
	at com.mysql.cj.jdbc.exceptions.SQLError.createCommunicationsException(SQLError.java:175) ~[mysql-connector-j-8.1.0.jar!/:8.1.0]
	at com.mysql.cj.jdbc.exceptions.SQLExceptionsMapping.translateException(SQLExceptionsMapping.java:64) ~[mysql-connector-j-8.1.0.jar!/:8.1.0]
	at com.mysql.cj.jdbc.ConnectionImpl.createNewIO(ConnectionImpl.java:819) ~[mysql-connector-j-8.1.0.jar!/:8.1.0]
	at com.mysql.cj.jdbc.ConnectionImpl.<init>(ConnectionImpl.java:440) ~[mysql-connector-j-8.1.0.jar!/:8.1.0]
	at com.mysql.cj.jdbc.ConnectionImpl.getInstance(ConnectionImpl.java:239) ~[mysql-connector-j-8.1.0.jar!/:8.1.0]
	at com.mysql.cj.jdbc.NonRegisteringDriver.connect(NonRegisteringDriver.java:188) ~[mysql-connector-j-8.1.0.jar!/:8.1.0]
    ...
	at com.zaxxer.hikari.HikariDataSource.getConnection(HikariDataSource.java:112) ~[HikariCP-5.0.1.jar!/:na]
	at org.springframework.jdbc.datasource.DataSourceUtils.fetchConnection(DataSourceUtils.java:160) ~[spring-jdbc-6.1.2.jar!/:6.1.2]
	at org.springframework.jdbc.datasource.DataSourceUtils.doGetConnection(DataSourceUtils.java:118) ~[spring-jdbc-6.1.2.jar!/:6.1.2]
    ...
Caused by: com.mysql.cj.exceptions.CJCommunicationsException: Communications link failure

```

> `com.mysql.cj.jdbc.exceptions.CommunicationsException: Communications link failure` <br>
> `Caused by: com.mysql.cj.exceptions.CJCommunicationsException: Communications link failure` 
{: .prompt-info }

가장 첫 줄과 마지막 줄을 확인해보면 알겠지만 JDBC가 데이터베이스와 연결하지 못하는 이슈가 발생하고 있었습니다.

## 해결의 실마리

로컬, 개발 서버에서는 모두 정상적으로 작동되고 있었기에 불현듯이.. 스쳐지나가는 단어가 있었습니다ㅎㅎ

바로 `방화벽`.. 보통 클라우드 환경이 로컬 환경과 차이가 난다면 92.45%로 방화벽 문제일 확률이 매우 크다고 생각했기에! GCP Cloud SQL의 인바운드 설정에서 문제점을 먼저 찾고자 하였습니다.

# 환경

## 호스트 환경
- GCP Compute Engine (Ubuntu 20.04.6 LTS)
- GCP Cloud SQL (MySQL 8.0.31)
- Docker version 27.1.2

# GCP 방화벽 확인하기

## Cloud SQL 방화벽

먼저 데이터베이스 서버를 설정해둔 Cloud SQL의 방화벽 규칙을 먼저 확인하였습니다.

![img](/assets/img/2024-10-01-ubuntu-ufw/img1.png)
_왼쪽 "연결" 클릭_

![img](/assets/img/2024-10-01-ubuntu-ufw/img2.png)
_"네트워킹" 탭 확인_

위 사진처럼 `연결` > `네트워킹` 으로 접속하면 Cloud SQL이 허용하고 있는 IP주소 목록을 확인할 수 있습니다.

하지만 절망적이게도 철저한 제가 Cloud SQL 방화벽 설정은 철저히 해둔 상태였기에.. 다른 방화벽 혹은 compute engine 내부에서 설정한 방화벽을 확인하게 됩니다.

# Ubuntu 방화벽 확인하기

## 개발 서버와 운영 서버의 방화벽 세팅 비교
UFW는 데비안 계열 및 다양한 리눅스 환경에서 작동되는 사용하기 쉬운 방화벽 관리 프로그램입니다!

운영 서버에서는 포트 관리를 위해 http(80), https(443), ssh(22)번을 열어주었는데, 여기서 제 착각이 있었습니다.

DNS 설정 당시, [이렇게](https://smwu-pochak.github.io/posts/dns-setting/#ubuntu-%EB%B0%A9%ED%99%94%EB%B2%BD-%ED%99%95%EC%9D%B8) 방화벽 설정을 해주었는데, 생각해보니 MySQL 서버 연결을 위해 3306 포트가 필요하다는 생각을 못했던 것입니다.. 🤦‍♀️🤦‍♀️

<br>

개발 서버는 따로 방화벽 설정을 해주지 않아 다음과 같이 ufw status를 확인했을 때 inactive로 설정이 되어있었습니다. <br> 
(애초에 방화벽이 없었으니 데이터베이스 서버 접근도 정상적으로 이루어졌던겁니다 .. 이런~)

![img](/assets/img/2024-10-01-ubuntu-ufw/img3.png)

## 방화벽이 문제의 원인!

먼저 방화벽을 inactive하게 설정해주고, 컨테이너가 정상적으로 실행되는지부터 확인해보겠습니다.

```sh
# 방화벽 비활성화
sudo ufw disable

# 방화벽 상태 확인
sudo ufw status
```

위와 같이 방화벽 비활성화를 실행시키면 개발 서버 처럼 `status: inactive`를 확인할 수 있습니다.
그리고 다시 도커 컨테이너를 실행시켜주면, 정상적으로 작동되는 모습을 확인할 수 있습니다.

![img](/assets/img/2024-10-01-ubuntu-ufw/img4.png)

## 방화벽 설정하기

일단 이대로 port를 전부 열어두는 것은 분명 편하지만 보안이 굉장히 위험해지기 때문에, 다시 폐쇄적인 정책으로 되돌려 주겠습니다.

원래대로 방화벽을 다시 활성화 해주고, 80, 443, 22, 3306 포트를 열어준 뒤 다시 도커 컨테이너를 실행시켜 주겠습니다.

```sh
# 방화벽 활성화
sudo ufw enable

# 방화벽 3306 규칙 추가
sudo ufw allow 3306

# 방화벽 상태 확인
sudo ufw status
```

그 뒤 다시 도커 컨테이너를 실행시켜주면, 이번에는 정상적으로 JDBC 커넥션 연결이 완료되는 모습을 확인할 수 있습니다!