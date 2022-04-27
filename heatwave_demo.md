# HeatWave & Autopolot 데모 시나리오

## 1. MySQL Database Service 쿼리 테스트

현재 데이터는 MySQL 데이터베이스에만 있고, HeatWave 클러스터는 구성은 되었지만 데이터는 HeatWave에 로딩되어 있지 않은 상태입니다.
HeatWave 에 클러스터에 데이터를 로딩하기 전 MySQL 에서 쿼리가 수행되었을 때의 성능을 살펴보겠습니다.

아래 과정은 Shell 환경에서 MySQL Shell Client를 사용하거나 
MySQL Workbench 를 사용하여 테스트를 수행할 수 있습니다.  
문서에서는 MySQL Workbench 와 MySQL Shell Client를 모두 사용하여 테스트를 진행하였습니다.


**MySQL Workbench 로 MySQL DB 접속**  

접속 후 다음의 SQL 을 수행하여 수행 시간을 확인합니다.  
아래 쿼리는 월별 예약 건수 및 금액을 구하는 간단한 쿼리입니다.

```sql
use airportdb;

select date_format(f.departure, '%Y-%m') as "Month", 
       count(b.booking_id) as Booking_Count,
       sum(b.price) as Total_Price
from flight f , booking b, passengerdetails p
where b.flight_id = f.flight_id
  and b.passenger_id = p.passenger_id
  and p.country in ("USA", "FRANCE", "ITALY")
group by date_format(f.departure, '%Y-%m')
order by Booking_Count desc
;
```

수행 결과

```
+---------+---------------+--------------+
| Month   | Booking_Count | Total_Price  |
+---------+---------------+--------------+
| 2015-08 |        727663 | 180172042.49 |
| 2015-07 |        726223 | 179477391.51 |
| 2015-06 |        702744 | 173904908.31 |
| 2015-09 |         24094 |   5947603.47 |
+---------+---------------+--------------+
4 rows in set (13.1477 sec)
```

<!-- MySQL 
```sql
# [MDS-Client] Shell
$ mysqlsh --mysql admin@10.1.1.136 --sql
Password: Oracle#1234

# MySQL> Prompt 

use airportdb;

select date_format(f.departure, '%Y-%m') as "Month", 
       count(b.booking_id) as Booking_Count,
       sum(b.price) as Total_Price
from flight f , booking b, passengerdetails p
where b.flight_id = f.flight_id
  and b.passenger_id = p.passenger_id
  and p.country in ("USA", "FRANCE", "ITALY")
group by date_format(f.departure, '%Y-%m')
order by Booking_Count desc;
```

수행 결과

```
MySQL  10.1.1.136:3306 ssl  SQL > use airportdb;
Default schema set to `airportdb`.
Fetching table and column names from `airportdb` for auto-completion... Press ^C to stop.
 MySQL  10.1.1.136:3306 ssl  airportdb  SQL > select date_format(f.departure, '%Y-%m') as "Month",
                                           ->        count(b.booking_id) as Booking_Count,
                                           ->        sum(b.price) as Total_Price
                                           -> from flight f , booking b, passengerdetails p
                                           -> where b.flight_id = f.flight_id
                                           ->   and b.passenger_id = p.passenger_id
                                           ->   and p.country in ("USA", "FRANCE", "ITALY")
                                           -> group by date_format(f.departure, '%Y-%m')
                                           -> order by Booking_Count desc
                                           -> ;
+---------+---------------+--------------+
| Month   | Booking_Count | Total_Price  |
+---------+---------------+--------------+
| 2015-08 |        727663 | 180172042.49 |
| 2015-07 |        726223 | 179477391.51 |
| 2015-06 |        702744 | 173904908.31 |
| 2015-09 |         24094 |   5947603.47 |
+---------+---------------+--------------+
4 rows in set (13.1477 sec)
```
-->

쿼리 수행 시간이 약 13초 정도 소요되는 것을 확인할 수 있습니다.  
그럼, HeatWave 에 데이터를 로딩한 후 수행시간이 어떻게 변화하는지 살펴보도록 하겠습니다.


## 2. Loading Data to HeatWave Cluster

MySQL DB 에 있는 데이터를 HeatWave Cluster 로 로딩하는 과정을 살펴보겠습니다.
Loading 작업은 HeatWave Autopiot 기능 중의 하나인 **Auto Parallel Loading** 기능을 사용하여 아주 쉽고 빠르게 로딩 작업을 수행할 수 있습니다.
 

<!--
>> [참고] Unloading Table
```sql
use airportdb;

alter table airline           SECONDARY_UNLOAD;
alter table airplane          SECONDARY_UNLOAD;
alter table airplane_type     SECONDARY_UNLOAD;
alter table airport           SECONDARY_UNLOAD;
alter table airport_geo       SECONDARY_UNLOAD;
alter table airport_reachable SECONDARY_UNLOAD;
alter table booking           SECONDARY_UNLOAD;
alter table employee          SECONDARY_UNLOAD;
alter table flight            SECONDARY_UNLOAD;
alter table flight_log        SECONDARY_UNLOAD;
alter table flightschedule    SECONDARY_UNLOAD;
alter table passenger         SECONDARY_UNLOAD;
alter table passengerdetails  SECONDARY_UNLOAD;
alter table weatherdata       SECONDARY_UNLOAD;
```

-->


### 2.1. AutoPilot - Auto Parallel Loading - Dry Run


실제 로딩 작업을 수행하지 않고 병렬 로딩 과정 검증 및 스크립트 생성만 하는 과정입니다.
`sys.heatwave_load()` 함수 수행시 Mode 를 `dryrun` 으로 지정하면 실제 로딩 작업은 수행하지 않고
로딩 예상 시간 및 오류 발생 여부 등을 확인할 수 있고, 로딩 스크립트도 생성할 수 있습니다.

MySQL Shell 을 통해 MySQL Database 에 접속한 후 다음을 수행합니다.
```
# [MDS-Client] Shell
$ mysqlsh --mysql admin@10.1.1.136 --sql
Password: Oracle#1234

CALL sys.heatwave_load(JSON_ARRAY("airportdb"), JSON_OBJECT("mode","dryrun"));
```

수행 결과

