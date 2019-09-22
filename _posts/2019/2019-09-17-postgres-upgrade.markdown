---
layout:     post
title:      "PostgreSQL 数据升级"
subtitle:   "Data Upgrade for PostgreSQL"
date:       2019-09-17
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - DB
---

## PostgreSQL Installation

为了演示后续的实验，可以通过 PostgreSQL 镜像，在本地快速搭建一个 PostgreSQL 的环境。

```shell
$ docker run -d --name pg -v data:/var/lib/postgresql/data -p 5432:5432 -e POSTGRES_USER=postgres -e POSTGRES_DB=postgres -e PGDATA=/var/lib/postgresql/data/pgdata postgres:12
$ docker exec -it pg bash
root@0e301a385d96:/# su postgres
postgres@0e301a385d96:/$ psql -U postgres -d postgres
psql (12beta4 (Debian 12~beta4-1.pgdg100+1))
Type "help" for help.

postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(3 rows)

postgres=# \d
Did not find any relations.
```

## Example

搭建好的 PostgreSQL 中是没有任何数据的。先要创建一张测试的表，并初始化一些数据。

```shell
postgres=# CREATE TABLE employees (
postgres(#     id                SERIAL              NOT NULL PRIMARY KEY,
postgres(#     first_name        VARCHAR (50)        NOT NULL,
postgres(#     last_name         VARCHAR (50)        NOT NULL,
postgres(#     salary            numeric             NULL CHECK(salary > 0),
postgres(#     created_at        TIMESTAMPTZ         NOT NULL DEFAULT now(),
postgres(#     updated_at        TIMESTAMPTZ         NOT NULL DEFAULT now()
postgres(# );
CREATE TABLE

postgres=# \d employees
                                       Table "public.employees"
   Column   |           Type           | Collation | Nullable |                Default
------------+--------------------------+-----------+----------+---------------------------------------
 id         | integer                  |           | not null | nextval('employees_id_seq'::regclass)
 first_name | character varying(50)    |           | not null |
 last_name  | character varying(50)    |           | not null |
 salary     | numeric                  |           |          |
 created_at | timestamp with time zone |           | not null | now()
 updated_at | timestamp with time zone |           | not null | now()
Indexes:
    "employees_pkey" PRIMARY KEY, btree (id)
Check constraints:
    "employees_salary_check" CHECK (salary > 0::numeric)

postgres=# INSERT INTO employees
postgres-#   ( first_name, last_name, salary )
postgres-# VALUES
postgres-#   ('Robin', 'Yue', 100),
postgres-#   ('Jack', 'Ma', 100);
INSERT 0 2

postgres=# INSERT INTO employees
postgres-#   ( first_name, last_name )
postgres-# VALUES
postgres-#   ('Pony', 'Ma');
INSERT 0 1

postgres=# SELECT * FROM employees;
 id | first_name | last_name | salary |          created_at           |          updated_at
----+------------+-----------+--------+-------------------------------+-------------------------------
  1 | Robin      | Yue       |    100 | 2019-09-17 12:52:42.470909+00 | 2019-09-17 12:52:42.470909+00
  2 | Jack       | Ma        |    100 | 2019-09-17 12:52:42.470909+00 | 2019-09-17 12:52:42.470909+00
  3 | Pony       | Ma        |        | 2019-09-17 12:55:22.71628+00  | 2019-09-17 12:55:22.71628+00
(3 rows)
```

## Upgrade Data

现在要将 age 列插入到表中的 salary 后面，同时在表的最后位置增加 notes 列。
PostgreSQL 支持通过 `ALTER TABLE addresses ADD COLUMN age` 命令往表中插入列，默认都是插入到表的最后一列，但是不支持在固定位置插入列。

