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
│   ├── daily_pipeline_dag.py     # 메인 파이프라인 (DockerOperator, 23:40 실행)
│   └── run_daily_dag.py          # 보조 파이프라인 (BashOperator, 자정 실행)
│
├── docker/
│   ├── aws-s3-warehouse/
│   │   ├── Dockerfile
│   │   └── requirements.txt      # boto3, pandas, sklearn 등
│   └── oracle-to-mysql/
│       ├── Dockerfile
│       └── requirements.txt      # oracledb, pymysql 등
│
├── requirements.txt              # Airflow 환경 의존성
├── setup_aws_credentials.py      # AWS 자격증명 설정 스크립트
└── .gitignore
```

---

## DAG 구성

### `daily_pipeline` — 매일 23:40 실행

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

### `run_daily` — 매일 자정 실행

BashOperator 방식으로 예측 파이프라인만 단독 실행합니다.

```
sales_feature_upload → pred_runner
```

---

## Docker 이미지

### `oracle-to-mysql:latest`

Oracle DB에서 데이터를 추출하여 MySQL로 적재합니다.

```
python migrate.py --table <table_name>
python migrate.py --group kpi
```

**주요 패키지**: oracledb, pymysql, pandas, python-dotenv

### `aws-s3-warehouse:latest`

S3 데이터 적재 및 예측 모델 실행을 담당합니다.

**주요 패키지**: boto3, s3fs, oracledb, pandas, pyarrow, scikit-learn, pmdarima, statsmodels

---

## 환경 설정

### Airflow 설치

```bash
python -m venv ~/airflow-env
source ~/airflow-env/bin/activate
pip install -r requirements.txt
airflow standalone
```

### AWS 자격증명 설정

```bash
python setup_aws_credentials.py
```

입력 후 `~/.aws/credentials`에 저장되며, DockerOperator 실행 시 컨테이너에 자동 마운트됩니다.

### Docker 이미지 빌드

```bash
docker build -t aws-s3-warehouse:latest ./docker/aws-s3-warehouse/
docker build -t oracle-to-mysql:latest ./docker/oracle-to-mysql/
```

---

## 설계 원칙

- **컨테이너 격리**: 각 태스크는 독립된 Docker 컨테이너에서 실행되어 의존성 충돌 없음
- **자동 재시도**: 실패 시 5분 간격으로 1회 자동 재시도 (`retries=1`)
- **부분 실패 허용**: `trigger_rule="all_done"` 적용으로 앞 태스크 실패 시에도 후속 태스크 진행
- **자격증명 분리**: AWS credentials는 Git에 포함하지 않고 bind mount로 주입

---

## 기술 스택

Python / Apache Airflow / Docker / Oracle DB / MySQL /
AWS S3 / AWS Athena / Boto3 / Pandas / Scikit-learn / Statsmodels