```
+------------------------------------------+
| INITIALIZING HEATWAVE AUTO PARALLEL LOAD |
+------------------------------------------+
| Version: 1.26                            |
|                                          |
| Load Mode: dryrun                        |
| Load Policy: disable_unsupported_columns |
| Output Mode: normal                      |
|                                          |
+------------------------------------------+
6 rows in set (0.0097 sec)

+------------------------------------------------------------------------+
| OFFLOAD ANALYSIS                                                       |
+------------------------------------------------------------------------+
| Verifying input schemas: 1                                             |
| User excluded items: 0                                                 |
|                                                                        |
| SCHEMA                       OFFLOADABLE    OFFLOADABLE     SUMMARY OF |
| NAME                              TABLES        COLUMNS     ISSUES     |
| ------                       -----------    -----------     ---------- |
| `airportdb`                           15            119                |
|                                                                        |
| Total offloadable schemas: 1                                           |
|                                                                        |
+------------------------------------------------------------------------+
10 rows in set (0.0097 sec)

+-----------------------------------------------------------------------------------------------------------------------------+
| CAPACITY ESTIMATION                                                                                                         |
+-----------------------------------------------------------------------------------------------------------------------------+
| Default load pool for tables: TRANSACTIONAL                                                                                 |
| Default encoding for string columns: VARLEN (unless specified in the schema)                                                |
| Estimating memory footprint for 1 schema(s)                                                                                 |
|                                                                                                                             |
|                                TOTAL       ESTIMATED       ESTIMATED       TOTAL     DICTIONARY      VARLEN       ESTIMATED |
| SCHEMA                   OFFLOADABLE   HEATWAVE NODE      MYSQL NODE      STRING        ENCODED     ENCODED            LOAD |
| NAME                          TABLES       FOOTPRINT       FOOTPRINT     COLUMNS        COLUMNS     COLUMNS            TIME |
| ------                   -----------       ---------       ---------     -------     ----------     -------       --------- |
| `airportdb`                       15        1.29 GiB       60.00 MiB          40              0          40         17.00 s |
|                                                                                                                             |
| Sufficient MySQL host memory available to load all tables.                                                                  |
| Sufficient HeatWave cluster memory available to load all tables.                                                            |
|                                                                                                                             |
+-----------------------------------------------------------------------------------------------------------------------------+
13 rows in set (0.0097 sec)

+-------------------------------------------------------------------------------------------------------+
| LOAD SCRIPT GENERATION                                                                                |
+-------------------------------------------------------------------------------------------------------+
| Dryrun mode only generates the load script                                                            |
| Set mode to "normal" in options to load tables                                                        |
|                                                                                                       |
| Retrieve load script containing 31 generated DDL commands using the query below:                      |
|   SELECT log->>"$.sql" AS "Load Script" FROM sys.heatwave_load_report WHERE type = "sql" ORDER BY id; |
|                                                                                                       |
| Applying changes will take approximately 17.00 s                                                      |
|                                                                                                       |
| Caution: Executing the generated load script will affect the secondary engine flags in the schema     |
|                                                                                                       |
+-------------------------------------------------------------------------------------------------------+
10 rows in set (0.0097 sec)

Query OK, 0 rows affected (0.0097 sec)
```

**스크립트 생성 SQL 수행**

다음의 SQL 문을 수행하여 실제 로딩 작업을 위한 스크립트를 확인할 수 있습니다.  
아래 결과를 보시면 테이블 별로 `innodb_parallel_read_threads` 값이 다르게 설정된 것을 확인할 수 있습니다.
테이블 별로 사이즈에 맞게 최적의 병렬도를 설정해 줍니다.
생성된 스크립트를 활용하여 테이블별로 로딩작업을 수행하거나 수정하여 로딩작업을 수행할 수 있습니다.

```sql
SELECT log->>"$.sql" AS "Load Script" FROM sys.heatwave_load_report WHERE type = "sql" ORDER BY id;
```

수행 결과
```sql
+-------------------------------------------------------------+
| Load Script                                                 |
+-------------------------------------------------------------+
| SET SESSION innodb_parallel_read_threads = 1;               |
| ALTER TABLE `airportdb`.`airline` SECONDARY_LOAD;           |
| SET SESSION innodb_parallel_read_threads = 1;               |
| ALTER TABLE `airportdb`.`airplane` SECONDARY_LOAD;          |
| SET SESSION innodb_parallel_read_threads = 1;               |
| ALTER TABLE `airportdb`.`airplane_type` SECONDARY_LOAD;     |
| SET SESSION innodb_parallel_read_threads = 1;               |
| ALTER TABLE `airportdb`.`airport` SECONDARY_LOAD;           |
| SET SESSION innodb_parallel_read_threads = 1;               |
| ALTER TABLE `airportdb`.`airport_geo` SECONDARY_LOAD;       |
| SET SESSION innodb_parallel_read_threads = 1;               |
| ALTER TABLE `airportdb`.`airport_reachable` SECONDARY_LOAD; |
| SET SESSION innodb_parallel_read_threads = 32;              |
| ALTER TABLE `airportdb`.`booking` SECONDARY_LOAD;           |
| SET SESSION innodb_parallel_read_threads = 1;               |
| ALTER TABLE `airportdb`.`employee` SECONDARY_LOAD;          |
| SET SESSION innodb_parallel_read_threads = 9;               |
| ALTER TABLE `airportdb`.`flight` SECONDARY_LOAD;            |
| SET SESSION innodb_parallel_read_threads = 1;               |
| ALTER TABLE `airportdb`.`flight_log` SECONDARY_LOAD;        |
| SET SESSION innodb_parallel_read_threads = 1;               |
| ALTER TABLE `airportdb`.`flightschedule` SECONDARY_LOAD;    |
| SET SESSION innodb_parallel_read_threads = 1;               |
| ALTER TABLE `airportdb`.`passenger` SECONDARY_LOAD;         |
| SET SESSION innodb_parallel_read_threads = 1;               |
| ALTER TABLE `airportdb`.`passengerdetails` SECONDARY_LOAD;  |
| SET SESSION innodb_parallel_read_threads = 18;              |
| ALTER TABLE `airportdb`.`weatherdata` SECONDARY_LOAD;       |
+-------------------------------------------------------------+
28 rows in set (0.0010 sec)
```

사용자는 위에서 생성된 스크립트를 참조하여 필요할 경우 스크립트를 수정하여 테이블 별로 로딩작업을 수행할 수 있습니다.  

### 2.2. AutoPilot - Auto Parallel Loading 수행

그럼 이제 Auto Parallel Loading 기능을 이용하여 HeatWave Cluster로 MySQL DB 에 있는 테이블들을 로딩해 보도록 하겠습니다.  
아래에서 보는 바와 같이 하나의 프로시져만 수행하면 되므로 아주 쉽고 간단하게 로딩 작업을 수행할 수 있습니다.
그리고 테이블 별로 자동 병렬 로딩을 수행하기 떄문에 아주 빠르게 로딩 작업을 완료할 수 있습니다.


```sql
CALL sys.heatwave_load(JSON_ARRAY('airportdb'), NULL);
```

수행 결과

