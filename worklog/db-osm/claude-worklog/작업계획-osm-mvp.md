# OSM MVP 적재 작업계획

- **작성일:** 2026-04-27
- **근거 문서:** `be/worklog/작업지시-osm-mvp.md`
- **상태:** 승인 완료, 구현 대기 중

---

## 0. Context

러닝 앱 MVP의 세 기능(자유 러닝 경로 검증 / 공유 경로 유사도 검증 / 에디터 드로잉 런 유효성 검사)이 모두 "이 GPS 좌표가 보행 가능 도로 위에 있는가"를 판단하는 데 의존한다. 이 판단의 기준 데이터를 OpenStreetMap에서 받아 PostgreSQL + PostGIS에 1회성으로 적재하는 작업이다.

- 대상 지역: 서울 마포구
- 데이터 소스: Overpass API
- 런타임: Python 3.11+
- DB: PostgreSQL 16 + PostGIS 3.4 (도커 컨테이너 `localPostgis`)
- 스케줄러 없음 / 운영 서버 적재 없음(이번 작업은 로컬 한정) / 그래프 테이블 없음

### 환경 점검 결과
- **git 루트는 `자율/`**. be/, fe/, ai/, docs/, exec/, infra/가 자율 리포의 하위 디렉토리. data/는 신규 생성.
- `be/docker-compose.yml`의 `localPostgis` 서비스가 작업지시의 모든 가정(컨테이너명/DB명/유저/비번/포트/이미지)과 일치. sql 자동 마운트 없음 확인.
- `be/sql/`, 기존 `nodes` 테이블, 기존 osm 테이블 모두 부재 → 충돌 없음.
- `be/.gitignore`에 `/worklog`가 이미 ignore 처리됨(개인 파일 정책). worklog는 git 추적되지 않지만 로컬에는 남음 → 작업지시 9번 의도("다음 세션 Claude Code가 worklog + 작업지시로 이어가기")와 정합.
- 자율/.gitignore가 `*.env`, `*.env.*`, `!*.env.example`를 이미 처리 → osm-loader/.env 자동 ignore, .env.example만 추적.

### 작업/커밋 흐름
- **git add/commit은 사용자가 직접 수행한다.** Claude는 한 커밋 단위(코드 변경 + 해당 worklog)까지 만들고 멈춘다.
- 사용자가 변경 검토 → git add/commit → 다음 커밋 진행 지시 → 다음 단위 시작.

---

## 1. 디렉토리 구조 결정

osm-loader는 자율 리포 안에서 be와 **형제 디렉토리**(`자율/data/osm-loader/`)로 둔다. 작업지시 1번이 "새 프로젝트"라고 표현했고, be 빌드 산출물과 섞이지 않게 분리. worklog는 작업지시 9번을 따라 `be/worklog/`에 그대로 작성한다.

```
자율/                                                  (git 루트)
├── .gitignore                                       (.env 정책 처리, 수정 X)
├── be/                                              (기존 Spring Boot)
│   ├── .gitignore                                   (/worklog ignore, 수정 X)
│   └── worklog/                                     (git 추적 X)
│       ├── 작업지시-osm-mvp.md                     (기존)
│       ├── 작업계획-osm-mvp.md                     (본 문서)
│       └── 2026-04-27_OSM_*.md × 6                 (커밋별 worklog)
├── ai/, fe/, docs/, exec/, infra/                   (기존, 무관)
└── data/                                            (신규)
    └── osm-loader/
        ├── .env.example
        ├── .gitignore                              (Python 관련만)
        ├── requirements.txt
        ├── sql/
        │   ├── 01_extensions.sql
        │   ├── 02_schema.sql
        │   └── 03_verify.sql
        └── loader/
            ├── __init__.py
            ├── config.py
            ├── overpass_client.py
            ├── parser.py
            ├── db.py
            └── main.py
```

be 리포 자체 코드 변경 없음. worklog 추가 분도 `/worklog` ignore로 git 무관. 모든 git 추적 변경은 `data/osm-loader/` 안에서만 발생.

---

## 2. 커밋 분할 (6단위)

작업지시 11번에 따라 매 커밋 후 worklog 작성 → 사용자 학습/검토 완료 후 다음 커밋으로 진행한다.

