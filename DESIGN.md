# DESIGN.md — World Weather Explorer

| 항목 | 내용 |
|---|---|
| 버전 | v1.0 |
| 작성일 | 2026-08-18 |
| 대상 | `PRD.md` v0.2 구현 |
| 방향 | 글래스모피즘 · 고채도 배경 · Pretendard 단일 서체 |

---

## 1. 디자인 원칙

1. **배경이 주인공이다.** 날씨는 색으로 먼저 전달된다. UI는 그 위에 떠 있는 유리일 뿐, 색을 뺏지 않는다.
2. **유리는 두 종류다.** 랜드마크 유리와 UI 유리를 다르게 만들어야 깊이가 생긴다 (§4.3).
3. **대비는 색보다 우선한다.** 고채도 배경 위에서도 본문 대비 4.5:1을 무조건 지킨다. 지키지 못하면 스크림을 더 깐다.
4. **큰 라운드는 큰 면에만.** 작은 카드에 24px 라운드를 주면 내용이 눌린다.

---

## 2. 타이포그래피

### 2.1 서체

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/variable/pretendardvariable-dynamic-subset.min.css">
```

```css
:root {
  font-family: "Pretendard Variable", Pretendard, -apple-system, system-ui, sans-serif;
  font-feature-settings: "tnum" 1;  /* 고정폭 숫자 — 필수 */
}
```

> `tnum`은 선택이 아니라 필수다. 현지 시계가 1초마다 갱신되고 기온도 바뀌는데, 비례폭 숫자를 쓰면 글자 폭이 출렁인다.

### 2.2 스케일

| 역할 | 데스크톱 | 모바일 | Weight | 자간 |
|---|---|---|---|---|
| 기온 (디스플레이) | 72px / 1.0 | 56px | 700 | -0.03em |
| 도시명 | 28px / 1.3 | 24px | 600 | -0.02em |
| 현지 시각 | 20px / 1.2 | 18px | 500 | -0.01em |
| 사이트 타이틀 | 18px / 1.4 | 16px | 700 | -0.01em |
| 본문 | 15px / 1.6 | 15px | 400 | 0 |
| 라벨 · 캡션 | 13px / 1.5 | 13px | 500 | 0 |
| 배지 · 칩 | 12px / 1.4 | 12px | 600 | 0 |

기온 숫자와 `°C` 단위는 크기를 분리한다 — 단위는 기온의 0.4배, `align-self: flex-start`.

---

## 3. 컬러

### 3.1 배경 그라디언트 (PRD FR-09 대응)

전부 `linear-gradient(160deg, {from} 0%, {to} 100%)`.

| 그룹 | 낮 (from → to) | 밤 (from → to) | 낮 텍스트 |
|---|---|---|---|
| `clear` | `#38BDF8` → `#FDE68A` | `#1E3A8A` → `#0B1026` | 어두운 |
| `cloudy` | `#5B8FB9` → `#C7D5E0` | `#1F2937` → `#030712` | 어두운 |
| `fog` | `#8FA3B0` → `#DDE4E9` | `#334155` → `#0F172A` | 어두운 |
| `rain` | `#1D4ED8` → `#0E7490` | `#0C2340` → `#020617` | 밝은 |
| `snow` | `#A5D8FF` → `#F8FAFC` | `#1E3A5F` → `#0B1B2B` | 어두운 |
| `storm` | `#5B21B6` → `#1E1B4B` | `#1E1B4B` → `#020617` | 밝은 |

기본 테마(장소 선택 전) = 사용자 로컬 시각 기준 `clear-day` 또는 `clear-night`.

### 3.2 잉크 (텍스트) 세트

배경 밝기에 따라 두 세트를 스위치한다. 각 테마가 `--ink-*`를 직접 선언한다.