为了满足在固定位置插入列，官方文档 [Alter Column Position](https://wiki.postgresql.org/wiki/Alter_column_position) 提供了两种方案：
* Recreate the table：按期望的列顺序新建一张表，然后将原来数据的表更新到新表中，将老的表删除，解决表之间的依赖关系，最后将新表重命名为原来表的名字。
* Add columns and move data：在表末尾增加需要添加的列以及要移动的列，然后迁移要移动的列的数据，最后将老的列删除，并将新的列重命名为原来列的名字。

比较上面两种方法，第二种方法通过移动列比第一种方法通过重建表更容易一些。下面通过演示如何通过第二种方法实现。

升级脚本 `pg-upgrade.sql`：

```sql
/*
Add age number column at specified position by moving data.
Refer to: https://wiki.postgresql.org/wiki/Alter_column_position
*/
ALTER TABLE employees ADD COLUMN age numeric NULL CHECK (age > 18 AND age <= 65);
ALTER TABLE employees ADD COLUMN cat TIMESTAMPTZ NOT NULL DEFAULT now();
ALTER TABLE employees ADD COLUMN uat TIMESTAMPTZ NOT NULL DEFAULT now();
UPDATE employees SET cat=created_at, uat=updated_at;
ALTER TABLE employees DROP COLUMN created_at CASCADE, DROP COLUMN updated_at CASCADE;
ALTER TABLE employees RENAME COLUMN cat TO created_at;
ALTER TABLE employees RENAME COLUMN uat TO updated_at;

/*
Add notes number column at the end.
*/
ALTER TABLE employees ADD COLUMN notes TEXT NULL CHECK (0 <= char_length(notes) AND char_length(notes) <= 256);
```

升级命令：

```shell
$ docker cp pg-upgrade.sql pg:pg-upgrade.sql
$ docker exec -it pg bash
root@0e301a385d96:/# su postgres
postgres@0e301a385d96:/$ ls
bin  boot  dev  docker-entrypoint-initdb.d  docker-entrypoint.sh  etc  home  lib  lib64  media  mnt  opt  pg-upgrade.sql  proc  root  run  sbin  srv  sys  tmp  usr  var
postgres@0e301a385d96:/$ psql -f pg-upgrade.sql
ALTER TABLE
ALTER TABLE
ALTER TABLE
UPDATE 3
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
postgres@0e301a385d96:/$ psql
psql (12beta4 (Debian 12~beta4-1.pgdg100+1))
Type "help" for help.

postgres=# \d employees;
                                       Table "public.employees"
   Column   |           Type           | Collation | Nullable |                Default
------------+--------------------------+-----------+----------+---------------------------------------
 id         | integer                  |           | not null | nextval('employees_id_seq'::regclass)
 first_name | character varying(50)    |           | not null |
 last_name  | character varying(50)    |           | not null |
 salary     | numeric                  |           |          |
 age        | numeric                  |           |          |
 created_at | timestamp with time zone |           | not null | now()
 updated_at | timestamp with time zone |           | not null | now()
 notes      | text                     |           |          |
Indexes:
    "employees_pkey" PRIMARY KEY, btree (id)
Check constraints:
    "employees_age_check" CHECK (age > 18::numeric AND age <= 65::numeric)
    "employees_notes_check" CHECK (0 <= char_length(notes) AND char_length(notes) <= 256)
    "employees_salary_check" CHECK (salary > 0::numeric)

postgres=# SELECT * FROM employees;
 id | first_name | last_name | salary | age |          created_at           |          updated_at           | notes
----+------------+-----------+--------+-----+-------------------------------+-------------------------------+-------
  1 | Robin      | Yue       |    100 |     | 2019-09-17 12:52:42.470909+00 | 2019-09-17 12:52:42.470909+00 |
  2 | Jack       | Ma        |    100 |     | 2019-09-17 12:52:42.470909+00 | 2019-09-17 12:52:42.470909+00 |
  3 | Pony       | Ma        |        |     | 2019-09-17 12:55:22.71628+00  | 2019-09-17 12:55:22.71628+00  |
(3 rows)
```

## Reference

- [Alter Column Position](https://wiki.postgresql.org/wiki/Alter_column_position)