> **각 커밋의 흐름**: Claude가 코드 변경 + worklog 작성까지 완료 후 멈춤 → 사용자가 변경 내용 검토 → 사용자가 직접 `git add` / `git commit` → 사용자가 다음 단계 진행 지시 → 다음 커밋 시작.

### C1. 프로젝트 부트스트랩
**범위**
- `자율/data/osm-loader/` 디렉토리 트리 생성
- `requirements.txt`: `psycopg2-binary>=2.9`, `requests>=2.31`, `python-dotenv>=1.0`
- `.env.example`: 작업지시 6번 환경변수 5개
- `.gitignore`: `__pycache__/`, `*.pyc`, `venv/`, `.python-version` (`.env`는 자율/.gitignore가 이미 처리)
- `loader/__init__.py` (빈 파일)
- venv 생성 및 `pip install -r requirements.txt` 실행

**완료 기준**: 디렉토리/파일 존재, venv 활성화 후 `python -c "import psycopg2, requests, dotenv"` 무에러
**worklog**: `2026-04-27_OSM_프로젝트_부트스트랩.md`

### C2. DB 스키마 및 적용
**범위**
- `sql/01_extensions.sql` — PostGIS extension
- `sql/02_schema.sql` — `osm_ways`, `osm_crossings` + GIST 공간 인덱스 + highway 인덱스
- `sql/03_verify.sql` — 작업지시 7번 4개 쿼리 + ST_IsValid 검증 쿼리 1개 추가
- 컨테이너에 01, 02 적용

**완료 기준**: `\dt`에 두 테이블 노출, `\d osm_ways`에서 14개 컬럼 + 2개 인덱스(공간 GIST + highway b-tree) 확인
**worklog**: `2026-04-27_OSM_DB_스키마_생성.md`

### C3. config + db 모듈
**범위**
- `loader/config.py`
  - `load_db_config() -> dict`: python-dotenv로 `.env` 로드. host/port/db/user/password 5필드 반환.
- `loader/db.py`
  - `get_connection(cfg) -> psycopg2.connection`
  - `insert_ways_batch(conn, ways: list[dict], batch_size=500) -> int`
    - SQL: `INSERT INTO osm_ways (...) VALUES (..., ST_GeomFromText(%s, 4326), ...) ON CONFLICT (osm_id) DO NOTHING`
    - `executemany`로 500건씩 처리, batch별 commit, 누적 적재 건수 반환
  - `insert_crossings_batch(conn, crossings: list[dict], batch_size=500) -> int` (동일 패턴, POINT)

**기술 결정**
- WKT 입력은 `ST_GeomFromText('LINESTRING(lon lat, ...)', 4326)`로 SRID 명시 (작업지시 5번 좌표 순서 주의 반영)
- 트랜잭션은 batch별 commit. ON CONFLICT DO NOTHING이라 재실행 멱등.

**완료 기준**: `python -c "from loader.config import load_db_config; from loader.db import get_connection; get_connection(load_db_config()).close()"` 무에러
**worklog**: `2026-04-27_OSM_DB_모듈.md`

### C4. Overpass 클라이언트
**범위**
- `loader/overpass_client.py`
  - `QUERY` 상수: 작업지시 4번 쿼리 그대로 하드코딩
  - `fetch_overpass(timeout=300, max_retries=3) -> dict`: requests.post(`https://overpass-api.de/api/interpreter`), `data={"data": QUERY}`. 실패 시 지수 백오프 1s → 2s → 4s. 3회 모두 실패 시 예외 raise.
  - logging.INFO로 요청/응답/재시도 로그

**완료 기준**: `python -m loader.overpass_client` 실행 시 응답 수신, `len(elements)` 출력
**worklog**: `2026-04-27_OSM_Overpass_클라이언트.md`

### C5. 파서
**범위**
- `loader/parser.py`
  - `parse_way(element: dict) -> dict | None`
    - geometry 좌표 < 2개면 `None`
    - 좌표 변환: `[{"lat":y,"lon":x},...]` → `"LINESTRING(x y, x y, ...)"` (lon lat 순)
    - 14개 필드 추출 (작업지시 5번 표 그대로)
  - `parse_node(element: dict) -> dict`
    - lat/lon → `"POINT(lon lat)"`
    - 4개 필드 추출
  - `parse_response(data: dict) -> tuple[list[dict], list[dict]]`
    - `data["elements"]`를 `type`별로 분기, way는 None 필터링.

