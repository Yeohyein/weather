# PRD — World Weather Explorer

| 항목 | 내용 |
|---|---|
| 문서 버전 | v0.3 |
| 작성일 | 2026-08-18 |
| 상태 | 개발 착수 가능 |
| 산출물 | 단일 `index.html` — Tailwind CDN + Leaflet.js(OpenStreetMap 타일) + 바닐라 JS, 빌드 도구 없음 |
| 외부 의존성 | Open-Meteo 공개 API(인증 불필요), Tailwind CDN, Leaflet.js CDN + OpenStreetMap 타일 서버, Wikimedia Commons 랜드마크 사진(외부 URL 참조, 파일 내장 없음) |

---

## 1. 개요

세계 각지의 랜드마크가 실사 사진 마커로 표시된 **인터랙티브 세계지도**에서 랜드마크를 클릭하면, 그 위치의 현재 날씨 · 현지 시각 · 7일 예보를 보여주는 단일 페이지 웹앱. 화면 배경은 조회된 날씨와 낮/밤에 따라 변한다. 지도에 없는 도시는 검색으로 찾는다.

### 1.1 문제 정의

기존 날씨 서비스는 "내 위치의 날씨"에 최적화되어 있다. 이 제품은 **"저기는 지금 어떨까"** 라는 호기심에 답한다. 여행 계획, 해외 지인과의 대화, 단순 탐색을 상정한다.

### 1.2 핵심 컨셉

메인 화면이 폼(form)이 아니라 **실제 세계지도**다. 사용자는 검색어를 떠올리기 전에 이미 지구본을 마주한다. 지도 위 12개 랜드마크는 원형 실사 사진 마커로 표시되고, 마커마다 현재 날씨 배지가 떠 있어서 클릭하기 전에도 "지금 파리는 비가 오는구나"를 알 수 있다. 지도는 실제 지리 좌표 위에 놓이므로 사용자는 확대·축소·드래그로 자유롭게 탐색할 수 있다.

### 1.3 성공 기준

| 지표 | 목표 |
|---|---|
| 첫 랜드마크 조회까지 | 클릭 1회, 진입 후 3초 이내 |
| 상태 화면 구현 | 5종(초기/로딩/빈결과/에러/성공) 전부 |
| 반응형 | 360px ~ 1440px 레이아웃 무결 |
| API 호출량 | 초기 진입 시 2회 이하 (날씨 프리페치 1회 + 필요 시 1회, 지도 타일·사진 요청은 제외) |

---

## 2. 범위

**포함** — 인터랙티브 세계지도(Leaflet + OpenStreetMap) 위 실사 랜드마크 사진 마커, 지도 확대·축소·드래그, 3축 필터, 도시 검색(검색 시 지도 이동), 현재 날씨 상세, 7일 예보, 현지 시각 실시간 표시, 날씨/시간 연동 배경 전환(UI 크롬 한정), 5종 상태 화면

**제외** — 사용자 계정, 서버 저장, 시간별(hourly) 예보, 오프라인 지도 타일 캐싱, 푸시 알림, 다국어 UI(한국어 고정), 과거 날씨 이력

---

## 3. 정보 구조 & 상태 흐름

### 3.1 화면 계층

```
┌──────────────────────────────────────────────┐
│ 배경 레이어 (z-0) — 날씨×낮밤 그라디언트       │
│   (지도 타일 뒤쪽 여백·헤더·패널에만 적용)     │
├──────────────────────────────────────────────┤
│ 헤더 (z-30) — 타이틀 · 검색창 · 필터 토글      │
│   └ 검색 후보 드롭다운 (z-40)                  │
├──────────────────────────────────────────────┤
│ 필터 바 (z-20) — 3축 칩 그룹, 접이식           │
├──────────────────────────────────────────────┤
│ 메인 (z-10) — 인터랙티브 세계지도(Leaflet)     │
│   OpenStreetMap 타일 + 랜드마크 사진 마커      │
├──────────────────────────────────────────────┤
│ 디테일 패널 (z-50) — 지도 위에 겹쳐 등장       │
│   데스크톱: 우측 사이드 패널 (440px)           │
│   모바일:   하단 바텀시트 (높이 85vh)          │
└──────────────────────────────────────────────┘
```