```
+------------------------------------------+
| INITIALIZING HEATWAVE AUTO PARALLEL LOAD |
+------------------------------------------+
| Version: 1.26                            |
|                                          |
| Load Mode: normal                        |
| Load Policy: disable_unsupported_columns |
| Output Mode: normal                      |
|                                          |
+------------------------------------------+
6 rows in set (0.0080 sec)

+------------------------------------------------------------------------+
| OFFLOAD ANALYSIS                                                       |
+------------------------------------------------------------------------+
| Verifying input schemas: 1                                             |
| User excluded items: 0                                                 |
|                                                                        |
| SCHEMA                       OFFLOADABLE    OFFLOADABLE     SUMMARY OF |
| NAME                              TABLES        COLUMNS     ISSUES     |
| ------                       -----------    -----------     ---------- |
| `airportdb`                           15            119                |
|                                                                        |
| Total offloadable schemas: 1                                           |
|                                                                        |
+------------------------------------------------------------------------+
10 rows in set (0.0080 sec)

+-----------------------------------------------------------------------------------------------------------------------------+
| CAPACITY ESTIMATION                                                                                                         |
+-----------------------------------------------------------------------------------------------------------------------------+
| Default load pool for tables: TRANSACTIONAL                                                                                 |
| Default encoding for string columns: VARLEN (unless specified in the schema)                                                |
| Estimating memory footprint for 1 schema(s)                                                                                 |
|                                                                                                                             |
|                                TOTAL       ESTIMATED       ESTIMATED       TOTAL     DICTIONARY      VARLEN       ESTIMATED |
| SCHEMA                   OFFLOADABLE   HEATWAVE NODE      MYSQL NODE      STRING        ENCODED     ENCODED            LOAD |
| NAME                          TABLES       FOOTPRINT       FOOTPRINT     COLUMNS        COLUMNS     COLUMNS            TIME |
| ------                   -----------       ---------       ---------     -------     ----------     -------       --------- |
| `airportdb`                       15        1.29 GiB       60.00 MiB          40              0          40         17.00 s |
|                                                                                                                             |
| Sufficient MySQL host memory available to load all tables.                                                                  |
| Sufficient HeatWave cluster memory available to load all tables.                                                            |
|                                                                                                                             |
+-----------------------------------------------------------------------------------------------------------------------------+
13 rows in set (0.0080 sec)

+---------------------------------------------------------------------------------------------------------------------------------------+
| EXECUTING LOAD                                                                                                                        |
+---------------------------------------------------------------------------------------------------------------------------------------+
| HeatWave Load script generated                                                                                                        |
|   Retrieve load script containing 31 generated DDL command(s) using the query below:                                                  |
|   SELECT log->>"$.sql" AS "Load Script" FROM sys.heatwave_load_report WHERE type = "sql" ORDER BY id;                                 |
|                                                                                                                                       |
| Adjusting load parallelism dynamically per table                                                                                      |
| Using current parallelism of 32 thread(s) as maximum                                                                                  |
|                                                                                                                                       |
| Using SQL_MODE: ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION |
|                                                                                                                                       |
| Proceeding to load 15 tables into HeatWave                                                                                            |
|                                                                                                                                       |
| Applying changes will take approximately 16.98 s                                                                                      |
|                                                                                                                                       |
+---------------------------------------------------------------------------------------------------------------------------------------+
13 rows in set (0.0080 sec)

+----------------------------------------+
| LOADING TABLE                          |
+----------------------------------------+
| TABLE (1 of 15): `airportdb`.`airline` |
| Commands executed successfully: 2 of 2 |
| Warnings encountered: 0                |
| Table loaded successfully!             |
|   Total columns loaded: 4              |
|   Table loaded using 1 thread(s)       |
|                                        |
+----------------------------------------+
7 rows in set (0.0080 sec)

+-----------------------------------------+
| LOADING TABLE                           |
+-----------------------------------------+
| TABLE (2 of 15): `airportdb`.`airplane` |
| Commands executed successfully: 2 of 2  |
| Warnings encountered: 0                 |
| Table loaded successfully!              |
|   Total columns loaded: 4               |
|   Table loaded using 1 thread(s)        |
|                                         |
+-----------------------------------------+
7 rows in set (0.0080 sec)

+----------------------------------------------+
| LOADING TABLE                                |
+----------------------------------------------+
| TABLE (3 of 15): `airportdb`.`airplane_type` |
| Commands executed successfully: 2 of 2       |
| Warnings encountered: 0                      |
| Table loaded successfully!                   |
|   Total columns loaded: 3                    |
|   Table loaded using 1 thread(s)             |
|                                              |
+----------------------------------------------+
7 rows in set (0.0080 sec)

+----------------------------------------+
| LOADING TABLE                          |
+----------------------------------------+
| TABLE (4 of 15): `airportdb`.`airport` |
| Commands executed successfully: 2 of 2 |
| Warnings encountered: 0                |
| Table loaded successfully!             |
|   Total columns loaded: 4              |
|   Table loaded using 1 thread(s)       |
|                                        |
+----------------------------------------+
7 rows in set (0.0080 sec)

+--------------------------------------------+
| LOADING TABLE                              |
+--------------------------------------------+
| TABLE (5 of 15): `airportdb`.`airport_geo` |
| Commands executed successfully: 2 of 2     |
| Warnings encountered: 0                    |
| Table loaded successfully!                 |
|   Total columns loaded: 6                  |
|   Table loaded using 1 thread(s)           |
|                                            |
+--------------------------------------------+
7 rows in set (0.0080 sec)

+--------------------------------------------------+
| LOADING TABLE                                    |
+--------------------------------------------------+
| TABLE (6 of 15): `airportdb`.`airport_reachable` |
| Commands executed successfully: 2 of 2           |
| Warnings encountered: 0                          |
| Table loaded successfully!                       |
|   Total columns loaded: 2                        |
|   Table loaded using 1 thread(s)                 |
|                                                  |
+--------------------------------------------------+
7 rows in set (0.0080 sec)

+----------------------------------------+
| LOADING TABLE                          |
+----------------------------------------+
| TABLE (7 of 15): `airportdb`.`booking` |
| Commands executed successfully: 2 of 2 |
| Warnings encountered: 0                |
| Table loaded successfully!             |
|   Total columns loaded: 5              |
|   Table loaded using 32 thread(s)      |
|                                        |
+----------------------------------------+
7 rows in set (0.0080 sec)

+-----------------------------------------+
| LOADING TABLE                           |
+-----------------------------------------+
| TABLE (8 of 15): `airportdb`.`employee` |
| Commands executed successfully: 2 of 2  |
| Warnings encountered: 0                 |
| Table loaded successfully!              |
|   Total columns loaded: 15              |
|   Table loaded using 1 thread(s)        |
|                                         |
+-----------------------------------------+
7 rows in set (0.0080 sec)

+----------------------------------------+
| LOADING TABLE                          |
+----------------------------------------+
| TABLE (9 of 15): `airportdb`.`flight`  |
| Commands executed successfully: 2 of 2 |
| Warnings encountered: 0                |
| Table loaded successfully!             |
|   Total columns loaded: 8              |
|   Table loaded using 9 thread(s)       |
|                                        |
+----------------------------------------+
7 rows in set (0.0080 sec)

+--------------------------------------------+
| LOADING TABLE                              |
+--------------------------------------------+
| TABLE (10 of 15): `airportdb`.`flight_log` |
| Commands executed successfully: 2 of 2     |
| Warnings encountered: 0                    |
| Table loaded successfully!                 |
|   Total columns loaded: 19                 |
|   Table loaded using 1 thread(s)           |
|                                            |
+--------------------------------------------+
7 rows in set (0.0080 sec)

+------------------------------------------------+
| LOADING TABLE                                  |
+------------------------------------------------+
| TABLE (11 of 15): `airportdb`.`flightschedule` |
| Commands executed successfully: 2 of 2         |
| Warnings encountered: 0                        |
| Table loaded successfully!                     |
|   Total columns loaded: 13                     |
|   Table loaded using 1 thread(s)               |
|                                                |
+------------------------------------------------+
7 rows in set (0.0080 sec)

+-------------------------------------------+
| LOADING TABLE                             |
+-------------------------------------------+
| TABLE (12 of 15): `airportdb`.`passenger` |
| Commands executed successfully: 2 of 2    |
| Warnings encountered: 0                   |
| Table loaded successfully!                |
|   Total columns loaded: 4                 |
|   Table loaded using 1 thread(s)          |
|                                           |
+-------------------------------------------+
7 rows in set (0.0080 sec)

+--------------------------------------------------+
| LOADING TABLE                                    |
+--------------------------------------------------+
| TABLE (13 of 15): `airportdb`.`passengerdetails` |
| Commands executed successfully: 2 of 2           |
| Warnings encountered: 0                          |
| Table loaded successfully!                       |
|   Total columns loaded: 9                        |
|   Table loaded using 1 thread(s)                 |
|                                                  |
+--------------------------------------------------+
7 rows in set (0.0080 sec)

+------------------------------------------------------------+
| LOADING TABLE                                              |
+------------------------------------------------------------+
| TABLE (14 of 15): `airportdb`.`supplementary_demographics` |
| Commands executed successfully: 3 of 3                     |
| Warnings encountered: 0                                    |
| Table loaded successfully!                                 |
|   Total columns loaded: 14                                 |
|   Table loaded using 1 thread(s)                           |
|                                                            |
+------------------------------------------------------------+
7 rows in set (0.0080 sec)

+---------------------------------------------+
| LOADING TABLE                               |
+---------------------------------------------+
| TABLE (15 of 15): `airportdb`.`weatherdata` |
| Commands executed successfully: 2 of 2      |
| Warnings encountered: 0                     |
| Table loaded successfully!                  |
|   Total columns loaded: 9                   |
|   Table loaded using 18 thread(s)           |
|                                             |
+---------------------------------------------+
7 rows in set (0.0080 sec)

+-------------------------------------------------------------------------------+
| LOAD SUMMARY                                                                  |
+-------------------------------------------------------------------------------+
|                                                                               |
| SCHEMA                          TABLES       TABLES      COLUMNS         LOAD |
| NAME                            LOADED       FAILED       LOADED     DURATION |
| ------                          ------       ------      -------     -------- |
| `airportdb`                         15            0          119      25.48 s |
|                                                                               |
+-------------------------------------------------------------------------------+
6 rows in set (0.0080 sec)

Query OK, 0 rows affected (0.0080 sec)    
```

