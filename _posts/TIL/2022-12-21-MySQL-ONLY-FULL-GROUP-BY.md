---
title: MySQL ONLY FULL GROUP BY
tags: [MySQL]
categories: [TIL]
date: 2022-12-21 22:12:57
---

### 발견 경로
다른 환경 DB에서는 실행되는 쿼리가 로컬 docker DB에서는 실행되지 않는 경우를 발견
```
1Expression #3 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'COLUMN' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by
```

### 원인
-   MySQL 5.7 버전 부터 생긴 `ONLY_FULL_GROUP_BY` 옵션때문에 발생하는 문제.
-   해당 옵션이 Enable 되어 있고, Select 컬럼, Having 조건, Order by 목록이 Group By 절에 지정되지 않는 경우 발생
-   옵션 확인
    -   `SELECT @@GLOBAL.sql_mode;`
    -   `SELECT @@SESSION.sql_mode;`
### 해결
1.  `ANY_VALUE()`
    -   Group by에 지정되지않은 컬럼에 `ANY_VALUE()`
2.  SQL_MODE 옵션 변경
    -   SQL_MODE에서 해당 옵션을 삭제해준다.
        -   현재 `SQL_MODE` 옵션을 확인한다.
        -   `ONLY_FULL_GROUP_BY` 만 제거해준다.
            -   e.g
                -   `SET GLOBAL sql_mode = 'NO_ENGINE_SUBSTITUTION'; *# Global Setting*`
                -   `SET SESSION sql_mode = 'NO_ENGINE_SUBSTITUTION'; *# Only Session*`
---

### References
- [MySQL :: MySQL 8.0 Reference Manual :: 12.20.3 MySQL Handling of GROUP BY](https://dev.mysql.com/doc/refman/8.0/en/group-by-handling.html)
- [MySQL :: MySQL 8.0 Reference Manual :: 5.1.11 Server SQL Modes](https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html#sqlmode_only_full_group_by)