| 토큰 | 어두운 텍스트 세트 | 밝은 텍스트 세트 |
|---|---|---|
| `--ink-primary` | `#0F172A` | `#F8FAFC` |
| `--ink-secondary` | `rgba(15,23,42,0.72)` | `rgba(248,250,252,0.78)` |
| `--ink-tertiary` | `rgba(15,23,42,0.52)` | `rgba(248,250,252,0.58)` |

### 3.3 포인트 컬러

| 토큰 | 값 | 용도 |
|---|---|---|
| `--accent` | `#FBBF24` | 활성 필터 칩, 선택된 랜드마크 링, 포커스 링 |
| `--danger` | `#FB7185` | 에러 아이콘·재시도 버튼 테두리 |

앰버 하나만 쓴다. 6종 배경 중 어느 것 위에서도 대비가 확보되는 색이라 상태 표시에 안전하다.

---

## 4. 유리 시스템

### 4.1 3단 레이어

| 티어 | 대상 | blur | 배경 | 테두리 | 그림자 |
|---|---|---|---|---|---|
| **L1 패널** | 디테일 패널, 헤더 | 24px | `rgba(255,255,255,0.14)` | `1px rgba(255,255,255,0.22)` | `0 8px 32px rgba(0,0,0,0.24)` |
| **L2 카드** | 예보 카드, 현재 날씨 서브 그리드 | 12px | `rgba(255,255,255,0.10)` | `1px rgba(255,255,255,0.16)` | `0 4px 16px rgba(0,0,0,0.16)` |
| **L3 칩** | 필터 칩, 날씨 배지 | 8px | `rgba(255,255,255,0.12)` | `1px rgba(255,255,255,0.20)` | 없음 |

밝은 텍스트 세트(rain/storm)에서는 유리 배경 알파를 그대로 두고, **어두운 텍스트 세트**(clear/snow/fog/cloudy 낮)에서는 `rgba(255,255,255,0.38)`로 올려 흰 유리가 실제로 밝게 보이도록 한다.

### 4.2 스크림 (고채도 대응 필수)

고채도 배경이 블러를 통과하면 색이 탁해지고 대비가 무너진다. L1·L2에는 유리 뒤에 스크림을 한 겹 더 깐다.

```css
.glass-panel { position: relative; isolation: isolate; }
.glass-panel::before {
  content: ""; position: absolute; inset: 0; z-index: -1;
  border-radius: inherit;
  background: var(--scrim);          /* 밝은잉크: rgba(0,0,0,0.28) */
}                                    /* 어두운잉크: rgba(255,255,255,0.32) */
```

`backdrop-filter` 미지원 브라우저에서는 스크림 알파를 `0.72`로 올려 폴백한다.

### 4.3 랜드마크 유리 (UI 유리와 구분)

두 유리가 같으면 씬과 UI가 한 층으로 뭉개진다. **랜드마크는 테두리로, UI는 블러로** 존재감을 만든다.

| 속성 | 랜드마크 SVG | UI 패널 |
|---|---|---|
| 블러 | **없음** | 강함 (12–24px) |
| 채움 | `rgba(255,255,255,0.14)` | 동일하나 스크림 있음 |
| 테두리 | **2px `rgba(255,255,255,0.55)`** — 강함 | 1px `rgba(255,255,255,0.22)` — 약함 |
| 하이라이트 | 상단 모서리 `path`에 `rgba(255,255,255,0.8)` 1.5px | 없음 |
| 그림자 | 없음 (풍경에 떠 있지 않음) | 있음 |

```html
<!-- SVG 공통 정의 -->
<defs>
  <linearGradient id="glassFill" x1="0" y1="0" x2="0" y2="1">
    <stop offset="0%"   stop-color="#fff" stop-opacity="0.22"/>
    <stop offset="100%" stop-color="#fff" stop-opacity="0.08"/>
  </linearGradient>
</defs>
```

**랜드마크 상태별**

