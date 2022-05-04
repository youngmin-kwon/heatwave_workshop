# Lab 6 :  HeatWave ML 

사용자 정보 중 부가적인 정보를 기반으로 AFFINITY_CARD 발급 여부를 예측하는 머신 러닝 모델을 학습하고
예측 및 예측 결과에 대한 설명 정보를 확인하는 테스트를 진행합니다.


## Task 1 : 데이터 로딩

앞의 시나리오에서 로딩한 테이블에 `supplementary_demographics` 테이블이 포함되어 있습니다.  
만약, 기존 airportdb 에 `supplementary_demographics` 테이블이 없다면 다음과 같이 해당 데이터를 로딩할 수 있습니다.

1. 데이터 다운로드 및 로딩

    ```sql
    # [MDS-Client Compute Instance]
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


2. 로딩된 Table 정보 조회

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

## Task 2 : 머신 러닝 테스트를 위한 DataSet 구성 - Train Data & Test Data

1. 머신 러닝 모델 학습 및 모델 성능 확인을 위해 Train Data와 Test Data를 구성합니다.

    ```sql
    -- Train Data - train_tab - 3000 rows
    drop table train_tab;
    create table train_tab as select * from supplementary_demographics limit 3000;

    -- Test Data - test_tab - 1500 rows
    drop table test_tab;
    create table test_tab  as 
    select a.* 
    from supplementary_demographics a
    where a.cust_id not in ( select cust_id from train_tab )

    -- Unlabeled Table 생성
    drop table test_tab2;
    create table test_tab2 as select * from test_tab limit 20;
    alter table test_tab2 drop affinity_card;
    ;
    ```


## Task 3 : 머신 러닝 모델 학습 - Training Models

HeatWave ML 은 Oracle AutoML 을 활용하여 머신 러닝 모듈을 제공합니다.  
따라서, 복잡한 절차없이 **sys.ML_TRAIN** 함수만 수행하면 머신 러닝 모델을 아주 쉽게 학습할 수 있습니다.

파라미터로 학습을 원하는 테이블, Target 컬럼 및 Classification / Regression 여부만 지정해 주면 됩니다.

```sql
call sys.ML_TRAIN( 'airportdb.train_tab', 'affinity_card', JSON_OBJECT('task', 'classification'), @affinity_model );
```
```sql
 MySQL  10.1.1.136:3306 ssl  airportdb  SQL > call sys.ML_TRAIN( 'airportdb.train_tab', 'affinity_card', JSON_OBJECT('task', 'classification'), @affinity_model );
Query OK, 0 rows affected (27.3090 sec)
```

머신 러닝 모델 학습에 약 27초 정도가 소요되었습니다.

<!--
# 생성된 모델 정보 확인
SELECT model_id, model_handle, model_owner FROM ML_SCHEMA_ADWML.MODEL_CATALOG;
-->

## Task 4 : 모델 스코어 확인

위에서 생성한 머신 러닝 모델의 정확도를 확인해 보겠습니다.  
Scoring 방법은 **sys.ML_SCORE** 함수를 호출하여 쉽게 확인할 수 있습니다. 

```sql
-- Load Model

call sys.ML_MODEL_LOAD( @affinity_model, NULL );
Query OK, 0 rows affected (0.7527 sec)

call sys.ML_SCORE( 'airportdb.test_tab', 'affinity_card', @affinity_model, 'balanced_accuracy', @score );
Query OK, 0 rows affected (5.8144 sec)