디테일 패널이 지도를 **대체하지 않고 덮는** 이유: 지도가 배경에 남아 있어야 "다른 랜드마크도 눌러볼 수 있다"는 게 계속 보이고, 패널을 닫으면 즉시 탐색(팬·줌)으로 복귀한다. 지도 타일은 실제 위성/도로 이미지이므로 날씨에 따라 색이 바뀌지 않는다 — 배경 그라디언트는 헤더·필터바·패널 등 지도 밖 UI 크롬에만 적용된다.

### 3.2 앱 상태 머신

| 상태 | 진입 조건 | 화면 |
|---|---|---|
| `PREFETCHING` | 최초 진입 | 지도는 렌더되되 랜드마크 마커 배지 자리에 스켈레톤 |
| `SCENE_READY` | 프리페치 성공 | 지도 + 사진 마커 + 날씨 배지 + 필터 활성화 |
| `SCENE_DEGRADED` | 프리페치 실패 | 지도·마커는 표시, 배지 없음, 필터 비활성 + 재시도 배너 |
| `DETAIL_LOADING` | 랜드마크/도시 선택 | 패널 등장 + 스켈레톤 |
| `DETAIL_READY` | 예보 수신 | 패널에 현재 날씨 + 7일 예보 |
| `DETAIL_ERROR` | 예보 조회 실패 | 패널에 에러 + 재시도 |
| `SEARCH_EMPTY` | 지오코딩 결과 0건 | 드롭다운 자리에 빈 결과 안내 |

상태는 단일 전역 객체로 관리한다.

```js
const state = {
  phase: 'PREFETCHING',
  landmarks: [],        // Landmark[] — 정적 정의 + 프리페치 결과 병합
  filters: { weather: [], region: [], daynight: [] },
  selected: null,       // Landmark | GeoResult | null
  detail: null,         // ForecastData | null
  error: null,          // { kind, message } | null
};
```

---

## 4. 기능 요구사항

### FR-01 · 인터랙티브 세계지도

메인 화면은 Leaflet.js로 렌더링한 실제 세계지도(OpenStreetMap 타일)다. 12개 랜드마크는 지도 위 실제 위경도 좌표에 원형 실사 사진 마커로 표시된다.

| ID | 요구사항 |
|---|---|
| FR-01-1 | Leaflet 지도, 초기 뷰는 12개 랜드마크가 모두 보이도록 `fitBounds`(또는 `center:[20,10], zoom:2` 근사). 타일 레이어는 OpenStreetMap 표준 타일(`{s}.tile.openstreetmap.org`) |
| FR-01-2 | 마커는 Leaflet `divIcon`으로 구현한 **원형 사진 핀** — 배경에 랜드마크 실사 사진(Wikimedia Commons, `object-fit:cover`로 원형 크롭), 테두리는 글래스 스트로크 |
| FR-01-3 | 사진은 실제 랜드마크를 정확히 식별할 수 있는 구도여야 한다. 저작권 상 자유 이용 라이선스(Wikimedia Commons) 이미지만 사용 |
| FR-01-4 | 각 마커 DOM에는 `role="button" tabindex="0" aria-label="에펠탑, 파리, 현재 15도 맑음"` 부여 — Leaflet 마커는 기본적으로 키보드 포커스를 받지 않으므로 커스텀 `divIcon` DOM에 직접 부여 |
| FR-01-5 | 마커 히트 영역(원형 핀 전체)이 최소 44×44px 상당을 보장하도록 CSS 크기 지정 |
| FR-01-6 | 마커 상단에 **날씨 배지** — 이모지 + 기온. 프리페치 완료 시 페이드인 |
| FR-01-7 | hover/focus 시: 마커 확대 + 테두리 강조 + 라벨(랜드마크명/도시명) 툴팁 표시 |
| FR-01-8 | 클릭/Enter/Space → `DETAIL_LOADING` 진입 |
| FR-01-9 | Tab 순서는 `LANDMARKS` 배열 순서(경도 오름차순, 서→동). 방향키 ←→ 로도 마커 간 이동 가능 |
| FR-01-10 | 지도 타일은 실사 이미지이므로 낮/밤에 따라 색이 바뀌지 않는다. 대신 지도 위에 반투명 날씨×낮밤 그라디언트 오버레이 레이어(pointer-events 없음)를 얹어 톤을 맞춘다 |
| FR-01-11 | 사용자는 표준 지도 조작(휠/핀치 줌, 드래그 팬)이 가능하다. 랜드마크가 없는 지역을 탐색해도 무방하다 — 지도는 실제 지리이므로 "빈 배경"이 아니다 |

