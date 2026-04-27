# 작업지시: OSM 데이터 적재 (MVP)

- **작성일:** 2026-04-27
- **작업 유형:** 1회성 풀 적재 (스케줄러 없음)
- **대상 지역:** 서울특별시 마포구
- **데이터 소스:** OpenStreetMap (Overpass API)
- **언어/런타임:** Python 3.11+
- **DB:** PostgreSQL 16 + PostGIS 3.4

---

## 0. 목적

러닝 앱의 아래 세 가지 MVP 기능을 위한 공간 데이터(도보 가능 도로)를 DB에 적재한다.

| 기능                | 적재 데이터가 하는 역할                   |
| ----------------- | ------------------------------- |
| 자유 러닝 경로 검증       | GPS 좌표 배열이 보행 가능 way 위를 지나는지 판별 |
| 공유 경로 유사도 검증      | 저장된 way geometry로 경로 간 거리 비교    |
| 에디터(드로잉 런) 유효성 검사 | 사용자가 찍은 좌표가 보행 가능한 길인지 실시간 판별   |

---

## 1. 레포지토리 구조 (새 프로젝트)

```
osm-loader/
├── .env.example
├── requirements.txt
├── sql/
│   ├── 01_extensions.sql       # PostGIS extension
│   ├── 02_schema.sql           # 테이블 + 인덱스 DDL
│   └── 03_verify.sql           # 적재 검증 쿼리 모음
└── loader/
    ├── __init__.py
    ├── config.py               # 환경변수 로드
    ├── overpass_client.py      # Overpass API 쿼리
    ├── parser.py               # API 응답 → Python 객체
    ├── db.py                   # DB 연결, upsert 함수
    └── main.py                 # 진입점
```

---

## 2. Docker 환경

Docker Compose는 기존 프로젝트 루트의 파일을 그대로 사용한다. **이 프로젝트에서 별도 docker-compose를 만들지 않는다.**

### 로컬 접속 정보

| 항목       | 값                        |
| -------- | ------------------------ |
| 컨테이너명    | `localPostgis`           |
| host     | `localhost`              |
| port     | `5432`                   |
| database | `running`                |
| user     | `root`                   |
| password | `ssafy1234!`             |
| image    | `postgis/postgis:16-3.4` |

### DDL 실행 방법

기존 docker-compose에 `./sql` 자동 마운트가 없으므로, 컨테이너 기동 후 수동으로 실행한다.

```bash
# 컨테이너가 이미 기동 중이어야 함
docker exec -i localPostgis psql -U root -d running < sql/01_extensions.sql
docker exec -i localPostgis psql -U root -d running < sql/02_schema.sql
```

---

## 3. DB 스키마

> **기존 ERD의 `nodes` 테이블과 완전히 분리된 별도 테이블을 생성한다.**  
> 기존 `nodes`는 사용자 GPS 경로 저장용이며 추후 별도로 수정될 예정이므로 건드리지 않는다.

