# 코스믹 스케일 (Cosmic Scale)

태양계에서 관측 가능한 우주의 끝(약 465억 광년)까지를 **하나의 3D 공간**에서 탐색하는 인터랙티브 우주 지도입니다.
모든 천체는 **실제 관측 방향(적경·적위)** 에 놓이고, 거리는 **로그(log₁₀) 축척**으로 압축되어 AU부터 Gly까지 한 화면에서 이어서 볼 수 있습니다.

- 배포 주소: https://cosmic-scale-radsci.web.app
- Firebase 프로젝트: `cosmic-scale-radsci` (Hosting)

---

## 1. 주요 기능

| 기능 | 설명 |
|---|---|
| 로그 축척 3D 지도 | 1 AU ~ 465억 광년을 한 공간에 배치. 바깥으로 한 눈금 = 10배 |
| 실제 좌표 | 천체 75개를 J2000 적경·적위와 실측 거리로 배치 |
| 단위 전환 | 자동 / 광년 / pc / AU / km, 큰 수는 만·억·조로 표기 |
| 시간 제어 | 실시간(LIVE), 과거·미래 빨리감기(1분/초 ~ 1만 년/초), 날짜 이동 (±약 10만 년) |
| 실제 운동 | 행성 케플러 궤도, 가까운 별 20개의 고유운동, 보이저 1·2호, 소행성대·카이퍼 벨트 공전 |
| 거리 측정 | 두 천체 사이의 실제 3D 직선거리 (광년·pc·AU·km·빛 시간·탐사선 시간) |
| 우주 투어 | 지구 → CMB까지 16단계 자동 카메라 투어 + 해설 |
| 정보 패널 | 천체별 정사각형 사진(관측 사진/상상도/도식 표시), 출처·라이선스, 거리·분광형 등 |
| 위성계 보기 | 지구·화성·목성·토성·천왕성·해왕성·명왕성을 행성 중심 로그 축척으로 확대, 주요 위성 27개 공전 |
| 표시 옵션 | 라벨, 공전 궤도, 거리 눈금 원(태양 중심 점선) 켜기/끄기 |
| 검색·축척 사다리 | 한글/영문 천체 검색, 왼쪽 축척 단계 클릭으로 해당 스케일로 이동 |

---

## 2. 기술 스택

| 영역 | 사용 기술 | 비고 |
|---|---|---|
| 렌더링 | **Three.js r147** (`three.min.js`) | jsDelivr CDN에서 로드, WebGL |
| 카메라 조작 | Three.js `OrbitControls` (examples/js 버전) | r148부터 examples/js가 제거되어 r147로 고정 |
| 언어 | 순수 HTML + CSS + JavaScript (ES2020) | 빌드 도구·프레임워크 없음, 파일 하나 |
| 폰트 | Google Fonts: **Orbit**(제목), **IBM Plex Sans KR**(본문), **IBM Plex Mono**(수치) | 시스템 폰트로 대체 가능 |
| 호스팅 | **Firebase Hosting** (정적 사이트) | 서버·DB 없음 |
| 이미지 | NASA, ESO, ESA/Hubble, ESA/Webb, Wikimedia Commons | 360×360 JPEG로 정규화 |

외부 의존성은 Three.js(CDN)와 Google Fonts뿐이며, 그 외 데이터는 모두 `index.html`과 `img/`에 들어 있습니다.

---

## 3. 폴더 구조

```
cosmic-scale/
├── firebase.json      # Hosting 설정 (site, public 폴더, 이미지 캐시 헤더)
├── .firebaserc        # 기본 프로젝트 = cosmic-scale-radsci
├── .gitignore
├── .github/workflows/ # GitHub Actions → Firebase 자동 배포 (main: 실서비스, PR: 미리보기)
├── README.md
└── public/
    ├── index.html     # 앱 전체 (HTML + CSS + JS + 천체 데이터)
    └── img/           # 천체 사진 74장 (id.jpg, 360×360)
```

---

## 4. 작동 방식

### 4.1 로그 거리 매핑

실제 거리 `d`(광년)를 화면 공간 반지름 `r`로 바꿉니다.

```
r = 10 · log10(1 + d / 1e-6)        // f(d)
d = 1e-6 · (10^(r/10) − 1)          // finv(r)
```

