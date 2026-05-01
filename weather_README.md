# 매장별 날씨 데이터 수집기 (weather_data)

기상청 API를 활용하여 매장 위치 기반으로 날씨 데이터를 수집하고,
매출 예측 모델의 피처로 활용하기 위한 프로젝트입니다.

---

## 프로젝트 배경

기존에 사용하던 전국 단위 하루 평균 날씨 데이터는 두 가지 한계가 있었습니다.

- 전국 단위 평균이라 **매장 위치별 차이를 반영할 수 없음**
- 하루 단위 집계라 **배달 피크 시간대의 날씨를 구분하지 못함**

이를 해결하기 위해 매장별 개별 날씨 피처를 설계하고 수집하는 파이프라인을 구축했습니다.

---

## 피처 설계

- 각 매장 좌표 → 가장 가까운 기상 관측소 자동 매핑
- 수집 시간: 배달 피크 점심(11~13시) / 저녁(17~20시) 두 구간
- 저녁 시간대에 가중치 부여 → 단일 날씨 지수 산출
- 4단계로 분류하여 일별 피처로 저장
- 결과: **매장마다 하루 하나의 날씨 값**을 갖는 일별 피처

---

## 주요 기능

- **좌표 변환**: 위경도 → 기상청 DFS 격자 좌표 변환 (Lambert Conformal Conic 투영)
- **관측소 매핑**: 매장별 최근접 ASOS 기상관측소 자동 매핑
- **데이터 수집**: 기상청 동네예보 API / ASOS 과거 관측 API
- **배치 처리**: 전체 매장 일괄 수집 및 스케줄러 자동화
- **재시도 로직**: 수집 실패 매장만 선택적으로 재시도

---

## 프로젝트 구조

```
weather_data/
│
├── src/
│   ├── geo/
│   │   ├── dfs.py                      # 위경도 → DFS 격자 변환
│   │   ├── preprocess_store_grid.py    # 격자 좌표 전처리
│   │   ├── station_matcher.py          # 관측소-매장 매핑
│   │   └── create_station_mapping.py   # 매핑 생성 CLI
│   │
│   └── weather/
│       ├── features.py                 # 날씨 피처 추출
│       ├── storage.py                  # 데이터 저장
│       ├── forecast/
│       │   ├── kma_client.py           # 기상청 예보 API 클라이언트
│       │   ├── fetch_store_weather.py  # 매장별 날씨 수집 CLI
│       │   └── fetch_precipitation.py  # 강수량 수집
│       └── observation/
│           ├── asos_client.py          # ASOS 과거 관측 API 클라이언트
│           └── fetch_historical.py     # 과거 데이터 수집 CLI
│
├── store_data/                         # 매장 위치 데이터 (비공개)
├── data/                               # 수집된 날씨 데이터 (비공개)
│
├── collect_precipitation_simple.py     # 전체 매장 강수량 수집
├── retry_failed_stores.py              # 실패 매장 재시도
├── scheduler.py                        # 자동 수집 스케줄러
├── start_scheduler.bat                 # 스케줄러 시작 (Windows)
└── stop_scheduler.bat                  # 스케줄러 중지 (Windows)
```

---

## 데이터 수집 전략

```
과거 데이터 (3개월 이전)
→ ASOS 관측 데이터 활용 (참고용)

최근 데이터 (3개월 이내)
→ 기상청 동네예보 API (5km 격자, 매장별 정확한 위치)

수집 주기
→ 영업시간(10:00~22:00) 매시간 자동 수집 (스케줄러)
```

---

## 현재 진행 상황

- 일별 배치 실행 시 전체 매장 기준 수집 성공률 **70~80%**
- 기상청 API 일일 호출 제한 및 간헐적 호출 실패가 원인
- 실패 매장만 선택적으로 재시도하는 로직 개발 중
- 완성 후 예측 모델에 통합하여 정확도 개선 효과 검증 예정

---

## 기술 스택

Python / Requests / Pandas / Schedule / 기상청 KMA API / ASOS API
