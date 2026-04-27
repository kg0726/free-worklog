## [2026-04-27] OSM Overpass 클라이언트

### 작업 범위
- `자율/data/osm-loader/loader/overpass_client.py` 신규 생성

### 구현 내용

**왜 별도 모듈로 분리했나:**
- HTTP 호출은 외부 의존(네트워크)이라 실패가 잦은 영역. parser/db 모듈에서 분리하면 재시도·타임아웃·로깅 정책을 한곳에서 관리.
- 추후 Overpass 외 다른 데이터 소스(예: 국가공간정보포털)를 추가할 때 동일 인터페이스로 모듈을 늘리기 쉬움.

**왜 QUERY를 모듈 상수로 하드코딩했나:**
- 작업지시 4번이 쿼리를 명시하고 있고, MVP는 마포구 1회성 적재라 외부 설정 분리 가치가 낮음.
- 추후 지역 확장 시 함수 인자로 빼는 게 자연스러움 (예: `fetch_overpass(area_name="강남구")`). 지금은 YAGNI.

**왜 User-Agent를 추가했나 (작업지시에 없는 추가 사항):**
- 첫 시도에서 `406 Not Acceptable`로 실패. Overpass 공개 서버는 기본 `python-requests/x.x` User-Agent 트래픽을 abuse 방지 목적으로 거부.
- `User-Agent: osm-loader/1.0 (running-app data ingestion)` 형식으로 자기 식별을 추가하면 서버가 정상 응답.
- 일반적인 Overpass 사용 매너이기도 하고, 서버 운영자가 abuse를 추적할 수 있는 수단(연락처/식별자)을 제공하는 게 공유 인프라 사용 예의.

**왜 지수 백오프를 1초부터 시작했나:**
- Overpass 공개 서버는 동시 접속 제한이 있어 일시적 503·429가 흔함. 첫 재시도까지 너무 짧으면 같은 부하를 다시 던지게 됨.
- 1s → 2s 백오프는 서버에 회복 시간을 주면서도, 일시적 네트워크 끊김(2초 미만) 정도라면 두 번째 시도에서 복구 가능.
- 마지막 시도(3회차) 실패 후엔 sleep 없이 즉시 raise — 사용자가 더 기다리는 의미가 없음.

**왜 RequestException + ValueError를 함께 잡았나:**
- `RequestException`: 네트워크 단절·타임아웃·HTTP 에러 (raise_for_status 포함)
- `ValueError`: response.json() 파싱 실패 (응답 body가 JSON이 아님 — 예: 서버 점검 페이지 HTML 반환)
- 둘 다 외부 의존 실패라 재시도 가치가 동일.

**왜 마지막 실패 시 RuntimeError로 감쌌나:**
- 호출자(C6 main.py) 입장에선 "재시도 다 했는데 실패"라는 상위 의미만 알면 됨. 마지막 예외(HTTPError 등 raw)를 그대로 raise하면 호출자가 internals를 알아야 함.
- `raise ... from last_exc`로 원인 체인은 보존 → 디버깅 시 원본 에러도 추적 가능.

**왜 main 진입점을 두었나:**
- C5 파서 작성 전에 "응답이 실제로 오고 형식이 예상대로인지" 단독 확인이 필요. 매번 main.py 거쳐 검증하면 dependency가 늘어남.
- `python -m loader.overpass_client`로 way/node 수 출력 → 작업지시 4번 쿼리가 의도한 데이터를 가져왔는지 즉시 검증.

### 주요 함수 / 클래스

#### `fetch_overpass(timeout: int = 300, max_retries: int = 3) -> dict`
- **위치:** `loader/overpass_client.py`
- **역할:** Overpass API에 마포구 보행 가능 way + crossing node 쿼리를 POST하고 응답을 dict로 반환.
- **파라미터:**
  - `timeout`: 단일 요청 타임아웃(초). 기본 300. 마포구 규모 응답은 보통 30~60초.
  - `max_retries`: 총 시도 횟수. 기본 3. 마지막 시도 실패 시 RuntimeError raise.