**HeatWave Cluster 에 로딩된 테이블 정보 확인**

이제 HeatWave 에 테이블들이 정상적으로 로딩되었는지 확인해 보도록 하겠습니다.   
쿼리 수행 결과 LOAD_STATUS 컬럼 값이 **AVAIL_RPDGSTABSTATE** 값이면 정상적으로 로딩이 완료된 상태입니다. 


```sql
SELECT NAME, LOAD_STATUS, nrows
FROM   performance_schema.rpd_tables,
       performance_schema.rpd_table_id 
WHERE  rpd_tables.ID = rpd_table_id.ID;

```

수행 결과

```sql
 MySQL  10.1.1.136:3306 ssl  airportdb  SQL > SELECT NAME, LOAD_STATUS, nrows
                                           -> FROM   performance_schema.rpd_tables,
                                                     performance_schema.rpd_table_id
                                           -> WHERE  rpd_tables.ID = rpd_table_id.ID;
+--------------------------------------+---------------------+----------+
| NAME                                 | LOAD_STATUS         | nrows    |
+--------------------------------------+---------------------+----------+
| airportdb.flight_log                 | AVAIL_RPDGSTABSTATE |        0 |
| airportdb.airport_geo                | AVAIL_RPDGSTABSTATE |     9854 |
| airportdb.flight                     | AVAIL_RPDGSTABSTATE |   462553 |
| airportdb.passengerdetails           | AVAIL_RPDGSTABSTATE |    36095 |
| airportdb.passenger                  | AVAIL_RPDGSTABSTATE |    36095 |
| airportdb.airplane                   | AVAIL_RPDGSTABSTATE |     5583 |
| airportdb.weatherdata                | AVAIL_RPDGSTABSTATE |  4626432 |
| airportdb.flightschedule             | AVAIL_RPDGSTABSTATE |     9881 |
| airportdb.booking                    | AVAIL_RPDGSTABSTATE | 54304619 |
| airportdb.employee                   | AVAIL_RPDGSTABSTATE |     1000 |
| airportdb.airplane_type              | AVAIL_RPDGSTABSTATE |      342 |
| airportdb.supplementary_demographics | AVAIL_RPDGSTABSTATE |     4500 |
| airportdb.airport                    | AVAIL_RPDGSTABSTATE |     9854 |
| airportdb.airline                    | AVAIL_RPDGSTABSTATE |      113 |
| airportdb.airport_reachable          | AVAIL_RPDGSTABSTATE |        0 |
+--------------------------------------+---------------------+----------+
15 rows in set (0.0009 sec)
```

## 3. 쿼리 성능 테스트

### 3.1. 쿼리 성능 확인

앞에서 테스트했던 동일한 쿼리를 다시 한번 수행해 보도록 하겠습니다.  
현재 접속되어 있는 MySQL Workbench 창에서 동일한 SQL 문을 재수행 해 보도록 하겠습니다.

```sql
select date_format(f.departure, '%Y-%m') as "Month", 
       count(b.booking_id) as Booking_Count,
       sum(b.price) as Total_Price
from flight f , booking b, passengerdetails p
where b.flight_id = f.flight_id
  and b.passenger_id = p.passenger_id
  and p.country in ("USA", "FRANCE", "ITALY")
group by date_format(f.departure, '%Y-%m')
order by Booking_Count desc
;
```

수행 결과

```
+---------+---------------+--------------+
| Month   | Booking_Count | Total_Price  |
+---------+---------------+--------------+
| 2015-08 |        727663 | 180172042.49 |
| 2015-07 |        726223 | 179477391.51 |
| 2015-06 |        702744 | 173904908.31 |
| 2015-09 |         24094 |   5947603.47 |
+---------+---------------+--------------+
4 rows in set (0.1944 sec)
```

동일한 쿼리 수행 시 HeatWave 에 데이터 로딩 전에는 약 13초 가량 소요되던 쿼리가 1초 이내 수행되는 것을 확인할 수 있습니다.  
쿼리 수행 결과가 13초 vs. 1초 이내 정도로 수십배 정도만 차이가 나는 것으로 보일 수 있지만, 
해당 쿼리는 테스트를 위해 COUNTRY 조건을 추가하여 전체 5천만건 데이터 중 200만건 정도의 데이터만 사용하도록 수행하였습니다.
만약 5천만건 데이터 전체를 사용할 경우 성능 차이는 훨씬 크게 발생합니다.  
수행 시간을 예상해 보면, HeatWave 를 사용하지 않는다면 약 5분 정도가 소요될 것입니다.

그럼 쿼리를 약간 수정하여 5천만건 전체 데이터에 대한 분석 작업을 수행할 때 HeatWave 사용 유무에 따라 성능 차이를 확인해 보도록 하겠습니다.

