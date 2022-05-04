# Lab 5 : AutoPilot 

## Task 1 : Autopilot - Auto Data Placement

HeatWave AutoPilot의 주요한 기능 중 하나인 Auto Data Plancement 기능에 대해 살펴보도록 하겠습니다.  
Auto Data Placement 기능은 기존에 수행된 쿼리를 분석하여 최적의 데이터 분산 키와 성능 향상 정도를 제안해 줍니다.

테스트를 위해 다양한 조건을 가지는 쿼리를 몇가지 더 추가로 수행하겠습니다.

1. 추가 쿼리 수행

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


2. Auto Data Placement advisor 수행

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

3. 스크립트 조회 - HeatWave 재로딩 스크립트

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

