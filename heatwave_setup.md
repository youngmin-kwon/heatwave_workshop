# MySQL HeatWave 데모 환경 구성

### 1.1. VCN 생성

1. **Navigation Menu** -> **Networking** -> **Virtual Cloud Networks**

2. **Start VCN Wizard** -> **Create VCN with Internet Connectivity** -> Click **Start VCN Wizard**

3. VCN 생성 화면에서 다음 정보 입력:
    - VCN Name : **MDS-VCN**
    - Compartment : **Compartment 선택 ( ex. HW_DEMO or User Compartment)**

4. `Next` -> Review -> `Create`

5. 생성한 MDS-VCN 정보 페이지 -> **Private Subnet-MDS-VCN** 클릭

6. **Security List for Private Subnet-MDS-VCN** 클릭

7. **Add Ingress Rules** 클릭

8. Add Ingress Rules 페이지에서 다음 정보 입력
    - Source CIDR : **0.0.0.0/0**
    - Destination Port Range : **3306,33060**
    - Description : **MySQL Port Access**

9. `Add Ingress Rules` 클릭

### 1.2. Create MySQL Database for HeatWave (DB System) instance

1. Navigation Menu -> Databases -> MySQL DB Systems
  <img src=./images/04mysql01.png>

2. **Create MySQL DB System** 클릭
  <img src=./images/04mysql02.png>

3. MySQL DB System 생성 dialog 에 다음의 정보 입력
    - Compartment 선택
    - Name : **MDS-HW**
    - Description : MySQL Database Service HeatWave instance
    - **HeatWave** 선택
    - Username : admin
    - Password : Oracle#1234
    - Configure Networking:
        + Virtual Cloud Network : **MDS-VCN**
        + Subnet: **Private Subnet-MDS-VCN (Regional)**
    - Configure Placement
        + Availability Domain : 디폴트 선택
        + `Choose a Fault Domain` 체크 박스 선택 안함
    - Configure Hardware
        + Shape: **MySQL.HeatWave.VM.Standard.E3**
        + Data Storage Size(GB) : **1024**
    - Configure Backup : `Enable Automatic Backup` 선택 안함
    - Review and **Create** 클릭
  
    <img src=./images/04mysql09-3.png>

4. 생성 완료 MySQL DB Systems 정보 페이지의 **Endpoint** 정보에서 **Private IP Address** 정보 확인
    - Private IP Address = **10.1.1.136**

    <img src=./images/04mysql10.png>


### 1.3. Add a HeatWave Cluster to MDS-HW MySQL Database System

1. Navigation Menu -> Databases -> MySQL DB Systems

2. **MDS-HW** 선택 

3. MDS-HW Detail Page -> More Action -> **Add HeatWave Cluster**
  <img src=./images/10addheat02.png>

4. `Add HeatWave Cluster` 페이지 
    - Shape : **MySQL.HeatWave.VM.Standard.E3**
    - Node Count : **2**
    - `Add HeatWave Cluster` 클릭

    <img src=./images/10addheat06.png>

5. MySQL Database System 정보 페이지에서 HeatWave Nodes Active 상태 확인
  <img src=./images/10addheat07.png>

만약, MySQL Database 에 기존에 데이터가 있는 상황이면, **Estimate Node Count** 를 클릭하여 **Auto Provisioning** 기능을 활용하여 
노드 수를 예측할 수 있습니다.

6. **Estimate Node Count** 클릭
  <img src=./images/10addheat03.png>

7. **Generate Estimate** 버튼을 클릭하여 HeatWave 메모리 사용량 예측 및 HeatWave Cluster 노드 수 예측
  <img src=./images/10addheat04.png>

8. HeatWave 에 로딩할 테이블들을 선택한 후 **Apply Node Count Estimate** 클릭
  <img src=./images/10addheat05.png>


### 1.4. MySQL Client(Bastion Host) 용으로 사용할 Compute Instance 생성

1. Navigation Menu -> Compute Instances

2. `Create Instance` 클릭

3. Create Instance 페이지
    - Name : **MDS-Client**
    - Compartment 선택
    - Shape 변경 : **VM.Standard.E2.2**
    - Networking : 
        + **MDS-VCN**
        + **Public Subnet-MDS-VCN
        + `Assign a public IP address` : **Yes**
    - Add SSH Keys: **Generate a key pair for me** 선택
        + **Save Private Key** 클릭 후 Private Key 파일( 파일명: ssh-key-<date>.key ) 저장
        + Public Key 파일도 저장 (추후 key pair 재사용을 위해 저장)
    - `Create` 클릭하여 생성

4. 생성된 Compute Instance 정보 페이지에서 **Public IP Address** 정보 확인 및 저장
    - Public IP Address = **129.154.222.226**


### 1.5. Compute Instance 환경 설정 및 MySQL Database Service 접속

Compute Instance 접속을 위한 PC Client 는 putty, PowerShell (Windows), terminal 등 사용 가능합니다.  
본 문서에서는 Cloud Shell 을 사용하여 Compute Intance 접속하여 환경 설정을 수행하였습니다. 

1. **Cloud Shell** 수행 - 브라우저 우측 상단 Region 옆에 있는 터미널아이콘 클릭
2. 앞에서 저장한 Private Key 파일 복사
    - Private Key 파일( `ssh-key-<date>.key` )을 Cloud Shell 화면으로 Drop & Drop
