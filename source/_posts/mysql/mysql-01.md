---
title: 「MySQL 拾遗」之 增删查改
tags:
  - MySQL
	- 拾遗
categories: Database
keywords: 'mysql'
description: MySQL 基础知识，增删查改
cover: >-
  https://graph.linganmin.cn/201007/f4579b44bb4078b0f0f9bcc986f56d73?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/201007/f4579b44bb4078b0f0f9bcc986f56d73?x-oss-process=image/format,webp/quality,q_40
abbrlink: mysql-crud
date: 2020-07-07 22:03:00
---

就只是把很多年前学的东西捡起来看一下，虽然对于一个`研发工程师`来说，这些都是再基本不过的知识了，但长时间不看呢记忆总会模糊。。。

- 连接数据库

  > mysql -h [数据库地址] -u [用户名] -p

  ```bash
  # 回车后输入密码
  ⚡  mysql -h 127.0.0.1 -u root -p root
  Enter password: root
  Welcome to the MySQL monitor.  Commands end with ; or \g.
  Your MySQL connection id is 5
  Server version: 5.7.22 MySQL Community Server (GPL)

  Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

  Oracle is a registered trademark of Oracle Corporation and/or its
  affiliates. Other names may be trademarks of their respective
  owners.

  Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
  mysql>
  ```

- 查看当前实例中有哪些数据库

  > show databases;

  ```bash
  mysql> show databases;
  +--------------------+
  | Database           |
  +--------------------+
  | information_schema |
  | mysql              |
  | performance_schema |
  | sys                |
  +--------------------+
  5 rows in set (0.02 sec)
  ```

- 创建一个数据库

  > create database [数据库名] charset=[字符集类型];

  ```bash
  mysql> create database my_database charset=utf8mb4;
  Query OK, 1 row affected (0.01 sec)
  ```

- 选择创建的数据库

  > use [数据库名];

  ```bash
  mysql> use my_database ;
  Database changed
  ```

- 查看当前数据库中有哪些表

  > show tables;

  ```bash
  # 空的数据库
  mysql> show tables ;
  Empty set (0.01 sec)
  ```

- 创建一个表

  > create table [表名] ([各种字段...]);

  ```bash
  mysql> create table users (
      ->   `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
      ->   `username` varchar(20) NOT NULL DEFAULT '',
      ->   `gender` tinyint(4) NOT NULL DEFAULT '0' COMMENT "0: 保密，1：男，2：女",
      ->   `status` tinyint(4) NOT NULL DEFAULT '1' COMMENT "1: 正常， 10：禁用",
      ->   `create_time` datetime DEFAULT CURRENT_TIMESTAMP,
      ->   `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
      ->   PRIMARY KEY (`id`)
      -> )ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4;
  Query OK, 0
  ```

- 显示表结构

  > desc [表名];

  ```bash
  mysql> desc users;
  +-------------+------------------+------+-----+-------------------+-----------------------------+
  | Field       | Type             | Null | Key | Default           | Extra                       |
  +-------------+------------------+------+-----+-------------------+-----------------------------+
  | id          | int(11) unsigned | NO   | PRI | NULL              | auto_increment              |
  | username    | varchar(20)      | NO   |     |                   |                             |
  | gender      | tinyint(4)       | NO   |     | 0                 |                             |
  | status      | tinyint(4)       | NO   |     | 1                 |                             |
  | create_time | datetime         | YES  |     | CURRENT_TIMESTAMP |                             |
  | update_time | datetime         | YES  |     | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP |
  +-------------+------------------+------+-----+-------------------+-----------------------------+
  6 rows in set (0.01 sec)

  ```

- 往表中插入记录

  > inster into [表名] (列名1,列名2,...) values(列值1, 列值2,...);

  ```bash
  # 新增单条记录
  mysql> insert into users (username,gender) values ("小下", 1);
  Query OK, 1 row affected (0.00 sec)

  # 新增多条记录
  mysql> insert into users (username,gender) values ("大上", 10), ("saboran",1);
  Query OK, 2 rows affected (0.00 sec)
  Records: 2  Duplicates: 0  Warnings: 0
  ```

