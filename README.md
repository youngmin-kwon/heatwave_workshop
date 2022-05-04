# MySQL HeatWave Workshop

## 워크숍 정보

Oracle MySQL Database Service는 개발자가 세계에서 가장 인기 있는 오픈 소스 데이터베이스를 사용하여 안전한 클라우드 네이티브 애플리케이션을 신속하게 개발 및 배포할 수 있도록 하는 완전 관리형 데이터베이스 서비스입니다. MySQL 데이터베이스 서비스는 고성능 인-메모리 쿼리 가속기인 HeatWave가 통합된 유일한 MySQL 클라우드 서비스입니다. MySQL HeatWave를 사용하면 고객이 운영 중인 MySQL 데이터베이스에 대해 직접 정교한 분석을 실행할 수 있으므로 복잡하고 시간이 많이 걸리며 값비싼 데이터 이동 및 별도의 분석 데이터베이스와의 통합이 필요하지 않습니다. HeatWave는 분석 및 혼합 워크로드에 대해 MySQL 성능을 몇 배나 가속화합니다. Oracle Cloud Infrastructure에 최적화된 MySQL Enterprise Edition에서 실행되는 유일한 데이터베이스 서비스입니다. Oracle Cloud Infrastructure 및 MySQL 엔지니어링 팀에서 100% 구축, 관리 및 지원합니다.

Oracle MySQL HeatWave는 고성능 인메모리 쿼리 가속기인 HeatWave가 내장된 유일한 MySQL 클라우드 서비스입니다. 기존 애플리케이션을 변경하지 않고도 분석 및 혼합 워크로드에 대해 MySQL 성능을 수십 배 향상시킬 수 있습니다.  MySQL HeatWave는 트랜잭션 및 분석 워크로드를 위한 단일 통합 플랫폼을 제공합니다. 따라서 복잡하고 시간이 많이 걸리며 값비싼 ETL과 별도의 분석 데이터베이스와의 통합이 필요하지 않습니다. HeatWave의 MySQL Autopilot은 프로비저닝, 데이터 로드, 쿼리 실행 및 오류 처리를 자동화하여 개발자와 DBA의 시간을 상당히 절약합니다. 

워크샵이 끝나면 MySQL 데이터베이스 서비스에 로드된 샘플 데이터에 대해 몇 가지 쿼리를 실행하고 HeatWave가 있는 경우와 없는 경우의 실행 시간을 비교할 수 있습니다.  또한, HeatWave AutoPilot 의 다양한 자동화 기능을 확인할 수 있습니다.  [MySQL HeatWave에 대한 자세한 정보는 이곳에서 확인할 수 있습니다.!](https://www.oracle.com/mysql/heatwave/)

예상 워크샵 시간: 100분

## Objectives

이 워크숍을 통해 다음을 배울 수 있습니다:
- HeatWave를 구성하기 위한 MySQL Database Service를 배포하는 방법을 알아봅니다.
- 다양한 MySQL 클라이언트를 활용하여 MySQL Database 에 접속하는 방법을 알아봅니다.
- HeatWave 클러스터를 MySQL 데이터베이스 서비스에 활성화하는 방법을 배웁니다.
- HeatWave에 테이블이 로드되는 방식을 이해합니다.
- HeatWave를 활용하거나 활용하지 않고 MySQL 데이터베이스 서비스에서 쿼리를 실행하는 방법을 알아보고 극적인 성능 차이를 확인할 수 있습니다.
- HeatWave AutoPilot의 다양한 자동화 기능에 대해 알아봅니다.

## Lab 구성도

![](images/heatwave-bastion-architecture-compute.png)

## Workshop 항목

- [Lab 1 : MySQL Database Service 생성](mds_setup.md) 

- [Lab 2 : MySQL Database Service 접속 클라이언트 구성 및 샘플 데이터 로드](mds_connect_load.md)

- [Lab 3 : HeatWave Cluster 생성](heatwave_setup.md)

- [Lab 4 : HeatWave로 데이터 로드 및 HeatWave 쿼리 성능 테스트](heatwave_query_performance.md)

- [Lab 5 : MySQL AutoPilot 기능 테스트](heatwave_autopilot.md)

- [Lab 6 : HeatWave ML 테스트](heatwave_ml.md)