먼저, **SET SESSION use_secondary_engine=OFF;** 명령을 수행하여 HeatWave를 사용하지 않도록 한 후 수행 시간을 확인해 보겠습니다.

```sql
# MySQL Workbench
SET SESSION use_secondary_engine=OFF;

select date_format(f.departure, '%Y-%m') as "Month", 
       count(b.booking_id) as Booking_Count,
       sum(b.price) as Total_Price
from flight f , booking b, passengerdetails p
where b.flight_id = f.flight_id
  and b.passenger_id = p.passenger_id
--  and p.country in ("USA", "FRANCE", "ITALY")
group by date_format(f.departure, '%Y-%m')
order by Booking_Count desc
;
```

수행 결과

```
+---------+---------------+---------------+
| Month   | Booking_Count | Total_Price   |
+---------+---------------+---------------+
| 2015-08 |      18087856 | 4539969453.11 |
| 2015-07 |      18071432 | 4535061010.08 |
| 2015-06 |      17540333 | 4402788741.38 |
| 2015-09 |        604998 |  151771352.61 |
+---------+---------------+---------------+
4 rows in set (6 min 17.6133 sec)
```

수행 시간이 5분 이상 걸릴것으로 예상되기 때문에 다른 창에서 HeatWave를 사용할 때의 수행 시간을 확인해 보도록 하겠습니다.

```sql
# MySQL Shell

SET SESSION use_secondary_engine=ON;  -- 확인용 ( default = ON )

select date_format(f.departure, '%Y-%m') as "Month", 
       count(b.booking_id) as Booking_Count,
       sum(b.price) as Total_Price
from flight f , booking b, passengerdetails p
where b.flight_id = f.flight_id
  and b.passenger_id = p.passenger_id
--  and p.country in ("USA", "FRANCE", "ITALY")
group by date_format(f.departure, '%Y-%m')
order by Booking_Count desc
;
```

수행 결과

```sql
 MySQL  10.1.1.136:3306 ssl  airportdb  SQL > select date_format(f.departure, '%Y-%m') as "Month",
                                           ->        count(b.booking_id) as Booking_Count,
                                           ->        sum(b.price) as Total_Price
                                           -> from flight f , booking b, passengerdetails p
                                           -> where b.flight_id = f.flight_id
                                           ->   and b.passenger_id = p.passenger_id
                                           -> --  and p.country in ("USA", "FRANCE", "ITALY")
                                           -> group by date_format(f.departure, '%Y-%m')
                                           -> order by Booking_Count desc
                                           -> ;
+---------+---------------+---------------+
| Month   | Booking_Count | Total_Price   |
+---------+---------------+---------------+
| 2015-08 |      18087856 | 4539969453.11 |
| 2015-07 |      18071432 | 4535061010.08 |
| 2015-06 |      17540333 | 4402788741.38 |
| 2015-09 |        604998 |  151771352.61 |
+---------+---------------+---------------+
4 rows in set (0.4405 sec)
```

HeatWave를 사용할 경우 전체 5천만건에 대한 분석 쿼리도 1초 이내에 수행되는 것을 확인할 수 있습니다.

사용자는 기존 쿼리에 대해 아무런 수정없이 분석 쿼리 수행 성능을 수십~수백배 향상시킬 수 있습니다.



**EXPLAIN 명령으로 Execution Plan 확인 - HeatWave Cluster 사용 여부 판단**

쿼리 수행 시 Optimizer 판단하여 자동으로 HeatWave에서 쿼리가 수행됩니다.  
쿼리 수행 전 HwatWave에서 수행될 지 여부는 **EXPLAIN** 명령을 사용하여 수행 계획을 보고 확인할 수 있습니다.
수행 계획 상에 **Using secondary engine RAPID** 구문을 통해 HeatWave Cluster를 사용한다는 것을 확인할 수 있습니다.

```sql
EXPLAIN 
select date_format(f.departure, '%Y-%m') as "Month", 
       count(b.booking_id) as Booking_Count,
       sum(b.price) as Total_Price
from flight f , booking b, passengerdetails p
where b.flight_id = f.flight_id
  and b.passenger_id = p.passenger_id
--  and p.country in ("USA", "FRANCE", "ITALY")
group by date_format(f.departure, '%Y-%m')
order by Booking_Count desc\G
```

수행 결과

```sql
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: p
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 33528
     filtered: 100
        Extra: Using temporary; Using filesort; Using secondary engine RAPID
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: b
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 48970929
     filtered: 10
        Extra: Using where; Using join buffer (hash join); Using secondary engine RAPID
*************************** 3. row ***************************
           id: 1
  select_type: SIMPLE
        table: f
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 461286
     filtered: 10
        Extra: Using where; Using join buffer (hash join); Using secondary engine RAPID
3 rows in set, 1 warning (0.0130 sec)
Note (code 1003): /* select#1 */ select date_format(`airportdb`.`f`.`departure`,'%Y-%m') AS `Month`,count(`airportdb`.`b`.`booking_id`) AS `Booking_Count`,sum(`airportdb`.`b`.`price`) AS `Total_Price` from `airportdb`.`flight` `f` join `airportdb`.`booking` `b` join `airportdb`.`passengerdetails` `p` where ((`airportdb`.`b`.`passenger_id` = `airportdb`.`p`.`passenger_id`) and (`airportdb`.`f`.`flight_id` = `airportdb`.`b`.`flight_id`)) group by date_format(`airportdb`.`f`.`departure`,'%Y-%m') order by `Booking_Count` desc
```

**테스트 쿼리 #1**

```sql
SELECT airline.airlinename,
        AVG(datediff(departure,birthdate)/365.25) as avg_age,
        count(*) as nb_people
FROM    booking, flight, airline, passengerdetails
WHERE   booking.flight_id=flight.flight_id 
AND     airline.airline_id=flight.airline_id 
AND     booking.passenger_id=passengerdetails.passenger_id 
AND     country IN ("SWITZERLAND", "FRANCE", "ITALY")
GROUP BY airline.airlinename
ORDER BY airline.airlinename, avg_age
LIMIT 10;
```

수행 결과

```sql
+----------------------+-------------+-----------+
| airlinename          | avg_age     | nb_people |
+----------------------+-------------+-----------+
| Afghanistan Airlines | 45.92607116 |     20570 |
| Albania Airlines     | 46.02480595 |     21804 |
| American Samoa Airli | 46.10093492 |     15249 |
| Angola Airlines      | 45.97209191 |     18539 |
| Argentina Airlines   | 46.03110048 |     21298 |
| Australia Airlines   | 46.07920841 |     19986 |
| Azerbaijan Airlines  | 46.01654847 |     16011 |
| Bahamas Airlines     | 46.22537506 |     22564 |
| Belarus Airlines     | 46.10813567 |     18226 |
| Bhutan Airlines      | 46.13247687 |     22873 |
+----------------------+-------------+-----------+
10 rows in set (0.1798 sec)
```

HeatWave를 사용하지 않을 경우

