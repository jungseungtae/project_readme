# 매출 예측 모델 벤치마킹 (pred_models_valid)

프랜차이즈 매장의 매출·주문수·재고 수요 예측 모델을 운영 투입 전,
최적 모델과 학습 기간을 검증하기 위한 벤치마킹 프레임워크입니다.

---

## 비교 모델 (7가지)

| 구분 | 모델 |
|---|---|
| 통계 | Seasonal Naive / ETS / SARIMA |
| ML | XGBoost / LightGBM / Random Forest / Ridge |

---

## 학습 기간 조합 (12가지)

최근 데이터 2~4개월 × 전년도 동기 데이터 0~12개월 조합
→ 총 **84가지 시나리오** 비교

---

## 검증 방식

Walk-Forward CV (3-fold, 테스트 30일) — 미래 데이터 누수 없음

---

## 평가 지표

| 지표 | 설명 |
|---|---|
| SMAPE | 대칭 평균 절대 백분율 오차 (주요 기준) |
| MAPE | 평균 절대 백분율 오차 |
| MAE | 평균 절대 오차 |
| RMSE | 평균 제곱근 오차 |

---

## 벤치마크 결과 요약

- **최적 모델**: SARIMA (최근 4개월 단독 학습)
- 전년도 데이터 추가 시 오히려 성능 저하 확인
  → 최근 추세가 계절성보다 중요한 업종 특성 반영
- 전체 통합 기준 **SMAPE 16.4%** 달성

---

## 프로젝트 구조

```
pred_models_valid/
│
├── forecast_benchmark.py       # 전체 매출 — 7모델 × 12기간 84가지 조합 비교
├── store_sample_benchmark.py   # 개별 매장 — 매장별 최적 모델·기간 탐색
├── forecast_weekday.py         # 요일 기반 접근법 — 동일 요일 이력만 사용
├── predict_yesterday.py        # 단일 날짜 예측 검증 — 실제값 vs 예측값 비교
├── forecast_store.py           # 매장별 SARIMA 단독 평가
│
├── benchmark_output/           # 전체 벤치마크 결과
├── store_sample_benchmark/     # 매장별 벤치마크 결과
├── weekday_forecast_output/    # 요일 기반 결과
└── predict_yesterday_output/   # 단일 날짜 검증 결과
```

---

## 기술 스택

Python / Pandas / Scikit-learn / Statsmodels / XGBoost / LightGBM