| 상태 | 처리 |
|---|---|
| 기본 | fill `url(#glassFill)`, stroke `rgba(255,255,255,0.55)` |
| hover / focus | stroke → `rgba(255,255,255,0.9)`, fill 알파 +0.08, 라벨 표시 |
| focus (키보드) | 추가로 `--accent` 2px 아웃라인 |
| 선택됨 | `--accent` stroke 2.5px 유지 |
| 필터 제외 | `opacity: 0.2` + `filter: grayscale(1)` (PRD FR-04-1) |

---

## 5. 형태 · 간격

### 5.1 라운드

| 대상 | 값 |
|---|---|
| 디테일 패널 | 24px (모바일 바텀시트는 상단만 24px) |
| 예보 카드 · 서브 그리드 셀 | 16px |
| 검색 입력창 | 16px |
| 필터 칩 · 날씨 배지 · 버튼 | `9999px` |

### 5.2 간격 (4px 배수)

| 대상 | 데스크톱 | 모바일 |
|---|---|---|
| 패널 안쪽 여백 | 24px | 20px |
| 카드 안쪽 여백 | 16px | 14px |
| 섹션 간격 | 24px | 20px |
| 카드 간 gap | 12px | 10px |
| 칩 간 gap | 8px | 8px |

### 5.3 레이아웃 치수

| 대상 | 값 |
|---|---|
| 디테일 패널 (데스크톱) | 폭 440px, 우측 고정, 상하 여백 24px |
| 디테일 패널 (모바일) | 하단 시트, 높이 85vh, 상단에 44px 드래그 핸들 영역 |
| 헤더 높이 | 64px (모바일 56px) |
| 예보 카드 | 폭 88px, 높이 148px |
| 터치 타깃 최소 | 44×44px |

---

## 6. 컴포넌트 스펙

### 6.1 검색창
L1 유리, 라운드 16px, 높이 44px, 좌측 🔍 16px 아이콘.
포커스 시 `--accent` 2px 링 + 유리 알파 +0.06.
후보 드롭다운은 L1 유리, 라운드 16px, 항목 높이 52px, 하이라이트 항목은 배경 `rgba(255,255,255,0.16)`.

### 6.2 필터 칩
L3 유리, 높이 32px, 좌우 패딩 14px, 12px/600.
**활성** → 배경 `--accent`, 텍스트 `#0F172A`, 우측에 12px `×`.
칩 그룹은 축별로 라벨(13px, `--ink-tertiary`)을 왼쪽에 두고 가로 나열. 모바일은 축마다 가로 스크롤.

### 6.3 랜드마크 날씨 배지
L3 유리, 라운드 999px, 높이 28px, 패딩 10px.
구성: `이모지 14px` + `기온 12px/600`. 랜드마크 실루엣 상단 중앙에서 12px 위.
프리페치 완료 시 `opacity 0→1` + `translateY(4px→0)`, 320ms, 랜드마크 인덱스당 40ms stagger.

### 6.4 현재 날씨 카드
패널 상단. 순서: 도시명(28px) → 국가(13px, tertiary) → 현지 시각(20px) → 기온(72px) + 이모지 48px 우측 → 날씨 설명(15px).
하단 2×2 서브 그리드(체감/습도/강수/풍속) — 각 셀 L2 유리, 라벨 13px tertiary + 값 20px/600.

### 6.5 예보 카드
88×148px, L2 유리.
위→아래: 요일(13px/600, 오늘은 `--accent`) → 날짜(12px tertiary) → 이모지 28px → 기온 막대 → 최고(16px/700)/최저(14px secondary) → 강수확률(12px, 0%면 생략).
**기온 막대** — 세로 40px 트랙, 7일 전체 min~max로 정규화. 트랙 `rgba(255,255,255,0.14)`, 채움은 하단 파랑 `#60A5FA` → 상단 앰버 `#FBBF24` 그라디언트, 폭 4px 라운드 999px.

### 6.6 상태 화면

