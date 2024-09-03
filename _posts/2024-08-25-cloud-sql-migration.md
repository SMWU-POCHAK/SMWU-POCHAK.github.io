---
title: AWS RDS에서 Cloud SQL로 마이그레이션 작업
date: 2024-08-25 18:30:00 +0900
categories: [Infra, Database]
tags: [backend]     # TAG names should always be lowercase
author: lucy-oh
---

# Cloud SQL로 데이터베이스를 변경한 이유

창업지원단의 지원금을 받고 있기에 대부분의 서비스를 AWS에서 GCP로 옮기고 있는 상황입니다.

따라서 기존에 사용하던 AWS의 RDS에서 GCP의 Cloud SQL로 마이그레이션을 하기로 결정하였습니다.

# 작업 과정
## GCP Database Migration API 활성화하기

GCP에서 제공해주는 Database Migration API를 사용하여 작업을 시작해보겠습니다. <br>
먼저 Database Migration API를 검색하여 활성화를 시켜주어야 합니다.

아래 이미지는 이미 활성화된 모습입니다.

![img](/assets/img/2024-08-25-cloud-sql-migration/1.png)

## 연결 프로필 생성

그 뒤, 연결 프로필 `Connection profiles` 을 생성해주어야 합니다.

> 연결 프로필은 소스 데이터베이스 인스턴스(예: MySQL용 Amazon RDS)에 대한 정보를 저장하고, <br> 
> Database Migration Service에서 대상 Cloud SQL 데이터베이스 인스턴스로 데이터를 마이그레이션하는 데 사용됩니다.
{: .prompt-tip }

<br>

`데이터베이스 마이그레이션 > 연결 프로필`에서 <br>
상단의 `+ 프로필 만들기` 버튼을 눌러 새로운 프로필을 만들어줍시다.

![img](/assets/img/2024-08-25-cloud-sql-migration/2.png)


다음과 같이 세팅해주었습니다.

1. 데이터베이스 엔진 : `MySQL`
2. 연결 프로필 이름 : `mysql-rds` (편한 이름으로 변경 가능)
3. 연결 프로필 ID : `mysql-rds` (편한 아이디로 변경 가능)
4. 호스트 이름 또는 IP 주소 : {RDS IP 주소}
5. 포트 : `3306` (MySQL 기본 포트 - 설정에 따라 변경될 수 있습니다.)
6. 사용자 이름 : {MySQL 사용자 이름}
7. 비밀번호 : {MySQL 비밀번호}
8. 리전 : `asia-northeast3 (서울)` (저는 서울로 설정하였습니다.)

## 마이그레이션 작업 생성

`데이터베이스 마이그레이션 > 마이그레이션 작업`에서<br>
상단의 `+ 마이그레이션 작업 만들기` 버튼을 눌러 새로운 마이그레이션 작업을 만들어줍시다.

![img](/assets/img/2024-08-25-cloud-sql-migration/3.png)

1. 소스 데이터베이스 엔진 : `MySQL`
2. 대상 리전: `asia-northeast3 (서울)`
3. 마이그레이션 작업 유형: 일회성

으로 설정하고 `저장하고 계속하기` 버튼을 클릭해줍니다.

<br>

다음으로 소스 연결 프로필로 아까 생성한 `mysql-rds`를 선택해줍니다. <br>
다른건 기본 세팅으로 두고 진행하였습니다.

<br>

마이그레이션 대상 데이터베이스를 생성합니다. <br>
비용을 위해 최소 사양으로 설정하였습니다!
- `Enterprise`
- 공유 코어 vCPU 1개, `0.614GB`
- `HDD`, `10GB`
- `MySQL 8.0.31`

<br>

대상 데이터베이스 생성이 완료되면, 다음으로 넘어가 연결 방법을 선택합니다.

연결 방법은 `IP 허용 목록` 으로 설정하고, `Destination IP Address`를 복사합니다.<br> 
복사한 AWS RDS 인스턴스의 보안그룹에 추가하여 접근이 가능하도록 설정해주어야 합니다.

<br> 

모두 설정해준 뒤, 마이그레이션 작업을 시작해줍니다. <br> 
- 설정에 이상이 없다면 마이그레이션 작업이 시작됩니다.

![img](/assets/img/2024-08-25-cloud-sql-migration/4.png)
_설정은 이 사진을 참고해주세요!_

약 13분 정도 작업이 진행되었고, <br> 
이후에 `Cloud SQL > (대상 인스턴스) > 작업`에 들어가서 <br> 
마이그레이션 작업 과정은 다음과 같이 확인할 수 있었습니다.

![img](/assets/img/2024-08-25-cloud-sql-migration/5.png)



# 참고자료
- [Migrating to Cloud SQL from Amazon RDS for MySQL Using Database Migration Service](https://www.cloudskillsboost.google/focuses/17696?catalog_rank=%7B%22rank%22:1,%22num_filters%22:0,%22has_search%22:true%7D&parent=catalog&search_id=22626074)