```sql
SET SESSION use_secondary_engine=OFF;

SELECT airline.airlinename,
        AVG(datediff(departure,birthdate)/365.25) as avg_age,
        count(*) as nb_people
FROM    booking, flight, airline, passengerdetails
WHERE   booking.flight_id=flight.flight_id 
AND     airline.airline_id=flight.airline_id 
AND     booking.passenger_id=passengerdetails.passenger_id 
AND     country IN ("SWITZERLAND", "FRANCE", "ITALY")
GROUP BY airline.airlinename
ORDER BY airline.airlinename, avg_age
LIMIT 10;

+----------------------+-------------+-----------+
| airlinename          | avg_age     | nb_people |
+----------------------+-------------+-----------+
| Afghanistan Airlines | 45.92612137 |     20570 |
| Albania Airlines     | 46.02485633 |     21804 |
| American Samoa Airli | 46.10098466 |     15249 |
| Angola Airlines      | 45.97214161 |     18539 |
| Argentina Airlines   | 46.03115082 |     21298 |
| Australia Airlines   | 46.07925759 |     19986 |
| Azerbaijan Airlines  | 46.01659919 |     16011 |
| Bahamas Airlines     | 46.22542566 |     22564 |
| Belarus Airlines     | 46.10818585 |     18226 |
| Bhutan Airlines      | 46.13252703 |     22873 |
+----------------------+-------------+-----------+
10 rows in set (16.6687 sec)
```

**테스트 쿼리 #2**

```sql
-- HeatWave 에서 수행될 경우

SET SESSION use_secondary_engine=ON;

SELECT
airline.airlinename,
SUM(booking.price) as price_tickets,
count(*) as nb_tickets
FROM
booking, flight, airline, airport_geo
WHERE
booking.flight_id=flight.flight_id AND
airline.airline_id=flight.airline_id AND
flight.from=airport_geo.airport_id AND
airport_geo.country = "UNITED STATES"
GROUP BY
airline.airlinename
ORDER BY
nb_tickets desc, airline.airlinename
LIMIT 10;

+----------------------+---------------+------------+
| airlinename          | price_tickets | nb_tickets |
+----------------------+---------------+------------+
| Falkland Is Airlines |   54329614.83 |     216237 |
| Micronesia Airlines  |   53875825.72 |     214275 |
| Brazil Airlines      |   50039222.97 |     199451 |
| Cyprus Airlines      |   47466718.63 |     189493 |
| Yugoslavia Airlines  |   46905122.48 |     186865 |
| Italy Airlines       |   45731672.06 |     182152 |
| Peru Airlines        |   45742817.89 |     182111 |
| Luxembourg Airlines  |   44537553.63 |     177952 |
| Chad Airlines        |   43162776.69 |     172159 |
| Australia Airlines   |   41201251.28 |     164501 |
+----------------------+---------------+------------+
10 rows in set (0.1528 sec)

-- HeatWave를 사용하지 않을 경우

SET SESSION use_secondary_engine=OFF;

SELECT
airline.airlinename,
SUM(booking.price) as price_tickets,
count(*) as nb_tickets
FROM
booking, flight, airline, airport_geo
WHERE
booking.flight_id=flight.flight_id AND
airline.airline_id=flight.airline_id AND
flight.from=airport_geo.airport_id AND
airport_geo.country = "UNITED STATES"
GROUP BY
airline.airlinename
ORDER BY
nb_tickets desc, airline.airlinename
LIMIT 10;

+----------------------+---------------+------------+
| airlinename          | price_tickets | nb_tickets |
+----------------------+---------------+------------+
| Falkland Is Airlines |   54329614.83 |     216237 |
| Micronesia Airlines  |   53875825.72 |     214275 |
| Brazil Airlines      |   50039222.97 |     199451 |
| Cyprus Airlines      |   47466718.63 |     189493 |
| Yugoslavia Airlines  |   46905122.48 |     186865 |
| Italy Airlines       |   45731672.06 |     182152 |
| Peru Airlines        |   45742817.89 |     182111 |
| Luxembourg Airlines  |   44537553.63 |     177952 |
| Chad Airlines        |   43162776.69 |     172159 |
| Australia Airlines   |   41201251.28 |     164501 |
+----------------------+---------------+------------+
10 rows in set (59.9572 sec)

```

**테스트 쿼리 #3**

```sql
-- HeatWave 에서 수행될 경우

SET SESSION use_secondary_engine=ON;

SELECT 
firstname,
lastname,
COUNT(booking.passenger_id) AS count_bookings
FROM
passenger,
booking
WHERE
booking.passenger_id = passenger.passenger_id
    AND passenger.lastname = 'Aldrin'
    OR (passenger.firstname = 'Neil'
    AND passenger.lastname = 'Armstrong')
    AND booking.price > 400.00
GROUP BY firstname , lastname;

+-----------+-----------+----------------+
| firstname | lastname  | count_bookings |
+-----------+-----------+----------------+
| Buzz      | Aldrin    |           1404 |
| Neil      | Armstrong |       10955025 |
+-----------+-----------+----------------+
2 rows in set (1.9372 sec)

-- HeatWave를 사용하지 않을 경우

SET SESSION use_secondary_engine=OFF;

SELECT 
firstname,
lastname,
COUNT(booking.passenger_id) AS count_bookings
FROM
passenger,
booking
WHERE
booking.passenger_id = passenger.passenger_id
    AND passenger.lastname = 'Aldrin'
    OR (passenger.firstname = 'Neil'
    AND passenger.lastname = 'Armstrong')
    AND booking.price > 400.00
GROUP BY firstname , lastname;

+-----------+-----------+----------------+
| firstname | lastname  | count_bookings |
+-----------+-----------+----------------+
| Neil      | Armstrong |       10955025 |
| Buzz      | Aldrin    |           1404 |
+-----------+-----------+----------------+
2 rows in set (36.4692 sec)

```


## 4. AutoPilot - Auto Data Placement

HeatWave AutoPilot의 주요한 기능 중 하나인 Auto Data Plancement 기능에 대해 살펴보도록 하겠습니다.  
Auto Data Placement 기능은 기존에 수행된 쿼리를 분석하여 최적의 데이터 분산 키와 성능 향상 정도를 제안해 줍니다.

테스트를 위해 다양한 조건을 가지는 추가적인 쿼리를 몇가지 더 수행하겠습니다.

