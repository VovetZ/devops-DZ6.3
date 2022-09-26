# devops-DZ6.3
# Домашняя работа к занятию "6.3. MySQL"

## Введение

Перед выполнением задания вы можете ознакомиться с 
[дополнительными материалами](https://github.com/netology-code/virt-homeworks/tree/master/additional/README.md).

## Задача 1

Используя docker поднимите инстанс MySQL (версию 8). Данные БД сохраните в volume.

Изучите [бэкап БД](https://github.com/netology-code/virt-homeworks/tree/master/06-db-03-mysql/test_data) и 
восстановитесь из него.

Перейдите в управляющую консоль `mysql` внутри контейнера.

Используя команду `\h` получите список управляющих команд.

Найдите команду для выдачи статуса БД и **приведите в ответе** из ее вывода версию сервера БД.

Подключитесь к восстановленной БД и получите список таблиц из этой БД.

**Приведите в ответе** количество записей с `price` > 300.

В следующих заданиях мы будем продолжать работу с данным контейнером.

## Решение  

Создадим docker-compose.yml файл:  

```yaml
version: '3.7'
services:
  mysql:
    image: mysql:8
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_DATABASE=testdb
    volumes:
      - ./data:/var/lib/mysql
      - ./backup:/data/backup/mysql
    ports:
      - "3306:3306"
    restart: always
```
```bash
$ docker-compose up -d
⋊> ~/DZ6.3 docker-compose up -d
Creating network "dz63_default" with the default driver
Pulling mysql (mysql:8)...
8: Pulling from library/mysql
051f419db9dd: Pull complete
7627573fa82a: Pull complete
a44b358d7796: Pull complete
95753aff4b95: Pull complete
a1fa3bee53f4: Pull complete
f5227e0d612c: Pull complete
b4b4368b1983: Pull complete
f26212810c32: Pull complete
d803d4215f95: Pull complete
d5358a7f7d07: Pull complete
435e8908cd69: Pull complete
Digest: sha256:b9532b1edea72b6cee12d9f5a78547bd3812ea5db842566e17f8b33291ed2921
Status: Downloaded newer image for mysql:8
Creating dz63_mysql_1 ... done
```
Cкопируем файл дампа в запущенный контейнер и восстановим базу
```bash
⋊> ~/D/backup docker cp test_dump.sql dz63_mysql_1:/tmp/
⋊> ~/D/backup sudo docker exec -it dz63_mysql_1 bash
/# mysql -u root -p testdb < /tmp/test_dump.sql
```
Определим версию сервера БД
```sql
bash-4.4# mysql -u root -p testdb     
Enter password: 
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 15
Server version: 8.0.30 MySQL Community Server - GPL

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> \s
--------------
mysql  Ver 8.0.30 for Linux on x86_64 (MySQL Community Server - GPL)

Connection id:      15
Current database:   testdb
Current user:       root@localhost
SSL:            Not in use
Current pager:      stdout
Using outfile:      ''
Using delimiter:    ;
Server version:     8.0.30 MySQL Community Server - GPL
Protocol version:   10
Connection:     Localhost via UNIX socket
Server characterset:    utf8mb4
Db     characterset:    utf8mb4
Client characterset:    latin1
Conn.  characterset:    latin1
UNIX socket:        /var/run/mysqld/mysqld.sock
Binary data as:     Hexadecimal
Uptime:         23 min 11 sec

Threads: 2  Questions: 38  Slow queries: 0  Opens: 161  Flush tables: 3  Open tables: 79  Queries per second avg: 0.027
--------------
```
Выведем список таблиц БД
```sql
mysql> show tables;
+------------------+
| Tables_in_testdb |
+------------------+
| orders           |
+------------------+
1 row in set (0.01 sec)
```
Посчитаем количество записей с price > 300
```sql
mysql> select count(*) from orders where price > 300;
+----------+
| count(*) |
+----------+
|        1 |
+----------+
1 row in set (0.00 sec)
```

## Задача 2

Создайте пользователя test в БД c паролем test-pass, используя:
- плагин авторизации mysql_native_password
- срок истечения пароля - 180 дней 
- количество попыток авторизации - 3 
- максимальное количество запросов в час - 100
- аттрибуты пользователя:
    - Фамилия "Pretty"
    - Имя "James"

Предоставьте привелегии пользователю `test` на операции SELECT базы `test_db`.
    
Используя таблицу INFORMATION_SCHEMA.USER_ATTRIBUTES получите данные по пользователю `test` и 
**приведите в ответе к задаче**.  

## Решение

Создадим пользователя в БД с требуемыми параметрами:  
``` sql
mysql> CREATE USER 'test'@'localhost' 
    ->     IDENTIFIED WITH mysql_native_password BY 'test-pass'
    ->     WITH MAX_CONNECTIONS_PER_HOUR 100
    ->     PASSWORD EXPIRE INTERVAL 180 DAY
    ->     FAILED_LOGIN_ATTEMPTS 3 PASSWORD_LOCK_TIME 2
    ->     ATTRIBUTE '{"first_name":"James", "last_name":"Pretty"}';
Query OK, 0 rows affected (0.01 sec)
```
Предоставим привилегии пользователю test на операции SELECT базы testdb:  

```sql
mysql> GRANT SELECT ON testdb.* to 'test'@'localhost';
Query OK, 0 rows affected, 1 warning (0.01 sec)
```  
Получим данные по пользователю test из INFORMATION_SCHEMA.USER_ATTRIBUTES 
``` sql
mysql> SELECT * from INFORMATION_SCHEMA.USER_ATTRIBUTES where USER = 'test';
+------+-----------+------------------------------------------------+
| USER | HOST      | ATTRIBUTE                                      |
+------+-----------+------------------------------------------------+
| test | localhost | {"last_name": "Pretty", "first_name": "James"} |
+------+-----------+------------------------------------------------+
1 row in set (0.00 sec)
```
## Задача 3

Установите профилирование `SET profiling = 1`.
Изучите вывод профилирования команд `SHOW PROFILES;`.

Исследуйте, какой `engine` используется в таблице БД `test_db` и **приведите в ответе**.

Измените `engine` и **приведите время выполнения и запрос на изменения из профайлера в ответе**:
- на `MyISAM`
- на `InnoDB`  

## Решение

Установим профилирование и изучим вывод профилирования команд:
```sql
mysql> SET profiling = 1;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> show profiles;
+----------+------------+----------------------------------------------------------------------+
| Query_ID | Duration   | Query                                                                |
+----------+------------+----------------------------------------------------------------------+
|        1 | 0.00131600 | SELECT * from INFORMATION_SCHEMA.USER_ATTRIBUTES where USER = 'test' |
|        2 | 0.00074000 | select count(*) from orders where price > 300                        |
+----------+------------+----------------------------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)
```  
В  таблице БД `testdb` используется engine InnoDB:
```sql
mysql> SELECT TABLE_NAME, ENGINE FROM information_schema.TABLES where TABLE_SCHEMA = 'testdb';
+------------+--------+
| TABLE_NAME | ENGINE |
+------------+--------+
| orders     | InnoDB |
+------------+--------+
1 row in set (0.00 sec)
```

Измените engine и **приведите время выполнения и запрос на изменения из профайлера в ответе**:
```sql
mysql> alter table testdb.orders engine='MyISAM';
Query OK, 5 rows affected (0.04 sec)
Records: 5  Duplicates: 0  Warnings: 0

mysql> alter table testdb.orders engine='InnoDB';
Query OK, 5 rows affected (0.05 sec)
Records: 5  Duplicates: 0  Warnings: 0

mysql> show profiles;
+----------+------------+----------------------------------------------------------------------------------------+
| Query_ID | Duration   | Query                                                                                  |
+----------+------------+----------------------------------------------------------------------------------------+
|        1 | 0.00131600 | SELECT * from INFORMATION_SCHEMA.USER_ATTRIBUTES where USER = 'test'                   |
|        2 | 0.00074000 | select count(*) from orders where price > 300                                          |
|        3 | 0.00251700 | SELECT TABLE_NAME, ENGINE FROM information_schema.TABLES where TABLE_SCHEMA = 'testdb' |
|        4 | 0.03654100 | alter table testdb.orders engine='MyISAM'                                              |
|        5 | 0.05159000 | alter table testdb.orders engine='InnoDB'                                              |
+----------+------------+----------------------------------------------------------------------------------------+
5 rows in set, 1 warning (0.00 sec)
```  

## Задача 4 

Изучите файл `my.cnf` в директории /etc/mysql.

Измените его согласно ТЗ (движок InnoDB):
- Скорость IO важнее сохранности данных
- Нужна компрессия таблиц для экономии места на диске
- Размер буффера с незакомиченными транзакциями 1 Мб
- Буффер кеширования 30% от ОЗУ
- Размер файла логов операций 100 Мб

Приведите в ответе измененный файл `my.cnf`.

## Решение 