- **리턴:** Overpass JSON 응답 전체 dict. `data["elements"]`에 way/node 배열.
- **주의사항 / 사이드이펙트:**
  - 외부 네트워크 호출 (https://overpass-api.de). 서버 점검 시간(매주 화·목 새벽)에는 다운될 수 있음.
  - User-Agent 헤더 자동 부착 (`osm-loader/1.0 (running-app data ingestion)`).
  - 재시도 정책: 첫 시도 실패 → 1s 대기 → 2번째 → 2s 대기 → 3번째. 마지막 시도 실패 시 sleep 없이 raise.
  - 메모리 사용량: 마포구 응답이 약 6,600 elements. 응답 JSON 전체를 dict로 들고 있음 (수십 MB 수준).

### 모듈 상수

| 이름 | 값 | 설명 |
|------|----|------|
| `OVERPASS_ENDPOINT` | `https://overpass-api.de/api/interpreter` | 공개 Overpass 서버. 무료, 동시 접속 제한 있음. |
| `USER_AGENT` | `osm-loader/1.0 (running-app data ingestion)` | 자기 식별. 서버 abuse 방지 정책 회피용. |
| `QUERY` | (작업지시 4번 그대로) | 마포구 area + highway 필터 + crossing node |

### DB 변경 사항
없음.

### 검증 결과 (실 호출)

```
$ python -m loader.overpass_client
[INFO] Overpass 요청 시도 1/3
[INFO] Overpass 응답 수신: elements=6627
total elements: 6627
  ways: 6290
  nodes: 337
```

- 응답 수신 시간: 약 11초
- ways 6,290 (작업지시 완료기준 5,000+ 충족 전망)
- nodes 337 (완료기준 100+ 충족 전망)
- 추후 파서에서 geometry < 2 way 제거 시 일부 감소 가능, 그래도 5,000 여유 확보.

### 미결 사항 / 다음 세션 이어받기
- **다음 커밋 C5**: 파서 작성
  - `loader/parser.py`: `parse_way`, `parse_node`, `parse_response`
  - way의 14개 필드 추출, geometry < 2개 좌표는 None 반환
  - 좌표 순서 변환: `[{"lat":y,"lon":x},...]` → `"LINESTRING(x y, ...)"` (lon lat 순)
  - node의 4개 필드 추출, `"POINT(lon lat)"`
- **샘플 응답**: 검증 시 받은 응답을 별도 파일로 저장하지 않음 (매번 fetch). 파서 sanity check도 실 호출로 진행 예정.

### 확인된 이슈 / 결정 사항

#### 이슈: 첫 시도 시 406 Not Acceptable
- **증상**: User-Agent 없이 POST 시 Overpass 서버가 즉시 (1초 미만) 406 응답.
- **원인**: Overpass 공개 서버는 abuse 방지 목적으로 기본 `python-requests/x.x` UA 트래픽을 차단.
- **해결**: `User-Agent` 헤더에 모듈 식별자 명시 (`osm-loader/1.0 (running-app data ingestion)`). 1차 시도부터 정상 응답.
- **교훈**: 공유 공개 API는 자기 식별 UA가 사실상 매너 + 필수. 사내 사용자 식별이 필요한 API라면 더더욱.

#### 결정: 재시도 시 sleep 정책
- 작업지시 6번 "3회 재시도 (exponential backoff)"를 max_retries=3 (총 3번 시도)로 해석.
- 시도 1 실패 후 1s, 시도 2 실패 후 2s sleep. 마지막 시도(3) 실패 시 sleep 없이 즉시 raise. (불필요한 마지막 sleep 제거)
- 작업계획서에 적은 "1s → 2s → 4s"는 부정확한 표현이었음 — 마지막 sleep은 의미 없어 제거.

### 사용자 확인 후 git add/commit 안내
변경 검토 후 직접:
```bash
git add data/osm-loader/loader/overpass_client.py
git commit -m "feat: Overpass API 클라이언트 (지수 백오프 재시도, UA 헤더)"
```