### `sql/01_extensions.sql`

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
```

### `sql/02_schema.sql`

```sql
-- ============================================================
-- osm_ways: 보행 가능 도로 세그먼트
-- ============================================================
CREATE TABLE IF NOT EXISTS osm_ways (
    id          BIGSERIAL       PRIMARY KEY,
    osm_id      BIGINT          NOT NULL UNIQUE,     -- OSM way ID (전역 고유)
    highway     VARCHAR(50)     NOT NULL,             -- 도로 분류
    name        VARCHAR(255),                         -- 도로명
    geom        GEOMETRY(LINESTRING, 4326) NOT NULL,  -- 좌표계: WGS84

    -- 통행 제어 (검증 로직에 직접 사용)
    foot        VARCHAR(20),    -- yes / no / designated / permissive
    access      VARCHAR(20),    -- yes / no / private / customers
    oneway      VARCHAR(10),    -- yes / no / -1

    -- 구조 속성 (GPS 끊김 구간 면책 처리용)
    bridge      BOOLEAN         DEFAULT FALSE,
    tunnel      BOOLEAN         DEFAULT FALSE,
    indoor      BOOLEAN         DEFAULT FALSE,
    layer       SMALLINT        DEFAULT 0,

    -- 러닝 UX 속성 (MVP 이후 기능용이나 지금 같이 적재)
    surface     VARCHAR(30),    -- asphalt / concrete / dirt ...
    lit         BOOLEAN,        -- 야간 안전 경로용
    sidewalk    VARCHAR(20),    -- both / left / right / none

    created_at  TIMESTAMP       NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- 공간 인덱스 (ST_DWithin 등 공간 쿼리 필수)
CREATE INDEX IF NOT EXISTS idx_osm_ways_geom
    ON osm_ways USING GIST(geom);

-- 도로 종류별 필터링용
CREATE INDEX IF NOT EXISTS idx_osm_ways_highway
    ON osm_ways(highway);


-- ============================================================
-- osm_crossings: 횡단보도 노드
-- 경로 생성·검증 시 도보 가능한 way 간 연결 판단에 사용
-- ============================================================
CREATE TABLE IF NOT EXISTS osm_crossings (
    id          BIGSERIAL       PRIMARY KEY,
    osm_id      BIGINT          NOT NULL UNIQUE,
    crossing    VARCHAR(30),    -- marked / unmarked / zebra / traffic_signals
    has_signals BOOLEAN         DEFAULT FALSE,
    geom        GEOMETRY(POINT, 4326) NOT NULL,

    created_at  TIMESTAMP       NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_osm_crossings_geom
    ON osm_crossings USING GIST(geom);
```

### 테이블 설계 의도 요약

| 테이블             | 설명                                                                       |
| --------------- | ------------------------------------------------------------------------ |
| `osm_ways`      | 도로 세그먼트 하나 = 행 하나. `geom`(LineString)으로 "이 좌표가 도로 위인가"를 ST_DWithin으로 판별. |
| `osm_crossings` | `highway=crossing` 노드만 별도 저장. 두 way가 횡단보도로 연결되는지 확인할 때 사용.               |

> **`osm_crossings` 운용 방침:**  
> MVP 검증 로직은 crossing 노드에 의존하지 않는다. way 간 거리 기반 연결 판단을 1순위로 쓰고, crossing은 보조 데이터로만 활용한다.  
> 이유: 한국 OSM의 `highway=crossing` 태깅 커버리지가 낮아(주요 교차로 일부만 존재) crossing 유무로 연결성을 판단하면 오탐이 많이 발생함.  
> 테이블은 지금 적재해두고, 추후 국가공간정보포털 등 외부 소스로 데이터를 보완하면 검증 로직 수정 없이 자동으로 정확도가 향상되는 구조로 설계한다.

> **MVP에서 제외하는 것:** 노드-엣지 그래프 테이블(pgRouting용). 연결성 검증은 ST_DWithin + ST_Intersects로 대체하고, 그래프 기반 라우팅은 다음 단계에서 추가한다.

---

## 4. Overpass API 쿼리

### 적재 대상 highway 분류

| 카테고리         | highway 값                                                 | 포함 이유     |
| ------------ | --------------------------------------------------------- | --------- |
| 보행 전용        | `footway`, `pedestrian`, `path`, `steps`                  | 1차 보행 인프라 |
| 생활권          | `residential`, `living_street`, `unclassified`, `service` | 주거지 러닝    |
| 간선(인도 포함 가정) | `primary`, `secondary`, `tertiary`, `*_link`              | 도심 경로 연결  |
| 트레일·자전거      | `cycleway`, `track`                                       | 한강변, 공원   |

**제외:** `motorway`, `trunk`, `motorway_link`, `trunk_link` (자동차 전용)

### Overpass 쿼리 (`loader/overpass_client.py`에 하드코딩)

```
[out:json][timeout:300];
area["name"="마포구"]["boundary"="administrative"]->.searchArea;
(
  way["highway"~"^(footway|pedestrian|path|steps|residential|living_street|unclassified|service|primary|secondary|tertiary|primary_link|secondary_link|tertiary_link|cycleway|track)$"]
     ["foot"!="no"]
     ["access"!="private"]
     ["access"!="no"]
     (area.searchArea);
  node["highway"="crossing"](area.searchArea);
);
out tags geom;
```

- `out tags geom`: way의 모든 태그 + geometry 좌표를 함께 반환
- 응답 형식: JSON

### API 엔드포인트

```
https://overpass-api.de/api/interpreter
```

요청 방식: POST, body: `data=<query>`

---

## 5. 파서 명세 (`loader/parser.py`)

Overpass JSON 응답에서 아래 필드만 추출한다.

### Way 파싱

```python
# 응답 element 예시
{
  "type": "way",
  "id": 12345678,
  "geometry": [{"lat": 37.55, "lon": 126.92}, ...],
  "tags": {
    "highway": "footway",
    "name": "상암로",
    "foot": "designated",
    "lit": "yes",
    "surface": "asphalt",
    ...
  }
}
```

추출 규칙:

| 필드         | 소스                                       | 변환                                   |
| ---------- | ---------------------------------------- | ------------------------------------ |
| `osm_id`   | `element["id"]`                          | 그대로                                  |
| `highway`  | `tags["highway"]`                        | 그대로                                  |
| `name`     | `tags.get("name")`                       | 없으면 None                             |
| `geom`     | `element["geometry"]`                    | `[(lon, lat), ...]` → WKT LineString |
| `foot`     | `tags.get("foot")`                       | 없으면 None                             |
| `access`   | `tags.get("access")`                     | 없으면 None                             |
| `oneway`   | `tags.get("oneway")`                     | 없으면 None                             |
| `bridge`   | `tags.get("bridge") == "yes"`            | bool                                 |
| `tunnel`   | `tags.get("tunnel") == "yes"`            | bool                                 |
| `indoor`   | `tags.get("indoor") == "yes"`            | bool                                 |
| `layer`    | `int(tags.get("layer", 0))`              | int, 실패 시 0                          |
| `surface`  | `tags.get("surface")`                    | 없으면 None                             |
| `lit`      | `tags.get("lit") == "yes"` (None이면 None) | bool or None                         |
| `sidewalk` | `tags.get("sidewalk")`                   | 없으면 None                             |

geometry 좌표 순서 주의: Overpass는 `{"lat": y, "lon": x}` 순서로 반환. PostGIS WKT는 `LINESTRING(x y, x y ...)` = `(lon lat, lon lat ...)` 순서.

### Node(횡단보도) 파싱

```python
{
  "type": "node",
  "id": 87654321,
  "lat": 37.55,
  "lon": 126.92,
  "tags": {
    "highway": "crossing",
    "crossing": "traffic_signals",
    "traffic_signals": "yes"
  }
}
```

| 필드            | 소스                                     | 변환               |
| ------------- | -------------------------------------- | ---------------- |
| `osm_id`      | `element["id"]`                        | 그대로              |
| `crossing`    | `tags.get("crossing")`                 | 없으면 None         |
| `has_signals` | `tags.get("traffic_signals") == "yes"` | bool             |
| `geom`        | `lat, lon`                             | `POINT(lon lat)` |

---

## 6. 적재 스크립트 명세 (`loader/main.py`)

### 실행 흐름

```
1. .env 로드 (DB 접속 정보)
2. DB 연결 확인
3. Overpass API 호출 → JSON 응답 수신
   - 타임아웃: 300초
   - 실패 시 3회 재시도 (exponential backoff)
4. 응답 파싱 → way 리스트, crossing 리스트 분리
5. geometry가 없는(좌표 배열 길이 < 2) way 제거
6. osm_ways 일괄 INSERT
7. osm_crossings 일괄 INSERT
8. 적재 건수 로그 출력
```

### INSERT 전략

- 충돌 처리: `ON CONFLICT (osm_id) DO NOTHING`
- 배치 크기: 500건씩 executemany

### 환경변수 (`.env.example`)

```
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=running
POSTGRES_USER=root
POSTGRES_PASSWORD=ssafy1234!
```

실제 `.env` 파일은 `.env.example`을 복사해서 만든다. 운영 서버 접속 시에는 host만 원격 서버 IP로 변경한다.

### 실행 방법

```bash
# 1. 의존성 설치
pip install -r requirements.txt

# 2. .env 파일 생성
cp .env.example .env

# 3. DDL 실행 (컨테이너가 기동 중이어야 함)
docker exec -i localPostgis psql -U root -d running < sql/01_extensions.sql
docker exec -i localPostgis psql -U root -d running < sql/02_schema.sql

# 4. 적재 실행
python -m loader.main
```

### `requirements.txt`

```
psycopg2-binary>=2.9
requests>=2.31
python-dotenv>=1.0
```

---

## 7. 적재 검증 (`sql/03_verify.sql`)

적재 완료 후 아래 쿼리로 결과를 확인한다.

```sql
-- 1. 총 적재 건수
SELECT
    'osm_ways'     AS tbl, COUNT(*) AS cnt FROM osm_ways
UNION ALL
SELECT
    'osm_crossings', COUNT(*) FROM osm_crossings;

-- 2. highway 종류별 분포
SELECT highway, COUNT(*) AS cnt
FROM osm_ways
GROUP BY highway
ORDER BY cnt DESC;

-- 3. 공간 인덱스 동작 확인 (마포구 중심 1km 반경 내 way)
-- 상암동 월드컵공원 근처 좌표 기준
EXPLAIN ANALYZE
SELECT osm_id, highway, name
FROM osm_ways
WHERE ST_DWithin(
    geom::geography,
    ST_SetSRID(ST_MakePoint(126.8875, 37.5683), 4326)::geography,
    1000  -- 미터
);

-- 4. 횡단보도와 500m 내 way 연결 확인 (연결성 검증 기본 동작)
SELECT c.osm_id AS crossing_id, w.osm_id AS way_id, w.highway
FROM osm_crossings c
JOIN osm_ways w
  ON ST_DWithin(c.geom::geography, w.geom::geography, 5)
LIMIT 10;
```

---

## 8. 완료 기준

아래 조건을 모두 만족하면 이 작업 단위 완료로 간주한다.

- [ ] `localPostgis` 컨테이너 기동 중 확인 (`docker ps`)
- [ ] `sql/01_extensions.sql`, `sql/02_schema.sql` 수동 실행 후 `osm_ways`, `osm_crossings` 테이블 생성 확인
- [ ] `python -m loader.main` 실행 시 오류 없이 종료
- [ ] `osm_ways` 적재 건수 > 5,000 (마포구 기준 예상치)
- [ ] `osm_crossings` 적재 건수 > 100
- [ ] `sql/03_verify.sql` 3번 쿼리에서 `EXPLAIN` 결과에 `Index Scan` 또는 `Bitmap Index Scan` 확인 (풀스캔 아닐 것)
- [ ] geometry 좌표 순서 오류 없음: `ST_IsValid(geom) = true` 인 행만 적재됐는지 확인

---

## 9. Worklog 작성 규칙

커밋 단위 작업이 끝날 때마다 `be/worklog/` 디렉토리 안에 아래 형식으로 항목을 추가한다.
파일명은 `YYYY-MM-DD_작업제목.md` 형식으로 생성한다. (예: `2026-04-27_OSM_스키마_생성.md`)
목적은 두 가지다: **다음 세션의 Claude Code가 worklog + 작업지시만 읽고 작업을 이어갈 수 있을 것**, **팀 개발자가 무슨 작업을 했는지 파악할 수 있을 것**.

### 9-1. 항목 형식

```markdown
## [YYYY-MM-DD] <작업 제목>

### 작업 범위
- 어떤 파일을 만들었거나 수정했는지 목록 (파일 경로 포함)
- 예: `loader/overpass_client.py` 신규 생성

### 구현 내용
변경 사항을 서술한다. 단순 나열이 아니라 **왜 이렇게 했는지** 이유까지 쓴다.

### 주요 함수 / 클래스

함수나 클래스를 새로 만들었다면 아래 형식으로 전부 기록한다.

#### `함수명(param1: 타입, param2: 타입) -> 리턴 타입`
- **위치:** `파일경로.py`
- **역할:** 한 줄 설명
- **파라미터:**
  - `param1`: 설명
  - `param2`: 설명
- **리턴:** 설명
- **주의사항 / 사이드이펙트:** (있을 때만)

### DB 변경 사항

테이블 생성·수정·삭제가 있었다면 아래 형식으로 기록한다.

#### 테이블명
| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| `컬럼명` | `타입` | PK / NOT NULL / ... | 용도 설명 |

- **인덱스:** 어떤 컬럼에 어떤 인덱스를 걸었는지, 이유
- **연관관계:** 다른 테이블과의 관계 (FK 또는 논리적 연결)
- **주의사항:** 설계 의도, 나중에 바꿀 가능성이 있는 부분

### 미결 사항 / 다음 세션 이어받기
- 다음에 해야 할 작업을 구체적으로 적는다
- 막힌 부분이나 결정을 미룬 사항도 여기에 기록
- 예: "Overpass 응답에서 geometry가 null인 케이스 발견 → parser.py 예외처리 추가 필요"

### 확인된 이슈 / 결정 사항
- 작업 중 발견한 문제와 그 해결 방법
- 여러 선택지 중 하나를 고른 경우 선택 이유
```

### 9-2. 작성 수준 기준

| 기록 대상     | 작성 수준              |
| --------- | ------------------ |
| 새 함수/메서드  | 파라미터·리턴·주의사항 전부    |
| 기존 함수 수정  | 변경된 파라미터·동작만       |
| 새 테이블/컬럼  | 전체 컬럼 + 인덱스 + 연관관계 |
| 기존 스키마 수정 | 변경된 컬럼·인덱스만        |
| 설정 파일 변경  | 추가된 환경변수와 기본값      |
| 버그 수정     | 원인·증상·해결 방법 3줄 이상  |

### 9-3. 금지 사항

- "기능 구현 완료" 같은 내용 없는 한 줄 요약만 쓰는 것
- 함수를 만들고 파라미터 설명을 생략하는 것
- DB 컬럼을 추가하고 용도 설명을 생략하는 것
- 결정을 미뤘거나 임시 처리한 부분을 기록하지 않는 것

---

## 10. 이 작업에서 명시적으로 하지 않는 것

| 항목                              | 이유                      |
| ------------------------------- | ----------------------- |
| 노드-엣지 그래프 테이블 (pgRouting)       | MVP 이후 단계. 공간 쿼리로 대체 가능 |
| 주기적 갱신 스케줄러                     | 1회성 적재로 결정              |
| 서울 전체 적재                        | 마포구 검증 후 범위 확장          |
| 기존 `nodes` 테이블 수정               | 적재 완료 후 별도 작업           |
| `route=foot/hiking` relation 적재 | 우선순위 3, MVP 이후          |

## 11. 작업 단위

- 작업은 반드시 **커밋 단위**로 진행한 뒤 멈추고 worklog를 작성한 뒤 나의 학습 및 검토(리뷰)가 완료된 뒤 다음 작업을 진행할 것.