- 查询

  - 所有列

    > select * from [表名];

    ```bash
    mysql> select * from users;
    +----+----------+--------+--------+---------------------+---------------------+
    | id | username | gender | status | create_time         | update_time         |
    +----+----------+--------+--------+---------------------+---------------------+
    |  1 | 小下     |      1 |      1 | 2020-10-07 15:35:49 | 2020-10-07 15:35:49 |
    |  2 | 大上     |     10 |      1 | 2020-10-07 15:50:50 | 2020-10-07 15:50:50 |
    |  3 | saboran  |      1 |      1 | 2020-10-07 15:50:50 | 2020-10-07 15:50:50 |
    +----+----------+--------+--------+---------------------+---------------------+
    3 rows in set (0.00 sec)
    ```

  - 去重

    > select distinct [列名] from [表名]

    ```bash
    mysql> select distinct gender from users ;
    +--------+
    | gender |
    +--------+
    |      1 |
    |     10 |
    +--------+
    2 rows in set (0.00 sec)
    ```

  - 统计个数
  
    > select count(*) from [表名];

    ```bash
    mysql> select count(*) from users ;
    +----------+
    | count(*) |
    +----------+
    |        3 |
    +----------+
    1 row in set (0.00 sec)
    ```

  - 指定列

    > select [列名1, 列名2, ...] from [表名];

    ```bash
    mysql> select username, status from users;
    +----------+--------+
    | username | status |
    +----------+--------+
    | 小下     |      1 |
    | 大上     |      1 |
    | saboran  |      1 |
    +----------+--------+
    3 rows in set (0.01 sec)
    ```

  - 带查询条件

    > select * from [表名] where [条件] ; // 查询条件包括： >、<、=、like、not、and 、or 、in 等

    ```bash
    mysql> select * from users where gender = 10 ;
    +----+----------+--------+--------+---------------------+---------------------+
    | id | username | gender | status | create_time         | update_time         |
    +----+----------+--------+--------+---------------------+---------------------+
    |  2 | 大上     |     10 |      1 | 2020-10-07 15:50:50 | 2020-10-07 15:50:50 |
    +----+----------+--------+--------+---------------------+---------------------+
    1 row in set (0.00 sec)
    ```

  - 限制数量

    > select * from [表名] limit [数量];

    ```bash
    mysql> select * from users limit 1 ;
    +----+----------+--------+--------+---------------------+---------------------+
    | id | username | gender | status | create_time         | update_time         |
    +----+----------+--------+--------+---------------------+---------------------+
    |  1 | 小下     |      1 |      1 | 2020-10-07 15:35:49 | 2020-10-07 15:35:49 |
    +----+----------+--------+--------+---------------------+---------------------+
    1 row in set (0.00 sec)
    ```

  - 排序

    > select * from [表名] order by [列名] [排序规则]; // asc: 正序（默认），desc：倒序

    ```bash
    mysql> select * from users order by id desc ;
    +----+----------+--------+--------+---------------------+---------------------+
    | id | username | gender | status | create_time         | update_time         |
    +----+----------+--------+--------+---------------------+---------------------+
    |  3 | saboran  |      1 |      1 | 2020-10-07 15:50:50 | 2020-10-07 15:50:50 |
    |  2 | 大上     |     10 |      1 | 2020-10-07 15:50:50 | 2020-10-07 15:50:50 |
    |  1 | 小下     |      1 |      1 | 2020-10-07 15:35:49 | 2020-10-07 15:35:49 |
    +----+----------+--------+--------+---------------------+---------------------+
    3 rows in set (0.00 sec)
    ```

  - 分组

    > select [列名1], count(*) from users group by [列名1];

    ```bash
    mysql> select gender,count(*) from users  group by gender ;
    +--------+----------+
    | gender | count(*) |
    +--------+----------+
    |      1 |        2 |
    |     10 |        1 |
    +--------+----------+
    2 rows in set (0.01 sec)
    ```

  - Having 子语句

    having 子句是对分组后的结果进行筛选，语法和 where 差不多。

    区别:
     1. having 只能用于`group by`之后，对分组后的结果进行筛选。
     2. where 后面的语句不能使用聚合函数，但 having 可以。
     3. having 后面的条件只能使用`group by`的字段，`group by`的字段也可以使用聚合函数，即按照字段聚合后的条件分组

    where、group by、having、order by 执行顺序:
     1. 先执行 where xxx 进行全表筛选，返回第一个结果集
     2. 执行 group by 对第一个结果集进行分组，返回第二个结果集
     3. 对第二个结果集中的每一组数据执行 select xxx, 返回第三个结果集
     4. 执行 having 对第三个结果集进行筛选，返回第四个结果集
     5. 执行 order by 对第四个结果集排序

     {% note info %}
      执行顺序：where > group by > having > order by
     {% endnote %}

    > select [列名1], count(*) from [表名] where [条件] group by [列名1] having [条件]

    ```bash
    mysql> select gender,count(*) from users where status = 1  group by gender having gender > 0 ;
    +--------+----------+
    | gender | count(*) |
    +--------+----------+
    |      1 |        2 |
    |     10 |        1 |
    +--------+----------+
    2 rows in set (0.00 sec)
    ```

- 更新表中的数据

  > update [表名] set [列名1] = [值1], [列名2] = [值2] where [条件]

  ```bash
  mysql> update users set gender = 2 , status = 10 where id = 2;
  Query OK, 1 row affected (0.00 sec)
  Rows matched: 1  Changed: 1  Warnings: 0
  ```

- 清空表

  > delete from [表名];

- 删除表

  > drop table [表名];

- 事务

  事务具有`原子性(Atomicity)`、`一致性(Consistency)`、`隔离性(Isolation)`、`持久性(Durability)`四个标准属性，缩写为`ACID`

  ```mysql
  -- 提交
  BEGIN;
  INSERT INTO ...
  ...
  COMMIT;
  -- 回滚
  BEGIN;
  ...
  ROLLBACK;
  ```
