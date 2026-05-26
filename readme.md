# 서울시 침수 위험도 예측 프로젝트

## 프로젝트 개요
서울시 하수관로 수위, 강우량, DEM(수치표고모델), 침수 흔적도 데이터를 활용하여
동 단위 침수 위험도를 예측하는 프로젝트입니다.

---

## 폴더 구조
flood/
├── config.py              # API 키 및 경로 설정
├── run.py                 # 실시간 수집 트리거
├── init_master.py         # 하수관로 위치 마스터 초기 생성
├── fix_master.py          # 위치 마스터 None 좌표 수정
├── README.md              # 프로젝트 설명
│
├── collect/               # 데이터 수집
│   ├── drainpipe_collect.py   # 서울시 하수관로 수위 수집
│   └── rainfall_collect.py    # 기상청 강우량 수집
│
├── preprocess/            # 데이터 전처리
│   ├── dem_process.py         # DEM 전처리
│   ├── flood_process.py       # 침수 흔적도 전처리
│   ├── merge.py               # 정적 데이터 동별 병합
│   └── realtime_merge.py      # 실시간 데이터 동별 병합
│
└── data/
├── raw/               # 원본 데이터
│   ├── dem/           # DEM 타일 (37608, 37705, 37709, 37612)
│   ├── rainfall/      # 강우량 원본 및 격자 엑셀
│   ├── drainpipe/     # 하수관로 수위 원본
│   ├── flood/         # 침수 흔적도 원본
│   └── boundary/      # 서울시 행정경계 shp
│
└── processed/         # 전처리 완료 데이터
├── dem_seoul.csv          # 서울 전체 픽셀별 위경도 + 고도값
├── location_master.csv    # 하수관로 관측소 위치 마스터 (581개)
├── grid_to_dong.csv       # 기상청 격자 → 법정동 코드 매핑
├── pipe_to_dong.csv       # 하수관로 관측소 → 법정동 코드 매핑
├── flood_detail.csv       # 침수 흔적도 상세 데이터 (1132행)
├── flood_by_dong.csv      # 침수 흔적도 동별 집계 (50개 동)
├── merged_by_dong.csv     # 정적 학습 베이스 (467개 동)
└── prediction_input.csv   # 모델 입력용 (실시간 병합)

---

## 데이터 설명

### 원본 데이터 (raw)

| 데이터 | 출처 | 설명 |
|---|---|---|
| DEM | 국토지리정보원 | 수치표고모델 (EPSG:5179, 90m 해상도) |
| 강우량 | 기상청 단기예보 API | 동별 1시간 강수량 (RN1) |
| 하수관로 수위 | 서울시 열린데이터광장 API | 관측소별 실시간 수위 |
| 침수 흔적도 | 서울시 | 침수심선 기반 침수 이력 (EPSG:5179) |
| 행정경계 | VWORLD | 서울시 읍면동 경계 shp |

### 전처리 완료 데이터 (processed)

#### dem_seoul.csv
| 컬럼 | 설명 |
|---|---|
| LAT | 위도 |
| LON | 경도 |
| ELEVATION | 고도값 (m) |

#### location_master.csv
| 컬럼 | 설명 |
|---|---|
| UNQ_NO | 관측소 고유번호 |
| SE_CD | 지역코드 |
| SE_NM | 지역명 |
| PSTN_INFO | 원본 위치 설명 |
| CLEANED_ADDR | 정제된 주소 |
| LAT | 위도 |
| LON | 경도 |

#### grid_to_dong.csv
| 컬럼 | 설명 |
|---|---|
| NX | 기상청 격자 X 좌표 |
| NY | 기상청 격자 Y 좌표 |
| EMD_CD | 법정동 코드 |

#### pipe_to_dong.csv
| 컬럼 | 설명 |
|---|---|
| UNQ_NO | 관측소 고유번호 |
| EMD_CD | 법정동 코드 |
| EMD_NM | 동 이름 |

#### flood_by_dong.csv
| 컬럼 | 설명 |
|---|---|
| EMD_CD | 법정동 코드 |
| FLOOD_COUNT | 침수 발생 횟수 |
| AVG_SHIM | 평균 침수심 (m) |
| MAX_SHIM | 최대 침수심 (m) |
| AVG_AREA | 평균 침수면적 (m²) |
| TOTAL_SCALE | 총 침수 규모 (F_SHIM × F_AREA) |
| AVG_DEPTH_PER_AREA | 평균 단위면적당 침수심 (F_SHIM ÷ F_AREA) |
| RECENT_YR | 가장 최근 침수 년도 |

#### merged_by_dong.csv
467개 동 전체를 기준으로 DEM + 침수 흔적도를 병합한 정적 학습 베이스

| 컬럼 | 설명 |
|---|---|
| EMD_CD | 법정동 코드 |
| EMD_NM | 동 이름 |
| AVG_ELEVATION | 동별 평균 고도 (m) |
| MIN_ELEVATION | 동별 최저 고도 (m) |
| MAX_ELEVATION | 동별 최고 고도 (m) |
| FLOOD_COUNT | 침수 발생 횟수 (없으면 0) |
| AVG_SHIM | 평균 침수심 (없으면 0) |
| MAX_SHIM | 최대 침수심 (없으면 0) |
| AVG_AREA | 평균 침수면적 (없으면 0) |
| TOTAL_SCALE | 총 침수 규모 (없으면 0) |
| AVG_DEPTH_PER_AREA | 평균 단위면적당 침수심 (없으면 0) |
| RECENT_YR | 가장 최근 침수 년도 (없으면 NaN) |

#### prediction_input.csv
merged_by_dong + 실시간 강우량 + 하수관로 수위를 합친 모델 입력용 데이터

| 컬럼 | 설명 |
|---|---|
| AVG_WATL | 동별 평균 하수관로 수위 |
| MAX_WATL | 동별 최대 하수관로 수위 |
| SENSOR_COUNT | 동별 관측소 수 |
| AVG_RN1 | 동별 평균 강수량 |
| MAX_RN1 | 동별 최대 강수량 |
| TOTAL_RN1 | 동별 총 강수량 |

---

## 실행 순서

### 1. 환경 설정
```bash
python -m venv venv
venv\Scripts\activate
python -m pip install rasterio geopandas numpy pandas requests schedule openpyxl
```

### 2. 최초 1회 전처리
```bash
# 위치 마스터 초기 생성
python init_master.py

# DEM 전처리
python preprocess/dem_process.py

# 침수 흔적도 전처리
python preprocess/flood_process.py

# 전체 데이터 동 단위 병합
python preprocess/merge.py
```

### 3. 학습 데이터 수집 (기간 지정)
run.py 에서 주석 해제 후 실행:
```python
drainpipe_train("2026010100", "2026052300")
rainfall_train("20260101", "20260523")
```

### 4. 실시간 수집 시작
```bash
python run.py
```
- 하수관로: 3시간마다 수집 → drainpipe_realtime.csv 덮어쓰기
- 강우량: 6시간마다 수집 → rainfall_realtime.csv 덮어쓰기
- 수집 후 자동으로 prediction_input.csv 갱신
- Ctrl+C 로 종료

---

## API 정보

| API | 출처 | 용도 |
|---|---|---|
| 서울시 열린데이터광장 | data.seoul.go.kr | 하수관로 수위 수집 |
| 카카오 로컬 API | developers.kakao.com | 주소 지오코딩 |
| 기상청 단기예보 API | data.go.kr | 강우량 수집 |

---

## 주의사항
- 기상청 단기예보 API 일일 호출 제한 10,000건