- `1e-6 광년 ≈ 0.063 AU`를 기준점으로 잡아 태양 근처가 0으로 뭉개지지 않게 했습니다.
- 대략적인 반지름: 지구 궤도 12, 해왕성 27, 프록시마 66, 은하 중심 104, 안드로메다 124, 처녀자리 은하단 137, CMB 166.6
- **방향은 그대로, 거리만 압축**합니다 (`toScene(real) = real.normalize() * f(|real|)`).
  그래서 화면에서 두 천체 사이의 간격은 실제 비율과 다르며, 거리 측정은 항상 **실제 좌표(real)** 로 계산합니다.

### 4.2 좌표계

- 입력: J2000 적도좌표 (적경 RA, 적위 Dec)
- IAU 표준 회전행렬로 **은하좌표**로 변환한 뒤 Three.js 축에 매핑합니다.
  - X = 은하 중심 방향, Y(위) = 은하 북극, Z = −(은경 90° 방향)
  - 결과적으로 **은하면이 수평면(XZ)** 이 됩니다.
- 행성 궤도(황도 좌표)는 황도 경사 23.4393°로 적도좌표로 바꾼 뒤 같은 변환을 거칩니다. 그래서 태양계 궤도면은 은하면에 대해 약 60° 기울어 보입니다.

관련 함수: `eqVec()`, `eqToThree()`, `eclToThree()`, `toScene()`

### 4.3 천체 데이터

`index.html` 안의 배열로 관리합니다.

- `PLANETS`: 행성 8개 + 명왕성. JPL "Approximate Positions of the Planets"의 J2000 궤도요소 `[a, e, I, L, ϖ, Ω]`와 세기당 변화율
- `OBJ`: `[id, 한글명, 영문명, 분류, RA°, Dec°, 거리(광년), 중요도, 설명, {sp, note, lb}]`
  - `sp`: 분광형(O~M) → 점 색상
  - `lb`: 빛이 출발한 시점(억 년 전, 우주론적 천체)
  - `note`: 패널 하단 주석
- `PM`: 가까운 별 20개의 고유운동(mas/yr)과 시선속도(km/s)
- `CAT`: 분류별 한글 이름, 색상, 점 크기
- `IMG`: 천체 id별 사진 메타데이터 `{t: 종류, c: 출처·라이선스, u: 원본 링크, s: 설명}`

거리 기준:
- 가까운 별: Hipparcos/Gaia 연주시차
- 근처 은하: 세페이드·TRGB 측정값
- 먼 천체(3C 273 이후): 적색편이 → Planck 우주론 기준 **공동거리(comoving distance)**

### 4.4 시간 시뮬레이션

상태 객체 `sim = {ms, live, paused, dir, idx}`가 시뮬레이션 시각(ms)을 관리하고, 매 프레임 `tickSim()` → `applySim()`이 위치를 갱신합니다.

| 대상 | 계산 방법 |
|---|---|
| 행성 | `keplerReal()`: 평균근점이각 → 케플러 방정식(뉴턴법 10회) → 궤도면 좌표 → 황도 회전. a·e·I·ϖ·Ω 변화율은 ±3000년까지만 적용 |
| 가까운 별 | 고유운동 + 시선속도로 3D 속도벡터(광년/년)를 만들고 J2000부터 등속 직선 운동 |
| 보이저 1·2호 | 2026년 말 거리에서 연 3.57 AU / 3.25 AU로 직선 외삽. 5 AU 미만(발사 전)이면 숨김 |
| 소행성대·카이퍼 벨트 | 점마다 `a`와 초기 위상을 갖고 주기 `a^1.5`년으로 공전 |

- 선택한 천체가 움직이면 카메라도 같은 만큼 따라갑니다 (`placeObj()`).
- 정보 패널과 거리 측정 값은 0.25초마다 갱신됩니다.

### 4.5 렌더링 구성

| 요소 | 구현 |
|---|---|
| 천체 | `THREE.Sprite` + 방사형 그라디언트 캔버스 텍스처, 가산 혼합. 카메라 거리에 비례해 크기를 조정해 화면상 크기를 일정하게 유지 |
| 은하수 원반 | 점 26,000개 절차 생성 (지수 원반 + 4개 로그 나선팔 + 막대/팽대부). 태양에서 900광년 이내는 제외 |
| 마젤란은하, 오르트 구름, 배경 은하 | 점 구름 (개념 모형) |
| CMB | 반지름 f(457억 광년)의 구에 절차 노이즈 텍스처, 안쪽/바깥쪽 2겹 반투명 |
| 궤도선 | J2000 궤도요소로 타원을 200점 샘플링해 로그 공간에 그림 |
| 거리 눈금 | 은하면 위 점선 원 16개 (1 AU ~ 465억 광년) |
| 깊이 | `logarithmicDepthBuffer: true`로 0.001 ~ 5000 범위 깊이 정밀도 확보 |

