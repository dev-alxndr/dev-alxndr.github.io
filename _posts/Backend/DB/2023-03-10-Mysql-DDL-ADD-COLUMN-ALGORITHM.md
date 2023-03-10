---
title: DDL ADD COLUMN ALGORITHM
tags: [Mysql]
categories: [Backend, DB]
date: 2023-03-10 20:07:41
---

운영환경 테이블에 새로운 컬럼을 추가 해야하는 경우가 생겼다.
운영환경에 DDL을 실행하게 되면 운영 중 환경에 문제가 생길 수 있기 때문에 조사를 했다

## Online DDL Algorithm
서비스 중지 없이 DDL이 가능한 Online DDL Algorithm 종류   
수행되는 방식에 따라서 주로 성능적인 차이를 보인다.

#### 1. Instant
Mysql 8.0.12 이상 버전부터 Default로 설정되어 있는 방법    
테이블 락이 전형 발생하지 않아 현재 최상의 알고리즘으로 권장   
메타데이터 정보만 바로 변경하며 8.0.29이전 버전에서는 마지막 컬럼에만 추가가 가능했지만   
8.0.29버전부터는 컬럼의 위치를 지정할 수 있다.

#### 2. INPLACE
Mysql 8.0.12 버전 이하 Default 설정    
원본 테이블에 직접 변경 작업이 발생   
작업의 준비 및 실행 단계에서 테이블에 배타적 메타 데이터 잠금이 잠깐 수행될 수 있습니다.
```
메타 데이터 잠금
메타 데이터의 잠금을 사용하여 객체(테이블, 트리거 등)에 대한 엑세스를 관리
``` 

#### 3. Copy
DDL 수행 시 원본 테이블의 복사본 테이블을 생성하여 row단위로 새로운 테이블에 복사 처리 방식   
복사가 완료 된 후 복사본 테이블이 원본 테이블명으로 변경되는 처리 방식

## Lock Clause
Online DDL 수행 중 테이블에 대한 동시 엑세스 수준을 조정   
동시 Access에 대한 차이를 보인다.

#### LOCK=NONE
Concurrent Query와 Concurrent DML을 허용

#### LOCK=SHARED
Concurrent Query 허용 Concurrent DML 불가

#### LOCK=DEFAULT
수행 가능한 동시성 작업에 대해 허용   
Lock 구문을 생략하면 Defualt로 사용

#### LOCK=EXCLUSIVE
Concurrent Query & DML 불가

---
### 환경
#### 운영환경
- AWS Aurora 8.0.mysql_aurora.3.02.2
	- [Mysql 8.0.23 호환버전](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraMySQLReleaseNotes/AuroraMySQL.Updates.3022.html)
#### 테스트 환경
- Docker Mysql 8.0.31

##### 테스트 환경 2백만건 기준
Algorithm=INSTANT : 즉시 컬럼 생성
Algorithm=INPLACE : 53sec
Algorithm=COPY : 1m 14sec

운영 환경에서는 Mysql 8.0.23과 호환되는 Aurora를 사용하고 있어서 컬럼 위치 지정 불가

[MySQL :: MySQL 8.0 Reference Manual :: 13.1.9 ALTER TABLE Statement-Performance and Space Requirements](https://dev.mysql.com/doc/refman/8.0/en/alter-table.html)

[MySQL :: MySQL 8.0 Reference Manual :: 15.12.1 Online DDL Operations](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html)

[MySQL :: MySQL 8.0: InnoDB now supports Instant ADD COLUMN](https://dev.mysql.com/blog-archive/mysql-8-0-innodb-now-supports-instant-add-column/)

[MySQL Online DDL - InnoDB Online DDL - Alter Online](https://hoing.io/archives/6693)