| 상태 | 스펙 |
|---|---|
| 스켈레톤 | 유리 위 `rgba(255,255,255,0.16)` 블록, 1.6s ease-in-out 무한 펄스 (0.16↔0.28). **스피너 금지** |
| 빈 결과 | 🔍 40px + 제목 15px/600 + 팁 3줄 13px tertiary, 세로 중앙, 상하 여백 32px |
| 에러 | ⚠️ 40px + 메시지 15px + 재시도 버튼(L2 유리, `--danger` 1px 테두리, 높이 40px) |
| 필터 0건 | 씬 위 중앙 오버레이. L1 유리, 패딩 24px, "조건에 맞는 곳이 없습니다" + 초기화 버튼 |

---

## 7. 모션

| 대상 | duration | easing |
|---|---|---|
| 배경 그라디언트 전환 | 800ms | `ease-in-out` |
| 패널 등장/퇴장 | 320ms | `cubic-bezier(0.32,0.72,0,1)` |
| 랜드마크 hover | 160ms | `ease-out` |
| 배지 페이드인 | 320ms (40ms stagger) | `ease-out` |
| 칩 토글 | 120ms | `ease-out` |

패널은 데스크톱 `translateX(24px)→0`, 모바일 `translateY(100%)→0`.

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```
스켈레톤 펄스는 이 경우 정적 `rgba(255,255,255,0.20)` 단색으로 대체한다.

---

## 8. Tailwind CDN 설정

```html
<script src="https://cdn.tailwindcss.com"></script>
<script>
tailwind.config = {
  theme: {
    extend: {
      fontFamily: { sans: ['"Pretendard Variable"','Pretendard','system-ui','sans-serif'] },
      borderRadius: { panel: '24px', card: '16px' },
      backdropBlur: { panel: '24px', card: '12px', chip: '8px' },
      boxShadow: {
        panel: '0 8px 32px rgba(0,0,0,0.24)',
        card:  '0 4px 16px rgba(0,0,0,0.16)',
      },
      colors: {
        accent: '#FBBF24',
        danger: '#FB7185',
        ink:  { 1:'var(--ink-primary)', 2:'var(--ink-secondary)', 3:'var(--ink-tertiary)' },
      },
      transitionTimingFunction: { panel: 'cubic-bezier(0.32,0.72,0,1)' },
    }
  }
}
</script>
```

테마 스위칭은 `<body data-theme="rain-night">` 로 하고, CSS에서 `[data-theme]` 별로 `--bg-from · --bg-to · --ink-* · --scrim · --glass-bg` 를 선언한다. Tailwind는 레이아웃만 담당하고 테마 색은 CSS 변수가 담당한다.

---

## 9. 접근성 체크리스트

- [ ] 6개 배경 × 낮/밤 12조합 전부에서 본문 대비 4.5:1, 큰 활자 3:1 이상
- [ ] `--accent` 포커스 링이 12조합 전부에서 식별 가능
- [ ] 랜드마크 `aria-label` — "에펠탑, 파리, 현재 15도 맑음"
- [ ] 필터 칩 `aria-pressed`
- [ ] 패널 `role="dialog"` + `aria-modal="true"` + 포커스 트랩 + 닫을 때 트리거로 복귀
- [ ] 검색 드롭다운 `role="listbox"` / `role="option"` / `aria-activedescendant`
- [ ] 유리 위 텍스트는 반드시 스크림 레이어 위에 올릴 것
- [ ] `prefers-reduced-motion` 검수
- [ ] 터치 타깃 44px 이상

---

## 10. 미결

| # | 항목 | 처리 |
|---|---|---|
| 1 | 랜드마크 12종 실루엣 `path` 데이터 | 구현 단계에서 작성 |
| 2 | `backdrop-filter` 폴백 검수 (구형 Firefox) | 스크림 알파 상향으로 대응 예정 |
| 3 | 원경 산맥·전경 지면 레이어의 유리 적용 여부 | 랜드마크만 유리, 배경 레이어는 단색 실루엣으로 진행 |
