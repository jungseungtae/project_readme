# Airflow 일별 데이터 파이프라인 자동화

Apache Airflow 2.10.4 기반으로 Oracle DB 마이그레이션, S3 데이터 적재,
매출 예측 모델 실행을 매일 자동으로 오케스트레이션하는 시스템입니다.

---

## 시스템 아키텍처

```
Oracle 운영 DB
      │
      ▼ DockerOperator (oracle-to-mysql:latest)
   MySQL DB
      │
      ▼ DockerOperator (aws-s3-warehouse:latest)
AWS S3 / Athena
      │
      ▼ DockerOperator (aws-s3-warehouse:latest)
예측 모델 실행 (sales_feature_upload → pred_runner)
```

---

## 프로젝트 구조

```
airflow-pipeline/
│
├── dags/
│   └── daily_pipeline_dag.py     # 메인 파이프라인 (DockerOperator, KST 08:40 실행)
│
├── docker/
│   ├── aws-s3-warehouse/
│   │   ├── Dockerfile
│   │   └── requirements.txt      # boto3, pandas, sklearn 등
│   └── oracle-to-mysql/
│       ├── Dockerfile            # Oracle Instant Client 11.2 포함
│       └── requirements.txt      # oracledb<2.0.0, pymysql 등
│
├── start_airflow.sh              # Airflow 시작 스크립트 (Windows 작업 스케줄러 연동)
├── requirements.txt              # Airflow 환경 의존성
├── setup_aws_credentials.py      # AWS 자격증명 설정 스크립트
└── .gitignore
```

---

## DAG 구성

### `daily_pipeline` — 매일 KST 08:40 실행 (cron: `40 23 * * *` UTC)

Oracle → MySQL 마이그레이션, S3 적재, 예측 실행을 순차 처리합니다.

```
migrate_order_info
  └─▶ migrate_order_detail
        └─▶ run_daily_batch
              └─▶ daily_order_summary_insert
                    └─▶ daily_detail_summary_insert
                          └─▶ migrate_kpi_group
                                └─▶ migrate_storekpi
                                      └─▶ sales_feature_upload
                                            └─▶ pred_runner
```

| 태스크 | 이미지 | 설명 |
|--------|--------|------|
| migrate_order_info | oracle-to-mysql:latest | 주문 정보 Oracle → MySQL |
| migrate_order_detail | oracle-to-mysql:latest | 주문 상세 Oracle → MySQL |
| run_daily_batch | aws-s3-warehouse:latest | 일별 S3 배치 적재 |
| daily_order_summary_insert | aws-s3-warehouse:latest | 주문 요약 집계 적재 |
| daily_detail_summary_insert | aws-s3-warehouse:latest | 상세 요약 집계 적재 |
| migrate_kpi_group | oracle-to-mysql:latest | KPI 그룹 마이그레이션 |
| migrate_storekpi | oracle-to-mysql:latest | 매장 KPI 마이그레이션 |
| sales_feature_upload | aws-s3-warehouse:latest | 예측 피처 데이터 S3 적재 |
| pred_runner | aws-s3-warehouse:latest | 매출·주문·재고 예측 실행 |

---

## Docker 이미지

### `oracle-to-mysql:latest`

Oracle DB에서 데이터를 추출하여 MySQL로 적재합니다.  
Oracle Instant Client 11.2가 이미지에 내장되어 있습니다.

```
python migrate.py --table <table_name>
python migrate.py --group kpi
```

**주요 패키지**: oracledb(<2.0.0), pymysql, pandas

### `aws-s3-warehouse:latest`

S3 데이터 적재 및 예측 모델 실행을 담당합니다.

**주요 패키지**: boto3, s3fs, pandas, pyarrow, scikit-learn, pmdarima, statsmodels

---

## 환경 설정

### Airflow 설치

```bash
python -m venv ~/airflow-env
source ~/airflow-env/bin/activate
pip install -r requirements.txt
airflow db migrate
airflow users create --username <user> --role Admin --email <email> ...
```

- **Executor**: LocalExecutor
- **Metadata DB**: PostgreSQL

### 크리덴셜 관리

Oracle / MySQL 접속 정보는 Airflow Connections(Web UI)에서 관리합니다.  
DAG에서 `BaseHook.get_connection()`으로 조회해 `env_vars`로 컨테이너에 주입합니다.

| Connection ID | 용도 |
|---|---|
| `oracle_source` | Oracle DB 접속 정보 |
| `mysql_target` | MySQL DB 접속 정보 |

### AWS 자격증명

`~/.aws/credentials`에 저장되며, DockerOperator 실행 시 컨테이너에 자동 마운트됩니다.

### Docker 이미지 빌드

```bash
docker build -t aws-s3-warehouse:latest ./docker/aws-s3-warehouse/
docker build -t oracle-to-mysql:latest ./docker/oracle-to-mysql/
```

`oracle-to-mysql` 빌드 시 `instantclient-basic-linux.x64-11.2.0.4.0.zip`을 해당 디렉토리에 미리 복사해야 합니다.

### 자동 시작 (Windows 작업 스케줄러)

매일 KST 08:35에 `start_airflow.sh`를 자동 실행합니다.

- 프로그램: `wsl.exe`
- 인수: `-d Ubuntu -e bash /home/jstcom/airflow-project/start_airflow.sh`

---

## 설계 원칙

- **컨테이너 격리**: 각 태스크는 독립된 Docker 컨테이너에서 실행되어 의존성 충돌 없음
- **자동 재시도**: 실패 시 5분 간격으로 1회 자동 재시도 (`retries=1`)
- **부분 실패 허용**: `trigger_rule="all_done"` 적용으로 앞 태스크 실패 시에도 후속 태스크 진행
- **크리덴셜 분리**: DB 접속 정보는 Airflow Connections에서 관리, 코드에 포함하지 않음
- **날짜 주입**: Airflow `{{ ds }}`를 `--date` 인수로 전달하여 timezone 오류 없이 날짜 처리
- **이메일 알림**: 태스크 실패 시 자동 이메일 알림 발송

---

## 기술 스택

Python / Apache Airflow / Docker / Oracle DB / MySQL /
AWS S3 / AWS Athena / Boto3 / Pandas / Scikit-learn / Statsmodels