**반응형 처리**

| 뷰포트 | 처리 |
|---|---|
| ≥1280px | 지도가 헤더/필터바를 제외한 전체 영역을 채움 |
| 768–1279px | 동일하게 전체 영역, 라벨 툴팁은 hover 시에만 |
| <768px | 지도가 뷰포트 전체 폭을 채우고 네이티브 터치 팬/핀치줌 사용. 별도의 가로 스크롤 컨테이너는 불필요(지도 자체가 팬 가능) |

**모션 제약** — `prefers-reduced-motion: reduce` 인 경우 마커 확대·배지 페이드·지도 `flyTo` 애니메이션을 전부 즉시 전환(`setView`)으로 대체.

### FR-02 · 랜드마크 프리셋 (12개)

```js
const LANDMARKS = [
  { id:'liberty',  name:'자유의 여신상', city:'뉴욕',        country:'미국',      region:'북미',       lat: 40.6892, lon: -74.0445,
    photo:'https://upload.wikimedia.org/wikipedia/commons/thumb/8/89/Front_view_of_Statue_of_Liberty_%28cropped%29.jpg/960px-Front_view_of_Statue_of_Liberty_%28cropped%29.jpg' },
  { id:'canyon',   name:'그랜드캐니언',  city:'애리조나',     country:'미국',      region:'북미',       lat: 36.1069, lon:-112.1129,
    photo:'https://upload.wikimedia.org/wikipedia/commons/thumb/3/31/Canyon_River_Tree_%28165872763%29.jpeg/960px-Canyon_River_Tree_%28165872763%29.jpeg' },
  { id:'machu',    name:'마추픽추',      city:'쿠스코',       country:'페루',      region:'남미',       lat:-13.1631, lon: -72.5450,
    photo:'https://upload.wikimedia.org/wikipedia/commons/thumb/b/bb/Machu_Picchu%2C_2023_%28012%29.jpg/960px-Machu_Picchu%2C_2023_%28012%29.jpg' },
  { id:'christ',   name:'예수상',        city:'리우데자네이루',country:'브라질',    region:'남미',       lat:-22.9519, lon: -43.2105,
    photo:'https://upload.wikimedia.org/wikipedia/commons/thumb/4/4f/Christ_the_Redeemer_-_Cristo_Redentor.jpg/960px-Christ_the_Redeemer_-_Cristo_Redentor.jpg' },
  { id:'bigben',   name:'빅벤',          city:'런던',         country:'영국',      region:'유럽',       lat: 51.5007, lon:  -0.1246,
    photo:'https://upload.wikimedia.org/wikipedia/commons/thumb/0/05/Elizabeth_Tower_and_the_north_front_of_the_Palace_of_Westminster%2C_London.jpg/960px-Elizabeth_Tower_and_the_north_front_of_the_Palace_of_Westminster%2C_London.jpg' },
  { id:'eiffel',   name:'에펠탑',        city:'파리',         country:'프랑스',    region:'유럽',       lat: 48.8584, lon:   2.2945,
    photo:'https://upload.wikimedia.org/wikipedia/commons/thumb/8/85/Tour_Eiffel_Wikimedia_Commons_%28cropped%29.jpg/960px-Tour_Eiffel_Wikimedia_Commons_%28cropped%29.jpg' },
  { id:'colosseo', name:'콜로세움',      city:'로마',         country:'이탈리아',  region:'유럽',       lat: 41.8902, lon:  12.4922,
    photo:'https://upload.wikimedia.org/wikipedia/commons/thumb/d/de/Colosseo_2020.jpg/960px-Colosseo_2020.jpg' },
  { id:'pyramid',  name:'기자 피라미드', city:'카이로',       country:'이집트',    region:'아프리카',   lat: 29.9792, lon:  31.1342,
    photo:'https://upload.wikimedia.org/wikipedia/commons/thumb/e/e7/Great_Pyramid_of_Giza_-_Pyramid_of_Khufu.jpg/960px-Great_Pyramid_of_Giza_-_Pyramid_of_Khufu.jpg' },
  { id:'taj',      name:'타지마할',      city:'아그라',       country:'인도',      region:'아시아',     lat: 27.1751, lon:  78.0421,
    photo:'https://upload.wikimedia.org/wikipedia/commons/thumb/1/1d/Taj_Mahal_%28Edited%29.jpeg/960px-Taj_Mahal_%28Edited%29.jpeg' },
  { id:'gyeongbok',name:'경복궁',        city:'서울',         country:'대한민국',  region:'아시아',     lat: 37.5796, lon: 126.9770,
    photo:'https://upload.wikimedia.org/wikipedia/commons/thumb/6/63/%EA%B4%91%ED%99%94%EB%AC%B8_%EC%9B%94%EB%8C%80.jpg/960px-%EA%B4%91%ED%99%94%EB%AC%B8_%EC%9B%94%EB%8C%80.jpg' },
  { id:'fuji',     name:'후지산',        city:'시즈오카',     country:'일본',      region:'아시아',     lat: 35.3606, lon: 138.7274,
    photo:'https://upload.wikimedia.org/wikipedia/commons/thumb/f/f8/View_of_Mount_Fuji_from_%C5%8Cwakudani_20211202.jpg/960px-View_of_Mount_Fuji_from_%C5%8Cwakudani_20211202.jpg' },
  { id:'opera',    name:'오페라하우스',  city:'시드니',       country:'호주',      region:'오세아니아', lat:-33.8568, lon: 151.2153,
    photo:'https://upload.wikimedia.org/wikipedia/commons/thumb/a/a0/Sydney_Australia._%2821339175489%29.jpg/960px-Sydney_Australia._%2821339175489%29.jpg' },
];
```

