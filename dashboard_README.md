# PJI Store Dashboard

프랜차이즈 250개 매장의 매출·제품·고객·예측 데이터를 통합 분석하는
Python Dash 기반 웹 대시보드입니다.

---

## 기술 스택

| 구분 | 기술 |
|---|---|
| 웹 프레임워크 | Dash (Python) + Dash Bootstrap Components |
| 차트 | Plotly |
| 데이터 소스 (Fact/Dimension) | AWS S3 — Parquet (s3fs + pyarrow) |
| 데이터 소스 (분석 마트) | AWS Athena (pyathena) |
| 인증 DB | Oracle DB (oracledb) |
| 서버 캐시 | Flask-Caching (TTL 5분) |
| 환경 변수 | python-dotenv |

---

## 프로젝트 구조

```
dash_project/
│
├── app.py                          # 진입점 — Dash 앱 초기화, 레이아웃, 공통 콜백
├── server.py                       # Dash 인스턴스 + Flask-Caching 설정
├── requirements.txt
│
├── auth/                           # 인증 모듈
│   ├── login_manager.py            # 로그인·로그아웃·잠금 관리
│   ├── session_manager.py          # 메모리 기반 세션 관리
│   └── oracle_connector.py         # Oracle DB 연결 (일반 사용자 인증)
│
├── data/                           # 데이터 레이어
│   ├── loader.py                   # S3 Parquet 로더 (Dimension + Fact)
│   ├── base_loader.py              # Athena 공통 인프라 (연결, 파티션 필터)
│   ├── sales_loader.py             # 매출 분석 쿼리
│   ├── product_loader.py           # 제품 분석 쿼리
│   ├── customer_loader.py          # 고객 분석 쿼리
│   ├── compare_loader.py           # 매장 비교 탭 쿼리
│   └── mart_loader.py              # 마트 공통 쿼리
│
├── pages/                          # 페이지 레이아웃
│   ├── dashboard.py                # 공통 레이아웃 (헤더·사이드바·탭)
│   ├── login.py                    # 로그인 페이지
│   ├── tab_sales.py                # 매출 분석 탭
│   ├── tab_product.py              # 제품 분석 탭
│   ├── tab_customer.py             # 고객 분석 탭
│   ├── tab_prediction.py           # 예측 탭
│   ├── tab_compare.py              # 매장 비교 탭
│   └── tab_region_compare.py       # 지역 평균 비교 탭
│
├── callbacks/                      # Dash 콜백 모듈
│   ├── common_callbacks.py         # 공통 콜백 (탭 전환, 날짜, 비교 모드)
│   ├── sales_callbacks.py          # 매출 탭 콜백
│   ├── product_callbacks.py        # 제품 탭 콜백
│   ├── customer_callbacks.py       # 고객 탭 콜백
│   ├── prediction_callbacks.py     # 예측 탭 콜백
│   ├── compare_callbacks.py        # 매장 비교 탭 콜백
│   └── region_compare_callbacks.py # 지역 평균 비교 탭 콜백
│
└── assets/                         # 정적 파일 (CSS, 이미지)
```

---

## 탭 구성 (6개)

| 탭 | 주요 기능 |
|---|---|
| **매출 분석** | KPI 카드 / 트렌드(일·주·월·분기·년) / YOY·MOM·WOW 비교 / 채널 도넛 / 시간대 / 매장 순위 / 지역별 집계 |
| **제품 분석** | 판매 순위 / 사이즈·크러스트 Mix 테이블(판매량·비율·YOY) / 쿠폰 순위 |
| **고객 분석** | RFM 산점도·파레토 / 코호트 보존율 히트맵 / K-Means 군집 / 성별·연령대·구매 분포 프로파일 |
| **예측** | SARIMA + Ridge 매출·주문수 예측 결과 / 전일 정확도 표시 |
| **매장 비교** | 기준 매장 vs 최대 5개 비교 / KPI 카드(총매출·일평균·객단가·주문수·할인율·기준 대비 ▲▼%) |
| **지역 평균 비교** | 기준 매장 vs 선택 지역 일평균 / 단일 날짜·기간 구분 콤보 차트 |

---

## 인증 시스템

```
로그인 요청
    ├→ 슈퍼관리자: 환경변수 비교 → 전체 매장 권한
    └→ 일반 사용자: Oracle DB 조회 → 권한 매장만 접근
           └→ 5회 실패 시 10분 잠금
              (슈퍼관리자 일괄 해제 가능)
```

- 세션: 서버 메모리 기반, 기본 만료 30분 (활동 시 자동 연장)

---

## 데이터 레이어 설계

### S3 Parquet (앱 시작 시 1회 로드)
- Dimension 데이터 (매장·제품·스페셜티 마스터) → 메모리 유지
- Fact 데이터 (일별 주문·매출) → 탭 전환·날짜 변경 시 로드

### Athena (쿼리 기반, TTL 5분 캐시)
- 매출·제품·고객·예측 분석 마트 조회
- 파티션 필터 자동 생성으로 스캔 비용 최소화

### 성능 최적화 사례
- 고객 프로파일: 100만 건 raw 데이터 전송 → Athena 서버 사이드 GROUP BY / CASE WHEN 집계로 전환
- 결과: 브라우저 전송 데이터 99% 이상 감소, 로딩 3초 이내 단축

---

## 실행 방법

```bash
# 의존성 설치
pip install -r requirements.txt

# 환경 변수 설정
cp .env.example .env
# .env 파일에 AWS / Athena / Oracle 연결 정보 입력

# 실행
python run.py
```

---

## 기술 스택

Python / Dash / Plotly / Pandas / PyArrow / s3fs /
PyAthena / oracledb / Flask-Caching / Boto3
