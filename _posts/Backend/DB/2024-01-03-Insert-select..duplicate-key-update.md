---
title: Insert...select duplicate key update Deadlock에 대하여
tags: [mysql]
categories: [DB]
date: 2024-01-03 18:36:49
---
### 문제

`insert....select duplicate key update` 
사용 시 Deadlock이 발생할 수 있음

```sql
insert into member (member_id, name, age)  
select * from member as c  
where name = 'choi'  
on duplicate key update age = c.age + 1;
```

이런 쿼리가 있다고 한다면 실제로 아래처럼 동작을 함
```sql
start transaction ;  
  
select * from member where name = 'choi' lock in share mode ;  
update member set age = age + 1 where name = 'choi';  
  
commit;
```

변경하고자하는 데이터 row에 SharedLcok을 잡은 후 실제 업데이트를 반영할 떄
Exclusive Lock을 획득하게됨

이 때 다른 세션에서 같은 쿼리를 실행하게 되면 서로 Exclusive Lock 획득을 시도하기 떄문에 DeadLock이 발생함

![](/assets/img/b5a3cee57687b989a680446721978936.png)
이렇게 2개의 Session을 열어서 아래 순서로 실행해보면 재현할 수 있다.

순서
1. start transaction 둘 다 실행
2. select ... 둘 다 실행
3. 하나의 세션에서 update... 실행
4. 또 다른 세션에서 update 를 실행

위의 순서로 실행을 해보면 4번에서 deadlock이 발생하는 걸 확인할 수 있다.
![](/assets/img/957b9c46917cadec4be2e99fcfd3a36f.png)

각 세션에서 Shared Lock을 잡고 Exclusive Lock으로 전환하려 할 때 서로의 Shared Lock을 내놓으라고 경합하기 때문이다.

![](/assets/img/d9e37384c8d5a45cfd5eac199a0f80e5.png)
서로 Shared Lock 획득을 기다리다가 Deadlock이 터진다.

> [!info] 
> 각 트랜잭션은 배타적 잠금을 획득하기 전에 다른 트랜잭션이 공유 잠금을 해제할 때까지 기다리고 있습니다. 어느 쪽도 먼저 잠금을 해제하려고 하지 않으므로(다른 쪽이 잠금을 해제하기를 기다리고 있으므로) 교착 상태가 발생합니다.

### 해결 방법

#### Index를 잘 만든다.

#### Shared Lock을 없앤다!

```sql
insert into member (member_id, name, age)  
select * from member as c  
where name = 'choi'  
on duplicate key update age = c.age + 1;
```
이 쿼리가 사실은 
```sql
start transaction ;  
  
select * from member where name = 'choi' lock in share mode ; [1]
update member set age = age + 1 where name = 'choi';  
  
commit;
```
쿼리와 같이 동작한다고 했으니 1번으로 표시한 Select문이 실행될 때 X락을 잡으면 된다.

```sql
insert into member (member_id, name, age)  
select * from member as c  
where name = 'choi' for update
on duplicate key update age = c.age + 1;
```
`~ for update` 구문을 사용하여 X락을 잡도록 한다.

- S Lock
	![](/assets/img/1a9a7e9b398d87b0b1aa493edf5e3d2b.png)
	S Lock 이후에 X Lock을 잡았다.

- for Update 적용
	![](/assets/img/056e11f438b9dd9cdca25e67f7d781d3.png)
	바로 X Lock이 잡힌걸 알 수 있다.