`photo`는 Wikimedia Commons의 자유 이용 라이선스 이미지를 MediaWiki API(`action=query&prop=pageimages&pithumbsize=640`)로 조회해 검증한 URL이다. 배열 순서는 경도 오름차순(서→동)이며, 지도 위 실제 위경도 좌표에 그대로 매핑되므로 씬처럼 "배치 순서"를 따로 관리할 필요는 없다 — Tab 순서 등 UI 상 순회에만 사용한다.

### FR-03 · 날씨 프리페치 (필터의 전제 조건)

날씨/낮밤 필터가 동작하려면 12곳의 현재 날씨를 **미리** 알아야 한다. Open-Meteo는 좌표를 콤마로 이어 붙이면 한 번의 요청으로 여러 지점을 반환한다.

```
GET https://api.open-meteo.com/v1/forecast
  ?latitude=40.6892,36.1069,-13.1631,...      ← 12개
  &longitude=-74.0445,-112.1129,-72.5450,...  ← 12개
  &current=temperature_2m,weather_code,is_day
  &timezone=auto
```

| ID | 요구사항 |
|---|---|
| FR-03-1 | 진입 시 1회만 호출. 응답은 **좌표 순서와 동일한 배열**로 오므로 인덱스로 매핑 |
| FR-03-2 | 다중 좌표 응답은 최상위가 객체가 아닌 **배열**이다. 단일 좌표 응답과 파싱 분기 필요 |
| FR-03-3 | 결과를 `sessionStorage`에 10분 TTL로 캐시 |
| FR-03-4 | 실패 시 `SCENE_DEGRADED` — 지도·마커는 그대로 두고 배지만 생략, 필터는 비활성화하고 "날씨 정보를 불러오지 못해 필터를 사용할 수 없습니다" 배너 + 재시도 |
| FR-03-5 | 프리페치 데이터는 **배지·필터 전용**. 랜드마크 클릭 시에는 7일 예보를 포함한 상세 요청을 별도로 보낸다 |

### FR-04 · 필터 (3축, 복수 선택)