### 4.6 라벨과 선택

- 라벨은 WebGL이 아니라 **DOM 요소**이며 매 프레임 화면 좌표로 투영합니다.
- 우선순위 = 중요도 × 3 − log₁₀(카메라 거리) × 4 (선택된 천체는 최우선)
- 사각형 충돌 검사로 겹치는 라벨은 숨기고, 화면당 최대 37개까지만 표시합니다.
- 클릭은 레이캐스트 대신 **화면 좌표 기준 24px 이내 최근접 천체**를 고릅니다 (작은 점도 잘 잡힘).

### 4.7 카메라 이동

`flyTo(target, dist, dur, dir)`:
- 타깃 위치는 선형 보간, 카메라 거리는 **로그 보간**(스케일이 크게 바뀌어도 자연스럽게), 방향은 정규화 보간
- easeInOutCubic, `prefers-reduced-motion`이면 즉시 이동
- 사용자가 드래그하면 비행이 중단되고 투어는 일시정지됩니다.

### 4.8 UI 구성

- 좌상단: 제목, 시점 중심, 시야 축척
- 좌측: 축척 사다리 (900px 이하에서 숨김)
- 우상단: 검색 / 우측: 정보 패널 (640px 이하에서 하단 시트)
- 하단 도크: 단위, 투어, 거리 측정, 라벨/궤도/거리 눈금 토글, 전체 보기, **시간** 버튼
  - 시간 버튼을 누르면 `grid-template-rows: 0fr → 1fr` 전환으로 시간 제어판이 아래로 펼쳐집니다.
  - 도크 높이는 `ResizeObserver`로 `--dockH` CSS 변수에 반영되어 투어 자막·모바일 패널이 자동으로 비켜납니다.
- 단일 다크 테마, 모든 색은 `:root` CSS 변수로 관리

### 4.9 위성계 보기

- 행성 패널의 **위성계 보기** 버튼이나 위성 이름 검색으로 들어갑니다. 별도 `THREE.Scene`(`sysScene`)으로 전환하고, 나갈 때 원래 카메라 위치를 복원합니다.
- 거리 매핑은 행성 반지름 `R` 기준 로그입니다: `r = 10 · log10(1 + d_km / R)` (행성 표면이 r≈3).
- 데이터: `MOONS` 객체 `[id, 한글명, 영문명, 장반경 km, 공전주기 일(음수=역행), 반지름 km, 색, 발견, 중요도, 설명]`, 행성 반지름 `PLANET_R`.
- 위성은 실제 공전 주기로 시간 제어와 연동해 돌지만, 궤도는 한 평면의 원으로 단순화했고 위상(현재 위치)은 근사입니다.
- 행성 원판은 `img/<행성>.jpg`를 원형으로 잘라 스프라이트로 그리며, 토성은 밝기 기반 투명 처리로 고리를 살립니다.

---

## 5. 개발 환경 설정

### 5.1 필요 도구

| 도구 | 버전 | 용도 |
|---|---|---|
| Node.js | 18 이상 | Firebase CLI 실행 |
| Firebase CLI | `firebase-tools` 최신 | 배포 |
| 브라우저 | WebGL 지원 (Chrome, Edge, Safari, Firefox) | 실행 |

```bash
npm install -g firebase-tools
firebase login            # jaykim6674@gmail.com 계정
```

### 5.2 로컬에서 실행

빌드가 필요 없습니다. 이미지를 상대 경로로 불러오므로 `file://`보다는 로컬 서버를 권장합니다.

```bash
cd cosmic-scale
firebase emulators:start --only hosting   # http://localhost:5000
# 또는
npx serve public
# 또는
python -m http.server 8000 -d public
```

### 5.3 배포

```bash
cd cosmic-scale
firebase deploy --only hosting
```

- `.firebaserc`에 기본 프로젝트(`cosmic-scale-radsci`)가, `firebase.json`에 사이트 이름이 지정되어 있어 추가 옵션이 필요 없습니다.
- 이미지(`img/**`)는 7일 캐시 헤더가 붙습니다. 이미지를 교체했는데 반영이 늦으면 파일명을 바꾸거나 강력 새로고침하세요.

### 5.5 GitHub 연동 (자동 배포)