**완료 기준**: 샘플 응답 한 건으로 way/node 파싱 sanity check
**worklog**: `2026-04-27_OSM_파서.md`

### C6. main + 적재 실행 + 검증
**범위**
- `loader/main.py`
  - 흐름: config 로드 → DB 연결 → fetch_overpass → parse_response → invalid geom 제거(parse 단계에서 이미 처리) → insert_ways_batch → insert_crossings_batch → 적재 건수 로그
- `python -m loader.main` 실행
- `docker exec -i localPostgis psql -U root -d running < sql/03_verify.sql`로 검증

**완료 기준 (작업지시 8번 체크리스트 그대로)**
- [ ] `localPostgis` 컨테이너 기동 (`docker ps`)
- [ ] `osm_ways`, `osm_crossings` 테이블 생성
- [ ] `python -m loader.main` 무에러 종료
- [ ] `osm_ways` 적재 건수 > 5,000
- [ ] `osm_crossings` 적재 건수 > 100
- [ ] 검증 3번 쿼리 `EXPLAIN`에서 Index Scan / Bitmap Index Scan 확인
- [ ] 모든 행 `ST_IsValid(geom) = true`

**worklog**: `2026-04-27_OSM_적재_실행_및_검증.md`

---

## 3. 핵심 기술 결정 (계획 시점)

1. **WKT 입력 방식**: `ST_GeomFromText('LINESTRING(lon lat, ...)', 4326)`
   - 이유: psycopg2는 PostGIS 타입 어댑터를 기본 제공하지 않음. WKT 문자열을 PostgreSQL 함수로 변환하는 게 의존성 최소.
   - 대안 탈락: shapely + GeoAlchemy2 → MVP 1회 적재에 과도한 의존성.

2. **재시도**: 직접 구현 (지수 백오프 1s → 2s → 4s)
   - 이유: 시도 횟수가 3회로 적어 tenacity 의존성 추가 불필요.

3. **로깅**: Python `logging` 모듈, INFO 레벨 기본
   - 이유: print는 stderr/stdout 구분이 모호. 운영 적재로 확장 시 레벨 제어가 어려움.

4. **트랜잭션 단위**: batch별 commit
   - 이유: ON CONFLICT DO NOTHING으로 멱등 → 부분 실패 후 재실행이 안전. 전체 트랜잭션으로 묶으면 마지막에 롤백되는 위험만 큼.

5. **Python 환경**: 프로젝트 폴더 안 venv (`자율/data/osm-loader/venv/`)
   - 이유: 시스템 Python 비오염, 격리.

---

## 4. 결정 사항

1. **git**: `자율/`이 이미 git 루트. 별도 리포 init 안 함. **모든 git add/commit은 사용자가 직접 수행**.
2. **운영 서버 적재**: 작업지시 8번이 로컬 기준. 이번 작업 범위 제외, 후속 별도 작업.

---

## 5. 핵심 파일

### 참조 (수정 없음)
- `be/docker-compose.yml` — `localPostgis` 컨테이너 정의
- `be/worklog/작업지시-osm-mvp.md` — DDL/쿼리/파서 명세 원본

### 신규 생성
- `자율/data/osm-loader/` 전체 트리
- `be/worklog/작업계획-osm-mvp.md` (본 문서)
- `be/worklog/2026-04-27_OSM_*.md` × 6 (커밋별 worklog)

### 수정
- 없음 (be 리포 코드 무변경)

---

## 6. E2E 검증

C6 단계에서 한 번에 수행:
1. `docker ps | grep localPostgis` 확인
2. `docker exec -i localPostgis psql -U root -d running -c "\dt"` 두 테이블 노출 확인
3. `python -m loader.main` 실행 → 적재 건수 로그
4. `docker exec -i localPostgis psql -U root -d running < sql/03_verify.sql` 4쿼리 결과 검토
5. `SELECT COUNT(*) FROM osm_ways WHERE NOT ST_IsValid(geom);` → 0이어야 함