| 축 | 옵션 | 판정 기준 |
|---|---|---|
| **날씨 상태** | ☀️ 맑음 · ☁️ 흐림 · 🌧️ 비 · 🌨️ 눈 · ⛈️ 뇌우 · 🌫️ 안개 | `weather_code` 그룹핑 (FR-08) |
| **지역** | 아시아 · 유럽 · 북미 · 남미 · 아프리카 · 오세아니아 | `LANDMARKS[].region` |
| **낮/밤** | ☀️ 낮 · 🌙 밤 | `current.is_day` (1/0) — 각 랜드마크의 **현지** 기준 |

**결합 규칙** — 축 내부는 OR, 축 간에는 AND.
예: `날씨=[비, 눈] · 지역=[유럽]` → 유럽에 있으면서 (비 또는 눈)인 랜드마크.
어떤 축이든 선택 0개면 그 축은 조건 없음으로 취급한다.

| ID | 요구사항 |
|---|---|
| FR-04-1 | 필터에서 제외된 랜드마크 마커는 **사라지지 않고** `opacity: 0.2` + `grayscale(1)` + `pointer-events: none` + `tabindex="-1"` |
| FR-04-2 | 마커는 실제 위경도에 고정되므로 위치가 바뀔 일은 없다. 디밍은 순수하게 "지금 조건에 안 맞는다"는 시각적 신호일 뿐, 지도 자체의 지리 정보는 그대로 유지된다 |
| FR-04-3 | 필터 바에 "12곳 중 N곳" 카운터 상시 표시 |
| FR-04-4 | 결과 0곳이면 지도 위에 오버레이 — "조건에 맞는 곳이 없습니다" + **필터 초기화** 버튼 |
| FR-04-5 | 활성 필터 칩은 개별 X로 해제 가능. "전체 해제" 버튼 별도 제공 |
| FR-04-6 | 필터 상태는 URL 쿼리에 반영 (`?w=rain,snow&r=europe&d=night`) — 새로고침·공유 시 유지 |
| FR-04-7 | 모바일에서는 필터 바가 기본 접힘. 헤더의 필터 버튼에 활성 개수 배지 표시 |

### FR-05 · 도시 검색

| ID | 요구사항 |
|---|---|
| FR-05-1 | 2글자 이상 입력 시 300ms 디바운스 후 지오코딩 호출 |
| FR-05-2 | 최대 6개 후보. 각 항목은 `국기 이모지 · 도시명 · admin1 · 국가` |
| FR-05-3 | 후보 클릭 / Enter(하이라이트된 항목) → `DETAIL_LOADING` |
| FR-05-4 | ↑↓ 이동, Esc 닫기, 포커스 이탈 시 닫기 |
| FR-05-5 | 진행 중 요청은 `AbortController`로 취소 |
| FR-05-6 | 결과 0건 → 드롭다운 자리에 빈 결과 안내 (FR-09) |
| FR-05-7 | 후보 선택 시 지도가 해당 좌표로 `flyTo`(reduced-motion 시 `setView`)하고 **임시 마커**(사진 없는 핀 아이콘)를 표시한다. 임시 마커는 12개 랜드마크 프리셋에 추가되지 않으며 필터 대상도 아니다. 패널을 닫거나 다른 곳을 선택하면 임시 마커는 제거된다 |

> **제약 — 한글 검색.** Open-Meteo 지오코딩은 한글 질의 지원이 불완전하다. 주요 도시(서울/도쿄/파리/런던 등)는 매칭되지만 중소도시는 영문명으로만 잡히는 경우가 많다. 결과 0건 시 **"영문 이름으로도 검색해보세요"** 안내를 반드시 노출한다.

### FR-06 · 디테일 패널 — 현재 날씨

| 항목 | 형식 |
|---|---|
| 랜드마크/도시명 | 국가명 병기 |
| **현지 시각** | `HH:MM` — 1초마다 갱신. 요일·날짜 병기 |
| **기온** | 정수 + `°C`. 패널 내 최대 활자 |
| 날씨 | 이모지 + 한국어 설명 |
| 체감 / 습도 / 강수 / 풍속 | 2×2 서브 그리드 |
| 닫기 | 우상단 X, Esc 키, 패널 외부 클릭, 모바일은 아래로 스와이프 |