```sql
SET SESSION use_secondary_engine=ON;

-- Find per-company average age of passengers from Switzerland, Italy and France
SELECT 
    airline.airlinename,
    AVG(DATEDIFF(departure, birthdate) / 365.25) AS avg_age,
    COUNT(*) AS nb_people
FROM
    booking,
    flight,
    airline,
    passengerdetails
WHERE
    booking.flight_id = flight.flight_id
        AND airline.airline_id = flight.airline_id
        AND booking.passenger_id = passengerdetails.passenger_id
        AND country IN ('SWITZERLAND' , 'FRANCE', 'ITALY')
GROUP BY airline.airlinename
ORDER BY airline.airlinename , avg_age
LIMIT 10;


-- Find top 10 companies selling the biggest amount of tickets for planes taking off from US airports
SELECT 
    airline.airlinename,
    SUM(booking.price) AS price_tickets,
    COUNT(*) AS nb_tickets
FROM
    booking,
    flight,
    airline,
    airport_geo
WHERE
    booking.flight_id = flight.flight_id
        AND airline.airline_id = flight.airline_id
        AND flight.from = airport_geo.airport_id
        AND airport_geo.country = 'UNITED STATES'
GROUP BY airline.airlinename
ORDER BY nb_tickets DESC , airline.airlinename
LIMIT 10;


-- Query: Ticket price greater than 500, grouped by price
SELECT 
booking.price, COUNT(*)
FROM
booking
WHERE
booking.price > 500
GROUP BY booking.price
ORDER BY booking.price
LIMIT 10;


-- Ticket price greater than 400, grouped by firstname , lastname
SELECT 
    firstname,
    lastname,
    COUNT(booking.passenger_id) AS count_bookings
FROM
    passenger,
    booking
WHERE
    booking.passenger_id = passenger.passenger_id
        AND passenger.lastname = 'Aldrin'
        OR (passenger.firstname = 'Neil'
        AND passenger.lastname = 'Armstrong')
        AND booking.price > 400.00
GROUP BY firstname , lastname;

```


### 4.1. Auto Data Placement advisor 수행

그러면, Auto Data Placement Advisor 기능을 수행해 보도록 하겠습니다.  
이 기능도 Auto Parallel Loading 처럼 간단한 함수 - **SYS.HEATWAVE_ADVISOR** 만 수행하면 됩니다.

```sql
call sys.heatwave_advisor(json_object('target_schema', JSON_ARRAY('airportdb'), 'auto_dp', json_object('benefit_threshold',0) ));
```

수행 결과

```
+-------------------------------+
| INITIALIZING HEATWAVE ADVISOR |
+-------------------------------+
| Version: 1.26                 |
|                               |
| Output Mode: normal           |
| Excluded Queries: 0           |
| Target Schemas: 1             |
|                               |
+-------------------------------+
6 rows in set (0.0085 sec)

+---------------------------------------------------------+
| ANALYZING LOADED DATA                                   |
+---------------------------------------------------------+
| Total 15 tables loaded in HeatWave for 1 schemas        |
| Tables excluded by user: 0 (within target schemas)      |
|                                                         |
| SCHEMA                            TABLES        COLUMNS |
| NAME                              LOADED         LOADED |
| ------                            ------         ------ |
| `airportdb`                           15            119 |
|                                                         |
+---------------------------------------------------------+
8 rows in set (0.0085 sec)

+--------------------------------------------------------------------+
| AUTO DATA PLACEMENT                                                |
+--------------------------------------------------------------------+
| Auto Data Placement Configuration:                                 |
|                                                                    |
|   Minimum benefit threshold: 0%                                    |
|                                                                    |
| Producing Data Placement suggestions for current setup:            |
|                                                                    |
|   Tables Loaded: 15                                                |
|   Queries used: 20                                                 |
|     Total query execution time: 6.21 s                             |
|     Most recent query executed on: Monday 25th April 2022 06:31:21 |
|     Oldest query executed on: Monday 25th April 2022 02:36:54      |
|   HeatWave cluster size: 2 nodes                                   |
|                                                                    |
+--------------------------------------------------------------------+
13 rows in set (0.0085 sec)

+-------------------------------------------------------------------------------------------------+
| DATA PLACEMENT SUGGESTIONS                                                                      |
+-------------------------------------------------------------------------------------------------+
| Total Data Placement suggestions produced for 2 tables                                          |
|                                                                                                 |
| TABLE                                        DATA PLACEMENT                      DATA PLACEMENT |
| NAME                                            CURRENT KEY                       SUGGESTED KEY |
| ------                                       --------------                      -------------- |
| `airportdb`.`booking`                            booking_id                        passenger_id |
| `airportdb`.`flight`                              flight_id                          airline_id |
|                                                                                                 |
| Expected benefit after applying Data Placement suggestions                                      |
|   Runtime saving: 121.77 ms                                                                     |
|   Performance benefit: 0.9%                                                                     |
|                                                                                                 |
+-------------------------------------------------------------------------------------------------+
12 rows in set (0.0085 sec)

+----------------------------------------------------------------------------------------------------------------+
| SCRIPT GENERATION                                                                                              |
+----------------------------------------------------------------------------------------------------------------+
| Script generated for applying suggestions for 2 loaded tables                                                  |
|                                                                                                                |
| Applying changes will take approximately 10.00 s                                                               |
|                                                                                                                |
| Retrieve script containing 12 generated DDL commands using the query below:                                    |
|   SELECT log->>"$.sql" AS "SQL Script" FROM sys.heatwave_advisor_report WHERE type = "sql" ORDER BY id;        |
|                                                                                                                |
| Caution: Executing the generated script will alter the column comment and secondary engine flags in the schema |
|                                                                                                                |
+----------------------------------------------------------------------------------------------------------------+
9 rows in set (0.0085 sec)

Query OK, 0 rows affected (0.0085 sec)
```

위의 결과에서 BOOKING 테이블은 BOOKING_ID 컬럼을 PASSENGER_ID 컬럼으로, 
FLIGHT 테이블은 FLIGHT_ID 컬럼을 AIRLINE_ID 컬럼으로 재분산할 것을 제안한 것을 확인할 수 있습니다.  
이때, 성능 개선 정도는 약 1% 정도로 미미하게 개선될 것으로 예상이 되었네요.  

사용자는 이 결과를 참조하여 해당 테이블에 대해 재분산 작업을 수행할 지 결정할 수 있습니다. 

Auto Data Placement Advisor 결과는 사용자가 수행한 쿼리에 따라 변하므로, 
시스템 구성 후 테스트 단계에서 여러번 수행하여 최적의 Data Placement Key 값을 결정할 수 있습니다.

**스크립트 조회**

```sql
SELECT log->>"$.sql" AS "SQL Script" FROM sys.heatwave_advisor_report WHERE type = "sql" ORDER BY id; 

+---------------------------------------------------------------------------------------------------------------------+
| SQL Script                                                                                                          |
+---------------------------------------------------------------------------------------------------------------------+
| SET SESSION innodb_parallel_read_threads = 32;                                                                      |
| ALTER TABLE `airportdb`.`booking` SECONDARY_UNLOAD;                                                                 |
| ALTER TABLE `airportdb`.`booking` SECONDARY_ENGINE=NULL;                                                            |
| ALTER TABLE `airportdb`.`booking` MODIFY `passenger_id` int NOT NULL COMMENT 'RAPID_COLUMN=DATA_PLACEMENT_KEY=1';   |
| ALTER TABLE `airportdb`.`booking` SECONDARY_ENGINE=RAPID;                                                           |
| ALTER TABLE `airportdb`.`booking` SECONDARY_LOAD;                                                                   |
| SET SESSION innodb_parallel_read_threads = 9;                                                                       |
| ALTER TABLE `airportdb`.`flight` SECONDARY_UNLOAD;                                                                  |
| ALTER TABLE `airportdb`.`flight` SECONDARY_ENGINE=NULL;                                                             |
| ALTER TABLE `airportdb`.`flight` MODIFY `airline_id` smallint NOT NULL COMMENT 'RAPID_COLUMN=DATA_PLACEMENT_KEY=1'; |
| ALTER TABLE `airportdb`.`flight` SECONDARY_ENGINE=RAPID;                                                            |
| ALTER TABLE `airportdb`.`flight` SECONDARY_LOAD;                                                                    |
+---------------------------------------------------------------------------------------------------------------------+
14 rows in set (0.0009 sec)
```