-- Score 확인 
select @score;
+--------------------+
| @score             |
+--------------------+
| 0.8425005674362183 |
+--------------------+
1 row in set (0.0004 sec)
```

테스트 데이터에 대한 모델의 정확도는 약 84% 인것을 확인할 수 있습니다.

## Task 5 : Row Prediction 및 Explanation

학습된 모델을 사용하여 새로운 데이터에 대해 예측값을 확인해 보겠습니다.  

1. Row Predictions 

    특정 Row 에 대한 머신 러닝 예측을 통해 
    신규 고객이 등록되었을 때 해당 고객에 대한 예측값을 확인할 수 있습니다.  
    Row Prediction 은 **sys.ML_PREDICT_ROW** 함수를 호출하여 수행할 수 있습니다.

    ```sql
    -- Load Machine Learning Model
    call sys.ML_MODEL_LOAD( @affinity_model, NULL );
    Query OK, 0 rows affected (0.8184 sec)


    --테스트 용 데이터

    SET @row_input = JSON_OBJECT(
    "CUST_ID",                 103001,
    "EDUCATION",               "HS-grad",
    "OCCUPATION",              "Sales",
    "HOUSEHOLD_SIZE",          "1",
    "YRS_RESIDENCE",           2,
    "BULK_PACK_DISKETTES",     1,
    "FLAT_PANEL_MONITOR",      1,
    "HOME_THEATER_PACKAGE",    0,
    "BOOKKEEPING_APPLICATION", 1,
    "PRINTER_SUPPLIES",        1,
    "Y_BOX_GAMES",             1,
    "OS_DOC_SET_KANJI",        0,
    "COMMENTS",                "Shopping at your store is a hassle. I rarely shop there and usually forget to bring your new loyalty card and hence never get the items at the sale price.  Can a store manager look up my account on-line?" );

    -- Row Prediction
    SELECT sys.ML_PREDICT_ROW(@row_input, @affinity_model);

    +----------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | sys.ML_PREDICT_ROW(@row_input @affinity_model) 
    +-----------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | {"CUST_ID": 103001, "COMMENTS": "Shopping at your store is a hassle. I rarely shop there and usually forget to bring your new loyalty card and hence never get the items at the sale price.  Can a store manager look up my account on-line?", "EDUCATION": "HS-grad", "OCCUPATION": "Sales", "Prediction": 0, "Y_BOX_GAMES": 1, "YRS_RESIDENCE": 2, "HOUSEHOLD_SIZE": "1", "OS_DOC_SET_KANJI": 0, "PRINTER_SUPPLIES": 1, "FLAT_PANEL_MONITOR": 1, "BULK_PACK_DISKETTES": 1, "HOME_THEATER_PACKAGE": 0, "BOOKKEEPING_APPLICATION": 1}
    +-----------------------------------------------------------------------------------------------------------------------------------------------------------------+
    1 row in set (2.1483 sec)
    ```

    Prediction 결과 ("Prediction" 컬럼) 0 으로 나오는 것을 확인할 수 있습니다.  

2. Row Prediction 결과 Explanation

    **ML_EXPLAIN_ROW** 함수를 사용하여 예측에 대한 설명을 확인할 수 있습니다.

    설명(Explanation)은 예측에 가장 큰 영향을 미치는 features를 이해하는 데 도움이 됩니다.  
    각 feature 별 중요도는 -1에서 1 사이의 값으로 표시됩니다.
    양수 값은 기능이 예측에 기여도가 높은 features 임을 표시하고, 0에 근접한 값은 예측에 영향이 거의 없는 features를 의미합니다.

    ```sql
    SELECT sys.ML_EXPLAIN_ROW(@row_input, @affinity_model);

    +-------------------------------------------------------------------------------------------------------------------------------------------------+
    | sys.ML_EXPLAIN_ROW(@row_input, @affinity_model)   +-------------------------------------------------------------------------------------------------------------------------------------------------+
    | {"CUST_ID": 103001, "COMMENTS": "Shopping at your store is a hassle. I rarely shop there and usually forget to bring your new loyalty card and hence never get the items at the sale price.  Can a store manager look up my account on-line?", "EDUCATION": "HS-grad", "OCCUPATION": "Sales", "Prediction": 0, "Y_BOX_GAMES": 1, "YRS_RESIDENCE": 2, "HOUSEHOLD_SIZE": "1", "OS_DOC_SET_KANJI": 0, "PRINTER_SUPPLIES": 1, "FLAT_PANEL_MONITOR": 1, "BULK_PACK_DISKETTES": 1, "COMMENTS_attribution": -0.0021, "HOME_THEATER_PACKAGE": 0, "EDUCATION_attribution": 0.0002, "BOOKKEEPING_APPLICATION": 1, "Y_BOX_GAMES_attribution": 0.0038, "YRS_RESIDENCE_attribution": 0.0256, "HOUSEHOLD_SIZE_attribution": 0.0148} |
    +-------------------------------------------------------------------------------------------------------------------------------------------------+
    1 row in set (2.8583 sec)
    ```

    위의 결과에서 각 feature 별 **_attribution** 값에서 각 컬럼(feature)별 영향도 정보를 확인할 수 있습니다.

## Task 6 : Table Predictions 및 Explanation

1. Table Prediction
   
    **ML_PREDICT_TABLE** 함수는 unlabeled data를 가지는 테이블 데이터 전체에 대해 Prediction을 수행하여 예측 결과를 output 테이블에 저장합니다.

    ```sql
    CALL sys.ML_MODEL_LOAD(@affinity_model, NULL);
    Query OK, 0 rows affected (0.7535 sec)

    -- unlabeled data가 있는 TEST_TAB2 테이블에 대해 Prediction 수행
    CALL sys.ML_PREDICT_TABLE('airportdb.test_tab2', @affinity_model, 'airportdb.affinity_predictions');

    Query OK, 0 rows affected (5.9274 sec)

    --> 만약 airportdb.affinity_predictions 테이블이 이미 존재한다면 drop 후 재수행합니다.

    -- Prediction 결과 확인
    select cust_id, education, occupation, household_size, prediction from airportdb.affinity_predictions;
    +---------+-----------+------------+----------------+------------+
    | cust_id | education | occupation | household_size | prediction |
    +---------+-----------+------------+----------------+------------+
    |  103065 | Bach.     | Cleric.    | 2              |          0 |
    |  103066 | < Bach.   | Sales      | 3              |          1 |
    |  103067 | HS-grad   | Cleric.    | 9+             |          0 |
    |  103068 | Assoc-V   | Machine    | 3              |          0 |
    |  103069 | < Bach.   | TechSup    | 3              |          0 |
    |  103070 | < Bach.   | Crafts     | 2              |          0 |
    |  103071 | Bach.     | Exec.      | 3              |          1 |
    |  103072 | Bach.     | Exec.      | 3              |          1 |
    |  103073 | 11th      | Exec.      | 3              |          0 |
    |  103074 | < Bach.   | ?          | 1              |          0 |
    +---------+-----------+------------+----------------+------------+
    ```

2. Table Prediction 에 대한 Explanation 정보 확인

    테이블에 대해서도 예측에 대한 설명 정보를 확인할 수 있습니다.
    **ML_EXPLAIN_TABLE** 함수를 사용하여 예측에 대한 설명을 확인할 수 있습니다.  설명에 대한 output은 지정한 테이블에 저장됩니다.

    ```sql
    call sys.ML_MODEL_LOAD( @affinity_model, NULL );

    -- Table Explanation 수행
    CALL sys.ML_EXPLAIN_TABLE('airportdb.test_tab2', @affinity_model, 'airportdb.affinity_explanations');

    Query OK, 0 rows affected (12.9186 sec)

    -- Table Explanation 정보 확인 
    desc airportdb.affinity_explanations;

    select cust_id, EDUCATION_attribution, HOUSEHOLD_SIZE_attribution, YRS_RESIDENCE_attribution, Y_BOX_GAMES_attribution, prediction
    from airportdb.affinity_explanations limit 10;

    +---------+-----------------------+----------------------------+---------------------------+-------------------------+------------+
    | cust_id | EDUCATION_attribution | HOUSEHOLD_SIZE_attribution | YRS_RESIDENCE_attribution | Y_BOX_GAMES_attribution | prediction |
    +---------+-----------------------+----------------------------+---------------------------+-------------------------+------------+
    |  103001 |                0.0002 |                     0.0148 |                    0.0256 |                  0.0038 | 0          |
    |  103002 |                0.0005 |                     0.0098 |                    0.0017 |                       0 | 0          |
    |  103003 |                0.0054 |                    -0.0009 |                    0.0015 |                  0.0082 | 0          |
    |  103004 |                0.4043 |                     0.0017 |                    0.0023 |                  0.0005 | 0          |
    |  103005 |               -0.0311 |                     -0.149 |                    0.2948 |                 -0.0473 | 0          |
    |  103006 |                0.0021 |                     0.0009 |                    0.0002 |                   0.003 | 0          |
    |  103007 |                0.3541 |                      0.323 |                    0.0527 |                  0.0141 | 1          |
    |  103008 |                0.1219 |                     0.2349 |                    0.4697 |                  0.0601 | 1          |
    |  103009 |                0.0005 |                     0.0019 |                    0.0039 |                  0.0016 | 0          |
    |  103010 |                0.5583 |                     0.0349 |                   -0.0119 |                 -0.0004 | 0          |
    +---------+-----------------------+----------------------------+---------------------------+-------------------------+------------+
    10 rows in set (0.0008 sec)
    ```