3. [Cloud Shell] 에서 다음 수행
    ```
    #[Cloud Shell]
    $ cp ssh-key-2022-04-22.key .ssh
    $ chmod 600 .ssh/ssh-key-2022-04-22.key
    ```

4. MDS-Client Compute Instance 접속
    ```
    #[Cloud Shell]
    $ ssh -i .ssh/ssh-key-<date>.key opc@<your_compute_instance_public_ip>
    $ ssh -i .ssh/ssh-key-2022-04-22.key opc@129.154.222.226
    ```

5. MySQL Shell Client 설치
    ```
    #[opc@mds-client ~]
    sudo yum install mysql-shell -y
    ```
6. MySQL Database Service 접속  
    MySQL Database Service 상세 정보 페이지에서 Endpoint Private IP Address 정보 확인 (**10.1.1.136**)

    ```
    mysqlsh -uadmin -p -h 10.1.1.136 --sql

    # MySQL 접속 후
    show databases;
    \exit
    ```
    ```
    mysqlsh -uadmin -p -h 10.1.1.136 --sql
    Please provide the password for 'admin@10.1.1.136' : Oracle#1234
    MySQL Shell 8.0.28

    Copyright (c) 2016, 2022, Oracle and/or its affiliates.
    Oracle is a registered trademark of Oracle Corporation and/or its affiliates.
    Other names may be trademarks of their respective owners.

    Type '\help' or '\?' for help; '\quit' to exit.
    Creating a session to 'ADWML@10.1.1.136'
    Fetching schema names for autocompletion... Press ^C to stop.
    Your MySQL connection id is 1907 (X protocol)
    Server version: 8.0.28-u2-cloud MySQL Enterprise - Cloud
    No default schema selected; type \use <schema> to set one.
    MySQL 10.1.1.136:33060+ ssl SQL > show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | sys                |
    +--------------------+
    7 rows in set (0.0007 sec)

    MySQL 10.1.1.136:33060+ ssl SQL > \exit

    ```

### 1.6. MySQL 로 데이터 로딩

테스트용 데이터를 MySQL Database로 로딩합니다.

1. 테스트 용 데이터 파일을 다운로드합니다.

    ```
    # [opc@mds-client ~]$
    wget -O airportdb.zip https://bit.ly/3K4B5W5
    unzip airportdb.zip
    ls airportdb
    ```

2. MySQL Database Service 에 접속하여 데이터 파일을 로딩합니다.
    ```
    $ mysqlsh --mysql admin@10.1.1.136 --js
    Password : Oracle#1234

    MySQL> util.loadDump("/home/opc/airportdb", {dryRun: false, threads: 8, resetProgress:true, ignoreVersion:true})
    ```

3. 로딩 된 테이블 정보 확인
    ```
    -- sql mode 로 변경
    MySQL> \sql

    SELECT table_name, table_rows 
    FROM INFORMATION_SCHEMA.TABLES 
    WHERE TABLE_SCHEMA = 'airportdb';
    ```

    ```
    MySQL  10.1.1.136:3306 ssl  JS > \sql
    Switching to SQL mode... Commands end with ;
    MySQL  10.1.1.136:3306 ssl  SQL > SELECT table_name, table_rows
                                    -> FROM INFORMATION_SCHEMA.TABLES
                                    -> WHERE TABLE_SCHEMA = 'airportdb';
    +----------------------------+------------+
    | TABLE_NAME                 | TABLE_ROWS |
    +----------------------------+------------+
    | airline                    |        113 |
    | airplane                   |       5583 |
    | airplane_type              |        315 |
    | airport                    |       9860 |
    | airport_geo                |       9993 |
    | airport_reachable          |          0 |
    | booking                    |   52061349 |
    | employee                   |       1000 |
    | flight                     |     461286 |
    | flight_log                 |          0 |
    | flightschedule             |       9952 |
    | passenger                  |      36208 |
    | passengerdetails           |      35137 |
    | supplementary_demographics |       4448 |
    | weatherdata                |    4606585 |
    +----------------------------+------------+
    ```

### 1.7. MySQL Workbench 설치 및 접속

https://www.mysql.com/products/workbench/ 페이지에서 MySQL Workbench dowonload 및 설치합니다.

설치 후 MySQL Workbench 수행 시 다음의 정보를 사용하여 MySQL DB를 접속합니다.
접속 방법은 **Standard TCP/IP over SSH** 방식을 통해 Bastion Compute 노드를 경유하여 MySQL DB 에 접속합니다.

- 입력 정보:
    - Connection Name : MySQL HetWave 
    - Connection Method : **Standard TCP/IP over SSH**
    - Parameters:
        + SSH Hostname : *Compute Instance Public IP* - 129.154.222.226
        + SSH Username : **opc**
        + SSH Key File : *SSH private key file*
        + MySQL Hostname : *MySQL Database Service Private IP* - 10.1.1.136
        + MySQL Username : admin
        + Default schema : airportdb
        + Test Connection 클릭 후 접속 확인

    <img src=./images/workbench_connection.png>

- Query Timeout 방지를 위한 Preference 설정
    + MySQL Workbench 메뉴 -> Edit -> Preferences..
    + SQL Editor 클릭
    + MySQL Session > DBMS Connection read timeout interval : 30 -> 600 으로 변경