<br><br>

# HeatWave ML 데모 시나리오

## 1. HeatWave ML 테스트

사용자 정보 중 부가적인 정보를 기반으로 AFFINITY_CARD 발급 여부를 예측하는 머신 러닝 모델을 학습합니다.


### 1.1. 필요 데이터 준비

앞의 시나리오에서 로딩한 테이블에 supplementary_demographics 테이블이 포함되어 있습니다.  
만약, 기존 airportdb 에 supplementary_demographics 테이블이 없다면 다음과 같이 해당 데이터를 로딩할 수 있습니다.

```sql
# [MDS-Client]
$ wget -O supplementary_demographics.csv https://bit.ly/3rAxiJJ
$ ls supplementary_demographics.csv

# mysql 접속 

$ mysqlsh --mysql admin@10.1.1.136 --database=airportdb --sql
Password : Oracle#1234

-- 테이블 생성
drop table supplementary_demographics;

create table supplementary_demographics (
CUST_ID                          int NOT NULL ,
EDUCATION                        char(21)    ,
OCCUPATION                       char(21)    ,
HOUSEHOLD_SIZE                   char(21)    ,
YRS_RESIDENCE                    int          ,
AFFINITY_CARD                    int      ,
BULK_PACK_DISKETTES              int  ,
FLAT_PANEL_MONITOR               int  ,
HOME_THEATER_PACKAGE             int  ,
BOOKKEEPING_APPLICATION          int  ,
PRINTER_SUPPLIES                 int  ,
Y_BOX_GAMES                      int  ,
OS_DOC_SET_KANJI                 int  ,
COMMENTS                         VARCHAR(4000)  ,
primary key( cust_id )
);

-- Data Loading 
\js
util.importTable("supplementary_demographics.csv", {schema: "airportdb", table: "supplementary_demographics", dialect: "csv-unix", skipRows: 1, showProgress: true})

\sql

```


### 1.2. Table 정보 조회

```sql
select count(*) from supplementary_demographics ;
```
```
+----------+
| count(*) |
+----------+
|     4500 |
+----------+
1 row in set (0.0034 sec)
```

```sql
select * from supplementary_demographics limit 10;
```
```
+---------+-----------+------------+----------------+---------------+---------------+---------------------+--------------------+----------------------+-------------------------+------------------+-------------+------------------+------------------------------------------------------------------------------------------------------------------------------------------------+
| CUST_ID | EDUCATION | OCCUPATION | HOUSEHOLD_SIZE | YRS_RESIDENCE | AFFINITY_CARD | BULK_PACK_DISKETTES | FLAT_PANEL_MONITOR | HOME_THEATER_PACKAGE | BOOKKEEPING_APPLICATION | PRINTER_SUPPLIES | Y_BOX_GAMES | OS_DOC_SET_KANJI | COMMENTS                                                                                                                                       |
+---------+-----------+------------+----------------+---------------+---------------+---------------------+--------------------+----------------------+-------------------------+------------------+-------------+------------------+------------------------------------------------------------------------------------------------------------------------------------------------+
|  100001 | < Bach.   | Exec.      | 2              |             3 |             0 |                   0 |                  0 |                    1 |                       1 |                1 |           0 |                0 | Thanks a lot for my new affinity card. I love the discounts and have since started shopping at your store for everything.                      |
|  100002 | Bach.     | Prof.      | 2              |             4 |             0 |                   1 |                  1 |                    1 |                       1 |                1 |           0 |                0 | The more times that I shop at your store, the more times I am impressed.  Don't change anything                                                |
|  100003 | < Bach.   | Sales      | 2              |             6 |             0 |                   1 |                  1 |                    0 |                       1 |                1 |           0 |                0 | It is a good way to attract new shoppers. After shopping at your store for more than a month, I am ready to move on though. Not enough variety |
|  100004 | < Bach.   | Sales      | 2              |             5 |             0 |                   1 |                  1 |                    1 |                       1 |                1 |           0 |                0 | Thanks but even with your discounts, your products are too expensive. Sorry.                                                                   |
|  100005 | Assoc-A   | Crafts     | 3              |             5 |             1 |                   0 |                  0 |                    1 |                       1 |                1 |           0 |                0 | Affinity card is a great idea. But your store is still too expensive. I am tired of your lousy junk mail.                                      |
|  100006 | < Bach.   | Prof.      | 9+             |             2 |             0 |                   0 |                  0 |                    0 |                       1 |                1 |           1 |                0 | I am not going to waste my time filling up this three page form. Lousy idea.                                                                   |
|  100007 | HS-grad   | Other      | 2              |             5 |             0 |                   1 |                  1 |                    1 |                       1 |                1 |           0 |                0 | Thanks but even with your discounts, your products are too expensive. Sorry.                                                                   |
|  100008 | < Bach.   | Crafts     | 2              |             5 |             0 |                   1 |                  1 |                    1 |                       1 |                1 |           0 |                0 | I shop your store a lot.  I love your weekly specials.                                                                                         |
|  100009 | Bach.     | Prof.      | 3              |             3 |             1 |                   0 |                  0 |                    0 |                       1 |                1 |           1 |                0 | Thank you! But I am very unhappy with all the junk mail you keep sending.                                                                      |
|  100010 | HS-grad   | Crafts     | 3              |             3 |             0 |                   1 |                  1 |                    0 |                       1 |                1 |           1 |                0 | I am unhappy with the service at your store. Do not consider me a loyal customer just because I use your Affinity Card                         |
+---------+-----------+------------+----------------+---------------+---------------+---------------------+--------------------+----------------------+-------------------------+------------------+-------------+------------------+------------------------------------------------------------------------------------------------------------------------------------------------+
10 rows in set (0.0006 sec)
```

```sql
select AFFINITY_CARD, count(*) from supplementary_demographics group by AFFINITY_CARD;
```
```
+---------------+----------+
| AFFINITY_CARD | count(*) |
+---------------+----------+
|             0 |     3428 |
|             1 |     1072 |
+---------------+----------+
2 rows in set (0.0023 sec)
```

```sql
select household_size,  affinity_card, count(*) 
from supplementary_demographics 
group by household_size, affinity_card
order by household_size, affinity_card ;
```
```
+----------------+---------------+----------+
| household_size | affinity_card | count(*) |
+----------------+---------------+----------+
| 1              |             0 |      681 |
| 1              |             1 |       11 |
| 2              |             0 |     1040 |
| 2              |             1 |      109 |
| 3              |             0 |      973 |
| 3              |             1 |      814 |
| 4-5            |             0 |      112 |
| 4-5            |             1 |      107 |
| 6-8            |             0 |      146 |
| 6-8            |             1 |        2 |
| 9+             |             0 |      476 |
| 9+             |             1 |       29 |
+----------------+---------------+----------+
12 rows in set (0.0031 sec)

```