**현지 시각 계산** — 브라우저 타임존이 이중 적용되지 않도록 주의한다.

```js
const localMs = Date.now() + utcOffsetSeconds * 1000;
const d = new Date(localMs);
const hh = String(d.getUTCHours()).padStart(2, '0');   // getHours() 아님
const mm = String(d.getUTCMinutes()).padStart(2, '0');
```

### FR-07 · 디테일 패널 — 7일 예보

| ID | 요구사항 |
|---|---|
| FR-07-1 | 가로 스크롤 카드 리스트. 데스크톱 패널 내에서도 가로 스크롤(패널 폭 440px 기준) |
| FR-07-2 | 카드 구성 — 요일(당일은 "오늘"), `M/D`, 이모지, 최고/최저, 강수확률 |
| FR-07-3 | 최고/최저를 7일 전체 범위에 정규화한 막대로 시각화해 주간 추이를 드러낸다 |
| FR-07-4 | `scroll-snap-align: start` 로 카드 단위 스냅 |

### FR-08 · weather_code 매핑

| 코드 | 이모지 | 한국어 | 필터 그룹 |
|---|---|---|---|
| 0 | ☀️ | 맑음 | clear |
| 1 | 🌤️ | 대체로 맑음 | clear |
| 2 | ⛅ | 구름 조금 | cloudy |
| 3 | ☁️ | 흐림 | cloudy |
| 45 | 🌫️ | 안개 | fog |
| 48 | 🌫️ | 서리 안개 | fog |
| 51 / 53 / 55 | 🌦️ | 약한 / 보통 / 강한 이슬비 | rain |
| 56 / 57 | 🌧️ | 어는 이슬비 | rain |
| 61 / 63 / 65 | 🌧️ | 약한 비 / 비 / 강한 비 | rain |
| 66 / 67 | 🌧️ | 어는 비 | rain |
| 71 / 73 / 75 | 🌨️ | 약한 눈 / 눈 / 강한 눈 | snow |
| 77 | 🌨️ | 싸락눈 | snow |
| 80 / 81 / 82 | 🌧️ | 소나기 / 강한 소나기 / 폭우 | rain |
| 85 / 86 | ❄️ | 소낙눈 | snow |
| 95 | ⛈️ | 뇌우 | storm |
| 96 / 99 | ⛈️ | 우박 동반 뇌우 | storm |

미정의 코드 폴백 → `🌡️ 정보 없음`, 그룹은 `unknown` (어떤 날씨 필터에도 걸리지 않음).

### FR-09 · 배경 전환

배경은 **선택된 장소** 기준이다. 선택 전에는 사용자 로컬 시각 기준 기본 테마. 지도 타일은 실사 이미지라 색이 바뀌지 않으므로, 이 그라디언트는 (a) 헤더·필터바·디테일 패널 등 지도 밖 UI 크롬과 (b) 지도 위에 얹는 반투명 오버레이 레이어(`pointer-events:none`, 낮은 알파)에 적용해 톤만 맞춘다.

| 그룹 | 낮 | 밤 |
|---|---|---|
| clear | 하늘색 → 연노랑 | 남색 → 자정색 |
| cloudy | 회청색 → 은색 | 짙은 회색 → 검정 |
| fog | 회백색 저채도 | 어두운 회색 |
| rain | 청회색 → 짙은 파랑 | 검푸른색 |
| snow | 연회색 → 흰색 | 청회색 → 남색 |
| storm | 짙은 보라 → 검정 | 검정 → 짙은 보라 |

- 전환: 0.8s `ease-in-out` 페이드 (`prefers-reduced-motion` 시 즉시 전환)
- 배경 밝기에 따라 텍스트 색을 명/암 두 세트로 자동 전환, 대비 4.5:1 이상 확보
- 구체적 색상값은 `DESIGN.md` 수령 후 확정 (§8 미결)

### FR-10 · 상태 화면

