# AWS S3 Data Warehouse & 매출 예측 파이프라인

Oracle DB에서 S3로 데이터를 적재하고, Athena 기반 분석 테이블과
매출 예측 파이프라인을 운영하는 시스템입니다.

---

## 시스템 아키텍처

```
Oracle 운영 DB
      │
      ▼ ETL (Python)
AWS S3 (Parquet)
      │
      ▼ Athena 외부 테이블
분석 / 예측 파이프라인
      │
      ▼
예측 결과 저장 (S3 → Athena)
```

---

## 프로젝트 구조

```
aws_s3_data_warehouse/
│
├── scripts/                          # ETL 파이프라인 (Oracle → S3)
│   ├── etl_main.py                   # ETL 메인 오케스트레이션
│   ├── db_connector.py               # Oracle 연결 모듈
│   ├── data_converter.py             # Parquet 변환 & 타입 최적화
│   ├── s3_uploader.py                # S3 업로드 모듈
│   ├── schema_definition.py          # Athena 호환 스키마 정의
│   ├── run_monthly_migration.py      # 월별 대량 마이그레이션
│   ├── run_daily_batch.py            # 일별 배치 실행
│   └── tools/
│       ├── check_parquet_schema.py   # Parquet 스키마 검증
│       └── verify_schema.py          # Athena 스키마 검증
│
├── sql/                              # SQL 파일 (비공개)
│
├── sales_feature_upload.py           # 예측 피처 데이터 적재 (매출 + 도우)
├── pred_runner.py                    # 매출/주문/도우 예측 실행
├── run_daily.bat                     # 일별 자동 실행 배치
│
├── data_mart_upload.py               # 데이터 마트 S3 업로드
├── customer_upload.py                # 고객 데이터 업로드
├── customer_cohort_mart_upload.py    # 코호트 마트 업로드
├── customer_kmeans_upload.py         # K-Means 군집 분석 & 업로드 (월 1회)
├── customer_summary_mart_upload.py   # 고객 요약 마트 업로드
├── daily_customer_upsert.py          # 일별 고객 Upsert
├── valid_file.py                     # 파일 유효성 검증
│
├── config/
│   └── config.yaml.example           # 설정 파일 예시 (실제 값 미포함)
└── requirements.txt
```

---

## 1. ETL 파이프라인 (Oracle → S3)

Oracle DB에서 데이터를 추출하여 Parquet 형식으로 S3에 적재합니다.

### 데이터 분류

| 구분 | 설명 | 적재 방식 |
|---|---|---|
| **Fact** | 주문·매출 (일별 누적) | 날짜 파티셔닝, 전일 데이터 매일 적재 |
| **Dimension** | 매장·상품·프로모션·고객 | 별도 주기로 전체 갱신 |

### 실행 모드

```bash
# 일별 배치 (전일 데이터)
python scripts/etl_main.py --mode daily

# 특정 날짜
python scripts/etl_main.py --mode daily --date 20241120

# 날짜 범위 (Fact 테이블)
python scripts/etl_main.py --mode range --start-date 20240101 --end-date 20240131

# 월별 대량 마이그레이션
python scripts/run_monthly_migration.py --year 2024
```

### 멱등성 설계

파티션 삭제 후 재적재 방식으로 중복·불일치를 원천 차단합니다.
배치 실패 시 재실행만으로 원상복구됩니다.

---

## 2. 매출 예측 파이프라인

SARIMA(전체 매장) + Ridge 회귀(개별 매장) 모델로
당일 매출·주문수·재고 수요를 예측합니다.

### 일별 실행 (Airflow DockerOperator)

Airflow `daily_pipeline` DAG에서 자동 실행됩니다 (KST 08:40).  
Airflow가 `--date {{ ds }}` 인수를 컨테이너에 주입하여 날짜를 전달합니다.

```
[1] sales_feature_upload.py --date {{ ds }}  → 학습 피처 데이터 적재
[2] pred_runner.py                            → 예측 모델 학습 및 결과 저장
```

### 예측 모델 구성

| 모델 | 예측 대상 | 전체 매장 | 개별 매장 |
|---|---|---|---|
| **Model A** | 일별 매출 + 주문수 | SARIMA (m=7) | Ridge (동요일 lag 피처) |
| **Model B** | 시간대별 주문수 | Model A × 시간대 비율 배분 | 동일 |
| **Model C** | 시간대별 재고 수요 | 동요일 평균 (사이즈별) | 동일 |

### 학습 데이터 범위

| 구분 | 학습 기간 |
|---|---|
| 매출/주문 (Model A, B) | 최근 120일 |
| 재고 수요 (Model C) | 최근 90일 |
| 개별 매장 Ridge | 최근 4개월 동요일 |

### 누락 구간 자동 보정

```python
# 세 예측 테이블의 MAX(pred_date)를 자동 조회
# → 가장 오래된 날짜 + 1일부터 당일까지 순차 예측
# → 배치 실패 후 재실행만으로 자동 복구
OVERRIDE_TARGET_DATE = None   # 특정 날짜 고정 시 지정
SALES_LOOKBACK_DAYS  = 120
DOUGH_LOOKBACK_DAYS  = 90
```

---

## 3. 고객 분석

### K-Means RFM 군집 분석 (월 1회)

고객 요약 마트 기반 5개 클러스터 자동 군집화

| 클러스터 | 세그먼트 |
|---|---|
| 1 | Lost (저가치) |
| 2 | At Risk (이탈 위험) |
| 3 | Potential (잠재 고객) |
| 4 | Loyal (충성 고객) |
| 5 | Champion (최우수 고객) |

---

## 4. 공통 설계 원칙

- **Parquet 포맷**: 컬럼형 저장으로 Athena 스캔 비용 최소화
- **날짜 파티셔닝**: year / month / day 기준 Fact 테이블 파티셔닝
- **멱등성**: 파티션 삭제 후 재적재로 중복 방지
- **자동화**: Airflow DockerOperator 기반 무인 운영
- **일 평균 처리량**: 약 6만 건

---

## 5. 환경 설정

```bash
# 의존성 설치
pip install -r requirements.txt

# 설정 파일 복사 후 값 입력
cp config/config.yaml.example config/config.yaml
```

### 필수 요건

- Python 3.10+ (Docker 컨테이너 기반 실행)
- Oracle DB 접근 권한
- AWS CLI 설정 완료 (`aws configure`)
- Oracle Instant Client 11.2 (Docker 이미지에 내장)

---

## 기술 스택

Python / Oracle PL/SQL / AWS S3 / AWS Athena / Parquet /
Pandas / Polars / Scikit-learn / Statsmodels / Boto3