- 저장소: https://github.com/jaykim1993/cosmic-scale
- `main`에 push하면 GitHub Actions가 Firebase Hosting(live)으로 자동 배포합니다 (`.github/workflows/firebase-hosting-merge.yml`).
- PR을 열면 미리보기 채널에 임시 배포하고 PR에 미리보기 URL을 댓글로 남깁니다 (`firebase-hosting-pull-request.yml`, 7일 후 만료).
- 배포 인증은 GitHub 저장소 Secret `FIREBASE_SERVICE_ACCOUNT_COSMIC_SCALE_RADSCI`(Firebase 서비스 계정 키 JSON)를 씁니다. 이 키는 `firebase init hosting:github`가 자동으로 만들어 등록하며, 저장소에 파일로 커밋하면 안 됩니다.

작업 흐름:

```bash
git checkout -b feature/새기능
# 수정
git commit -am "설명"
git push -u origin feature/새기능   # → PR 생성 시 미리보기 URL
# PR 머지 → main → 실서비스 자동 배포
```

### 5.6 커스텀 도메인

Firebase 콘솔 → Hosting → **커스텀 도메인 추가** → 안내되는 A/TXT 레코드를 DNS에 등록하면 SSL까지 자동 발급됩니다.

---

## 6. 수정 가이드

### 천체 추가

1. `OBJ` 배열에 한 줄 추가
   ```js
   ['m42b','새 천체','New Object','nebula', RA, Dec, 거리_광년, 중요도(1~5), '설명', {note:'주석'}],
   ```
2. 사진이 있으면 `public/img/<id>.jpg`(360×360 권장)를 넣고 `IMG`에 메타데이터 추가
   ```js
   Object.assign(IMG, {"m42b": {"t":"photo","c":"NASA","u":"https://...","s":"설명"}});
   ```
   `t`는 `photo`(관측 사진) / `art`(상상도) / `diagram`(도식) / `map`(관측 지도) 중 하나입니다. 사진이 없으면 분광형 색 기반 대체 이미지가 자동으로 표시됩니다.
3. 가까운 별이라 시간에 따라 움직이게 하려면 `PM`에 `[μα*, μδ, 시선속도]` 추가

### 투어 단계 수정

`STEPS` 배열에서 `{o: 천체id}` 또는 `{s: 거리_광년}`과 제목(`h`), 설명(`p`)을 편집합니다. 단계당 9초(`tour.dur`)입니다.

### 축척 바꾸기

`D0`(기준 거리)와 `A`(배율)를 바꾸면 전체 매핑이 바뀝니다. 바꾼 뒤에는 `viewDistFor()`의 카메라 거리와 `controls.maxDistance`도 함께 조정하세요.

---

## 7. 이미지 출처와 라이선스

| 출처 | 라이선스 | 비고 |
|---|---|---|
| NASA (images.nasa.gov) | 퍼블릭 도메인 | 일부 이미지는 공동 기관 크레딧 포함 · 위성 사진 27장 포함 |
| ESO | CC BY 4.0 | EHT 블랙홀 이미지는 EHT Collaboration 크레딧 |
| ESA/Hubble | CC BY 4.0 | |
| ESA/Webb | CC BY 4.0 | |
| Wikimedia Commons | 파일별 (PD, CC0, CC BY, CC BY-SA, GFDL) | 작가·라이선스를 패널에 표시 |

각 천체 패널의 출처 링크가 원본 페이지로 연결되며, CC 라이선스의 저작자 표시 요건은 이 링크와 표기로 충족합니다. 사진이 없는 천체(카펠라)는 분광형 색으로 그린 대체 이미지를 씁니다.

---

## 8. 알려진 한계

- **로그 축척 특성상** 화면 간격은 실제 비율이 아닙니다. 거리 수치는 항상 실제 좌표 기준입니다.
- 행성 위치는 JPL 근사식이라 1800~2050년 밖으로 갈수록 오차가 커집니다.
- 별의 운동은 등속 직선 가정이라 수만 년 규모에서는 근사입니다. 적경·적위 표시는 J2000 고정값입니다.
- 보이저의 1980년대 이전 경로는 실제 비행경로가 아닌 직선 외삽입니다.
- 은하수, 오르트 구름, 배경 은하 분포는 실제 구조를 본뜬 **개념 모형**입니다.
- 베텔게우스, 데네브 등 일부 초거성은 거리 추정치 편차가 큽니다.
- 은하 회전과 먼 천체의 움직임은 포함하지 않았습니다.

---

## 9. 참고 자료

- JPL Solar System Dynamics — Approximate Positions of the Planets
- Hipparcos / Gaia 카탈로그, SIMBAD, NED
- Planck 2018 우주론 파라미터
- GRAVITY Collaboration (궁수자리 A* 거리 8.18 kpc)
- Three.js r147 문서