| 상태 | 화면 |
|---|---|
| 초기/프리페치 | 지도 표시 + 마커 배지 자리 스켈레톤 펄스 (스피너 사용하지 않음) |
| 디테일 로딩 | 패널 열림 + 카드 형태 스켈레톤 |
| 빈 결과 | `'{검색어}'에 해당하는 도시를 찾을 수 없습니다` + 팁 3종 (철자 확인 / 영문 시도 / 더 큰 도시명) |
| 필터 0건 | 지도 오버레이 + 필터 초기화 버튼 |
| 에러 | 원인별 메시지 + 재시도 + 닫기 |

**에러 분기**

| 조건 | 메시지 |
|---|---|
| `!navigator.onLine` | 인터넷 연결을 확인해주세요 |
| HTTP 429 | 요청이 많습니다. 잠시 후 다시 시도해주세요 |
| HTTP 5xx / `AbortError`(타임아웃) | 날씨 정보를 불러오지 못했습니다 |
| JSON 파싱 실패 / 기타 | 알 수 없는 오류가 발생했습니다 |

---

## 5. API 명세

인증 불필요. 전부 클라이언트에서 직접 `fetch`. 공통 정책: **타임아웃 8초**(`AbortController`+`setTimeout`), **자동 재시도 없음**(사용자가 버튼을 눌렀을 때만).

### 5.1 지오코딩

```
GET https://geocoding-api.open-meteo.com/v1/search
  ?name={q}&count=6&language=ko&format=json
```

사용 필드: `results[].id · name · latitude · longitude · country · country_code · admin1 · timezone`

> 결과가 없으면 `results` 키가 **아예 생략된다.** 빈 배열이 아니라 `undefined`를 체크해야 한다.
> `country_code`(ISO-3166 alpha-2)는 유니코드 리저널 인디케이터로 변환해 국기 이모지를 만든다.

### 5.2 날씨 프리페치 (다중 좌표)

```
GET https://api.open-meteo.com/v1/forecast
  ?latitude={12개 콤마 구분}&longitude={12개 콤마 구분}
  &current=temperature_2m,weather_code,is_day
  &timezone=auto
```

응답 최상위가 **배열**. 요청한 좌표 순서와 동일하게 정렬되어 돌아온다.

### 5.3 상세 예보 (단일 좌표)

```
GET https://api.open-meteo.com/v1/forecast
  ?latitude={lat}&longitude={lon}
  &current=temperature_2m,apparent_temperature,relative_humidity_2m,
           is_day,precipitation,weather_code,wind_speed_10m
  &daily=weather_code,temperature_2m_max,temperature_2m_min,
         precipitation_probability_max
  &timezone=auto&forecast_days=7
```

사용 필드: `current.*`, `daily.time[] · weather_code[] · temperature_2m_max[] · temperature_2m_min[] · precipitation_probability_max[]`, `utc_offset_seconds`

### 5.4 캐시

| 대상 | 저장소 | TTL |
|---|---|---|
| 날씨 프리페치 | `sessionStorage` | 10분 |
| 상세 예보 | 메모리 `Map` (키: `lat,lon` 소수 4자리) | 10분 |

### 5.5 지도 타일 & 랜드마크 사진

```
지도 타일: https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png   (Leaflet TileLayer가 자동 요청, 앱 코드에서 직접 fetch하지 않음)
랜드마크 사진: FR-02의 photo 필드 URL을 <img src>로 직접 참조 (fetch 불필요, 브라우저가 알아서 로드)
```

