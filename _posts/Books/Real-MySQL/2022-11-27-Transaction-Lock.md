---
title: Transaction & Lock
tags: [Real-MySQL, Transaction, Lock, Mysql]
categories: [Books, Real-MySQL]
date: 2022-11-27 22:11:01
---
[Real MySQL 8.0 (1권) - YES24](http://www.yes24.com/Product/Goods/103415627) 을 읽고 정리했습니다.

# 5. Transaction & Lock
## Transaction
트랜잭션은 작업의 완정성을 보장해주는 것이다.

논리적인 작업 셋 자체가 100% 적용되거나 (COMMIT을 실행했을 때) 아무것도 적용되지 않아야 (ROLLBACK 또는 트랜잭션을 ROLLBACK 시키는 오류가 발생했을 때) 함을 보장해주는 것

### 잠금과 트랜잭션의 기능적 차이
- 잠금 : 동시성 제어
- 트랜잭션 : 데이터의 정합성
### 격리 수준
하나의 트랜잭션 내에서 또는 여러 트랜잭션 간의 작업 내용을 어떻게 공유하고 차단할 것인지를 결정하는 레벨
### 트랜잭션 처리에 좋지않은 영향을 미치는 부분
- 일반적으로 데이터베이스 커넥션은 캐수가 제한적이어서 각 단위 프로그램이 커넥션을 소유하는 시간이 길어질수록 사용 가능한 여유 커넥션의 개수는 줄어든다.

- 메일전송이나 FTP 파일 전송 작업 또는 **네트워크를 통해 원격서버와 통신하는 등과 같은 작업은 어떻게 해서든 DBMS의 트랜잭션 내에서 제거하는 것이 좋다.**

## MySQL 엔진의 잠금
스토리지 엔진레벨과 MySQL 엔진 레벨로 나눌 수 있다.

### 글로벌 락
`FLUSH TABLES WITH READ LOCK` 명령으로 획득 할 수 있다.
MySQL에서 제공하는 잠금 가운데 가장 범위가 크다. (MySQL 서버 전체)   
한 세션에서 글로벌 락을 획득하면 다른 세션에서 SELECT를 제외한 대부분의 DDL 문장이나 DML 문장을 실행하는 경우 글로벌 락이 해제 될 때까지 대기 상태로 남는다.    
DB 스토리지 엔진은 트랜잭션을 지원하기 때문에 모든 데이터 변경 작업을 멈출 필요는 없다.

Mysql 8.0부터 `Xtrabackup` , `Enterprise Backup`과 같은 백업 툴들의 안정적인 실행을 위해 백업 락이 도입됐다.
```sql
>> LOCK INSTANCE FOR BACKUP;
-- // Start Backup
>> UNLOCK INSTANCE;
```
특정세션에서 백업 락을 획득하면 모든 세션에서 변경 할 수 없다.
- 데이터베이스 및 테이블 등 모든 객체 생성 및 변경, 삭제
- REPAIR TABLE 과 OPTIMIZE TABLE 명령
- 사용자 관리 및 비밀번호 변경
### 테이블 락
- MyISAM / MEMORY

데이터를 변경하는 쿼리를 실행하면 발생. 데이터가 변경되는 테이블에 잠금을 설정하고 데이터를 변경한 후 즉시 잠금을 해제하는 형태로 사용한다.
- InnoDB

스토리지 엔진 차원에서 레코드 기반의 잠금을 제공하기 때문에 단순 데이터 변경 쿼리로 묵시적인 테이블 락이 설정 되지 않는다.   
스키마를 변경하는 쿼리 DDL의 경우에만 영향을 미친다.

  

### 네임드 락
`GET_LOCK()` 함수를 이용해서 임의의 문자열에 대해 잠금을 설정한다.   
네임드 락은 단순히 사용자가 지정한 문자열(String)에 대해 획득하고 반납(해제)하는 잠금이다.   
상호 동기화를 처리해야 할 때 네임드 락을 이용하면 쉽게 해결할 수 있다.
```sql
-- "mylock"이라는 문자열에 대해 잠금을 획득한다.
>> SELECT GET_LOCK('mylock', 2);
-- "mylock"이라는 문자열에 대해 잠금이 설정돼 있는지 확인한다.
>> SELECT IS_FREE_LOCK('mylock');
-- "mylock"이라는 문자열에 대해 획득했던 잠금을 반납(해제)한다
>> SELECT RELEASE_LOCK('mylock');
```
> 배치 프로그램처럼 한꺼번에 많은 레코드를 변경하는 쿼리는 자주 데드락의 원인이 되곤 한다.

MySQL 8.0 버전부터 네임드 락을 중첩해서 사용할 수 있으며, 현재 세션에서 획득한 네임드 락을 한번에 모두 해제하는 기능이 추가되었다.
```sql
>> SELECT GET_LOCK('mylock_1', 10);
>> SELECT GET_LOCK('mylock_2', 10);
-- 현재 세션에서 잡은 락을 모두 해제한다.
>> SELECT RELEASE_ALL_LOCKS();
```

### 메타데이터 락
데이터베이스 객체의 이름이나 구조를 변경하는 경우에 획득하는 잠금이다.
명시적으로 획득하거나 해제할 수 없다.

## InnoDB 스토리지 엔진 잠금
[MySQL :: MySQL 8.0 Reference Manual :: 15.7.1 InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)   
InnoDB 스토리지 엔진은 스토리지 엔진 내부에서 레코드 기반의 잠금 방식을 탑재하고 있다.
MyISAM 스토리지 엔진에 비해 훨씬 뛰어난 동시성 처리를 제공할 수 있다.
- MySQL 5.5이상 InnoDB 잠금정보를 진단할 수 있는 방법   
`information_schema` > `INNODB_TRX`, `INNODB_LOCKS`, `INNODB_LOCK_WAITS`

조인해서 조회하면 현재 어떤 트랜잭션이 어떤 잠금을 대기하고 있고, 해당 잠금을 어느 트랜잭션이 가지고 있는지 확인 할 수 있으며, 잠금을 가진 클라이언트를 찾아서 종료시킬 수 있다.  

### InnoDB 스토리지 엔진의 잠금

InnoDB 스토리지 엔진에서는 레코드 락뿐 아니라 레코드와 레코드 사이의 간격을 잠그는 갭(GAP)락이라는 것이 존재한다.
![Untitled](/assets/img/untitled.png)

- 레코드 락
레코드 자체만을 잠그는 것을 레코드 락이라 한다.
한 가지 중요한 차이는 InnoDB 스토리지 엔진은 레코드 자체가 아니라 인덱스의 레코드를 잠근다는 점이다.
인덱스가 하나도 없는 테이블이더라도 내부적으로 자동 생성된 클러스터 인덱스를 이용해 잠금을 설정한다.

- 갭(GAP) 락
갭 락은 레코드 자체가 아니라 레코드와 바로 인접한 레코드 사이의 간격만을 잠그는 것을 의미한다.
갭 락의 역할은 레코드와 레코드 사이의 간격에 새로운 레코드가 생성(INSERT) 되는 것을 제어하는 것이다.

- 넥스트 키 락
레코드 락과 갭 락을 합쳐 놓은 형태의 잠금을 넥스트 키 락(Next Key Lock)이라고 한다.
넥스트 키 락과 갭 락으로 인해 데드락이 발생하거나 다른 트랜잭션을 기다리게 만드는 일이 자주 발생한다.

- [자동 증가 락](https://dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html)

내부적으로 AUTO_INCREMENT LOCK 이라고 하는 테이블 수준의 잠금을 사용한다.
새로운 레크도를 저장하는 쿼리에서만 필요하며, UPDATE, DELETE 등의 쿼리에서는 걸리지 않는다.
- `innodb_autoinc_lock_mode = 0`   
MySQL 5.0과 동일한 잠금 방식으로 모든 INSERT 문장은 자동 증가 락을 사용한다.
- `innodb_autoinc_lock_mode = 1`   
단순히 한 건 또는 여러 건의 레코드를 INSERT하는 SQL 중에서 INSERT되는 레코드 건 수를 정확히 예측할 수 있을 때는 자동 증가 락을 사용하지 않고, 훨씬 빠른 래치(뮤텍스)를 이용해 처리한다.
`INSERT…SELECT` 와 같이 MySQL 서버가 건수를 예측할 수 없을 때는 자동 증가 락을 사용한다.
INSERT 문장이 완료되기 전까지는 자동 증가 락은 해제되지 않기 때문에 다른 커넥션에서는 INSERT를 실행하지 못하고 대기하게 된다.

- `innodb_autoinc_lock_mode = 2`    
InnoDB 스토리지 엔진은 절대 자동 증가 락을 걸지 않고 경량화된 래치(뮤텍스)를 사용한다.   
연속된 증가 값을 보장하지 않는다. 동시 처리 성능이 높아지지만 자동 증가 기능은 유니크한 값이 생성된다는 것만 보장한다. STATEMENT 포멧의 바이너리 로그를 사용하는 복제에서는 소스 서버와 레플리카 서버의 자동 증가 값이 달라질 수도 있기 떄문에 주의해야 한다.

> MySQL 8.0 부터는 innodb_autoinc_lock_mode 기본 값이 2이다. 8.0부터 바이너리 로그 포멧이 STATEMENT 가 아니라 ROW 포멧이 기본 값이 됐기 때문이다. STATEMENT가 아니기 때문에 레플리카 서버와 자동 증가 값이 달라지지 않는다.

  

## 인덱스와 잠금
InnoDB의 잠금은 레코드를 잠그는 것이 아니라 인덱스를 잠그는 방식으로 처리되며, 변경해야 할 레코드를 찾기 위해 검색한 인덱스의 레코드를 모두 락을 걸어야 한다.

만약 UPDATE 문장을 위해 적절한 인덱스가 준비되어 있지 않다면 모든 레코드가 잠기게 되며, 각 클라이언트 간의 동시성이 상당이 떨어지게 된다.

InnoDB에서 인덱스 설계가 중요한 이유이다.  

## 레코드 수준의 잠금 확인 및 해제
InnoDB 스토리지 엔진을 사용하는 테이블의 레코드 수준 잠금은 테이블 수준의 잠금보다는 조금 더 복잡하다.
각 버전 별 레코드 잠금과 잠금을 대기하는 클라이언트의 정보를 확인하는 방법을 알아본다.

- MySQL 5.1   
`information_schema` DB에 `INNODB_TRX`, `INNODB_LOCKS`, `INNODB_LOCK_WAITS`를 통해 확인 가능했다.

- MySQL 8.0   
`performance_schema` DB에 `data_locks`, `data_lock_waits` 테이블로 대체되고 있다.

![Untitled](/assets/img/untitled1.png)

## MySQL 격리 수준
[MySQL :: MySQL 8.0 Reference Manual :: 15.7.2.1 Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)

트랜잭션의 격리 수준(isolation level)이란 여러 트랜잭션이 동시에 처리될 때 특정 트랜잭션이 다른 트랜잭션에서 변경하거나 조회하는 데이터를 볼 수 있게 허용할지 말지를 결정하는 것이다.   
4개의 격리 수준에서 순서대로 뒤로 갈수록 각 트랜잭션 간의 데이터 격리(고립) 정도가 높아지며, 동시 처리 성능도 떨어지는 것이 일반적이다.
서버의 처리 성능이 많이 떨어질 것으로 생각하는 사용자가 많지만, `SERIALIZABLE` 격리 수준이 아니라면 크게 성능의 개선이나 저하는 없다.

| | DIRTY READ | NON_REPEATABLE_READ | PHANTOM_READ |
| --- | --- | --- | --- |
| READ UNCOMMITED | 발생 | 발생 | 발생 |
| READ COMMITED | 없음 | 발생 | 발생 |=
| REPEATABLE READ | 없음 | 없음 | 발생(InnoDB는 없음) |
| SERIALIZABLE | 없음 | 없음 | 없음 |

### READ UNCOMMITED
READ UNCOMMITED 각 트랜잭션에서의 변경 내용이 COMMIT이나 ROLLBACK 여부에 상관없이 다른 트랜잭션에 보인다.
어떤 트랜잭션에서 처리한 작업이 완료되지 않았는데도 다른 트랜잭션에서 볼 수 있는 현상을 더티 리드(Dirty Read)라고 한다. MySQL을 사용한다면 최소한 READ COMMITED 이상의 격리 수준 사용을 권장한다.

### READ COMMITED
COMMIT이 완료된 데이터만 다른 트랜잭션에서 조회할 수 있기 때문에 더티 리드(Dirty Read) 현상은 발생하지 않는다. 하지만 `NON-REPEATABLE READ` (REPEATABLE READ가 불가능하다)라는 부정합의 문제가 있다.
COMMIT 된 데이터를 읽기 때문에 A 트랜잭션에서 한번 읽고 B 트랜잭션에 해당 데이터를 수정하고 A 트랜잭션에서 다시 읽게 된다면 COMMIT된 데이터를 읽기 때문에 하나의 트랜잭션 내에서 똑같은 SELECT 쿼리를 실행했을 때는 항상 같은 결과를 가져와야 한다는 `REPEATABLE READ` 정합성에 어긋나게 된다.
> READ COMMITED 격리 수준에서는 트랜잭션 내에서 실행되는 SELECT 문장과 트랜잭션 외부에서 실행되는 SELECT 문장의 차이가 별로 없다. (REPEATABLE READ와 차이점은 밑에서..)

### REPEATBLE READ
MySQL InnoDB 스토리지 엔진에서 기본으로 사용되는 격리 수준이다.
`NON-REPEATABLE READ` 부정합이 발생하지 않는다. InnoDB 스토리지 엔진은 트랜잭션이 ROLLBACK될 가능성에 대비해 변경되기 전 레코드를 언두(Undo) 공간에 백업해두고 실제 레코드 값을 변경한다. 이러한 방식을 [MVCC](https://youtu.be/wiVvVanI3p4)라고 한다.
장시간 트랜잭션을 종료하지 않으면 언두 영역이 백업된 데이터로 무한정 커질 수 있다. 언두에 백업된 레코드가 많아지면 MySQL 서버의 처리 성능이 떨어질 수 있다.
![PHANTOM READ](/assets/img/untitled2.png)

- PHANTOM READ   
사용자 B는 `BEGIN` 명령으로 트랜잭션을 시작한 후 SELECT를 수행한다. 두 번의 SELECT 쿼리 결과는 같아야하지만 사용자 B가 실행하는 두 번의 `SELECT…FOR UPDATE` 쿼리 결과는 서로 다르다.
다른 트랜잭션에서 수행한 변경 작업에 의해 레코드가 보였다 안보였다 하는 현상을 **PHANTOM READ**(또는 PHANTOM ROW)라고 한다.

`SELECT…FOR UPDATE` 쿼리는 SELECT 하는 레코드에 쓰기 잠금을 걸어야 하는데, 언두 레코드에는 잠금을 걸 수 없다. (여기서 말하는 언두 레코드는 사용자 A가 INSERT한 데이터의 언두 로그 인 것 같은데 잘 이해한지 모르겠다.)

그래서 `SELECT…FOR UPDATE` or `SELECT..LOCK IN SHARE`로 조회하는 레코드는 언두 영역의 변경 전 데이터를 가져오는 것이 아니라 현재 레코드의 값을 가져오게 되는 것이다.

### SERIALIZABLE
가장 단순한 격리 수준이자 동시에 가장 엄격한 격리 수준이다.
읽기 작업도 공유잠금(읽기 잠금)을 획득해야만 한다.
한 트랜잭션에서 읽고 쓰는 레코드를 다른 트랜잭션에서는 절대 접근할 수 없는 것이다.
> InnoDB 스토리지 엔진에서는 갭 락과 넥스트 키 락 덕분에 REPEATABLE READ 격리 수준에서도 이미 **PHANTOM READ**가 발생하지 않기 때문에 굳이 SERIALIZABLE이 필요해보이지 않는다.