OpenStreetMap 타일은 [사용 정책](https://operations.osmfoundation.org/policies/tiles/)상 과도한 트래픽을 금지하지만, 랜드마크 12곳 주변만 탐색하는 일반 사용 패턴에서는 문제되지 않는다. Wikimedia Commons 이미지는 인증 없이 직접 `<img>`로 로드 가능하며, 로드 실패 시 `onerror`로 이모지 플레이스홀더(랜드마크 대표 이모지)로 폴백한다.

---

## 6. 구현 분해

| 순서 | 작업 | 산출 |
|---|---|---|
| 1 | HTML 골격 + Tailwind 설정 + Leaflet CDN 로드 + CSS 변수 정의 | 빈 셸 |
| 2 | 상수 테이블 — `LANDMARKS`(+photo), `WEATHER_CODES`, `BG_THEMES` | 데이터 레이어 |
| 3 | Leaflet 지도 초기화 + OSM 타일 레이어 + 12개 사진 마커(`divIcon`) + 배지 슬롯 | 정적 지도 |
| 4 | API 래퍼 — `fetchJSON(url, {timeout})`, `geocode()`, `prefetchScene()`, `fetchForecast()` | 통신 레이어 |
| 5 | 상태 머신 + 렌더 디스패처 `render(state)` | 코어 |
| 6 | 프리페치 → 배지 렌더 | `SCENE_READY` 도달 |
| 7 | 필터 3축 + URL 동기화 + 마커 디밍 처리 | 필터 완성 |
| 8 | 디테일 패널 — 현재 날씨 + 현지 시계 타이머 | 상세 상반 |
| 9 | 7일 예보 리스트 + 기온 막대 | 상세 하반 |
| 10 | 검색창 + 디바운스 + 후보 드롭다운 + 키보드 조작 + 지도 `flyTo` | 검색 완성 |
| 11 | 상태 화면 5종 + 에러 분기 + 사진 로드 실패 폴백 | 예외 처리 |
| 12 | 반응형 검수 · 접근성 검수(마커 키보드 포커스 포함) · reduced-motion | 마감 |
| 13 | `DESIGN.md` 반영 (색상·타이포·마커 스타일) | 디자인 적용 |

---

## 7. 비기능 요구사항

| 구분 | 요구사항 |
|---|---|
| 배포 | 단일 `index.html`. 외부 리소스는 Tailwind CDN + Leaflet.js CDN + OpenStreetMap 타일 + Wikimedia Commons 랜드마크 사진 12장(외부 URL 참조, 파일 내장 없음). 빌드 도구 없음 |
| 반응형 | 360 / 768 / 1024 / 1440px 검수 |
| 성능 | 첫 렌더 1초 이내, API 응답 후 렌더 200ms 이내. 사진은 lazy load(`loading="lazy"`)로 초기 로드 부담 최소화 |
| 접근성 | 랜드마크 마커 `role="button"` + `aria-label`("에펠탑, 파리, 현재 15도 맑음"), 검색 드롭다운 `role="listbox"`/`option` + `aria-activedescendant`, 패널 `role="dialog"` + 포커스 트랩 + 닫을 때 트리거로 포커스 복귀, 대비 4.5:1 이상, 키보드 전용 조작 완결(Leaflet 기본 마커는 키보드 접근이 안 되므로 커스텀 `divIcon` DOM에 직접 포커스·키보드 이벤트 부여) |
| 모션 | `prefers-reduced-motion: reduce` 대응 |
| 오류 내성 | 응답 필드 누락 시 해당 항목만 `–` 표시, 화면 전체는 유지 |
| 브라우저 | 최신 Chrome / Safari / Firefox / Edge |

---

## 8. 미결 사항

| # | 항목 | 상태 |
|---|---|---|
| 1 | `DESIGN.md` 수령·반영 완료(글래스모피즘) — 단, 마커가 실루엣→사진 핀으로 바뀌며 §4.3(랜드마크 유리)은 사진 핀 스타일로 재정의 필요 | **DESIGN.md 갱신 필요** |
| 2 | 지도 위 날씨 오버레이 톤 — 반투명 그라디언트 레이어의 알파값 구체 수치 | DESIGN.md에서 확정 예정 |
| 3 | 온도 단위 전환 | 섭씨 고정 |
| 4 | 즐겨찾기 (localStorage) | 미구현 |
| 5 | 최근 검색 기록 | 미구현 |
| 6 | Wikimedia 이미지 URL 장기 안정성 — Commons 파일이 이동/삭제되면 링크가 깨질 수 있음 | `onerror` 폴백(이모지)으로 완화, 주기적 점검 권장 |

---

## 9. 향후 확장 후보

- 시간별 예보 24시간 그래프
- 브라우저 위치 기반 "내 주변 날씨"
- 두 도시 비교 뷰
- 랜드마크 확장 (24곳 이상 — 지도는 이미 전 세계를 담으므로 씬 폭 제약 없이 자유롭게 추가 가능)
- 공유용 URL (`?lat=&lon=&name=`)
- 날씨별 파티클 애니메이션 (비/눈), reduced-motion 대응 포함
