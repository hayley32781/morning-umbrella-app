# 아침 우산 알리미 v2 비주얼 리프레시 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 기존 "아침 우산 알리미" 단일 페이지에 날씨별 움직이는 일러스트 배경, 카드형 고정 폭 레이아웃, UV 지수 기반 "양산" 탭, 습도·바람·한줄 코멘트·시간대별 타임라인을 추가한다.

**Architecture:** 기존 `index.html` 하나를 계속 확장한다. 핵심 로직 함수(`needsUmbrella`, `getCurrentWeatherCode`, `weatherCodeToCategory` 등)와 `getUserLocation`은 그대로 유지하고, HTML/CSS를 카드+레이어 구조로 교체한 뒤, `fetchWeather`에 UV/습도/바람 데이터를 추가하고, 우산/양산 두 "뷰"를 계산해 탭으로 전환하는 방식으로 확장한다.

**Tech Stack:** HTML5, CSS3(순수 `@keyframes` 애니메이션), Vanilla JavaScript, Open-Meteo Forecast API, `navigator.geolocation`, GitHub Pages(기존 저장소에 재배포).

## Global Constraints

- 파일은 계속 `index.html` 하나로 구성한다(`README.md`는 별도 문서). 빌드 도구, 프레임워크, npm 패키지를 사용하지 않는다.
- Open-Meteo hourly 요청에 기존 `precipitation_probability,weathercode`에 더해 `uv_index`, `relative_humidity_2m`, `wind_speed_10m`을 추가한다.
- 판단 시간대는 기존과 동일하게 **06:00~21:00**. 강수확률 임계값은 기존과 동일하게 **40**. UV 임계값은 **6** (신규 상수 `UV_THRESHOLD`).
- 레이아웃은 **카드형 고정 폭**(너비 380px, `max-width: 92vw`)으로 화면 중앙에 배치하고, 카드 바깥은 차분한 여백 배경.
- 카드 내부는 **배경 애니메이션 레이어**(뒤)와 **텍스트 콘텐츠 레이어**(앞)로 명확히 분리한다. 텍스트 레이어에는 배경 패널을 넣지 않는다(텍스트만 애니메이션 위에 직접 표시).
- 비/눈 장면에서 낙하 파티클(빗방울/눈송이)은 구름 캐릭터보다 **아래** 레이어에 그려, 구름 근처에서 시작해 구름 뒤로 가려졌다 나오는 것처럼 보이게 한다.
- 시간대별 타임라인은 **06, 09, 12, 15, 18, 21시**(3시간 간격, 6개 지점)만 샘플링한다.
- 양산 탭은 UV 판정 결과에 따라 **맑음 또는 흐림 장면만** 사용한다(비/눈 장면은 양산 탭에 쓰지 않는다).
- 자동 테스트 코드(pytest, jest 등)는 작성하지 않는다. 순수 함수는 실제 `node` 실행으로, DOM과 얽힌 코드는 정독을 통한 코드 추적으로 검증한다.
- 참고 spec: `docs/superpowers/specs/2026-07-31-umbrella-app-v2-visual-refresh-design.md`
- 기존 배포 주소(변경 없음): https://hayley32781.github.io/morning-umbrella-app/

---

### Task 1: 카드형 레이아웃 + 4개 날씨 애니메이션 장면 (비주얼 셸 교체)

**Files:**
- Modify: `index.html` (전체 내용 교체)

**Interfaces:**
- Consumes: 없음 (첫 작업, 기존 `index.html`의 로직 함수는 그대로 옮겨온다)
- Produces:
  - DOM id: `scene-sunny`, `scene-cloudy`, `scene-rainy`, `scene-snowy` (각각 `.scene` 클래스, `.active`가 붙은 것만 보임)
  - DOM id: `tab-umbrella`, `tab-parasol` (아직 클릭 이벤트 없음, 다음 태스크에서 연결)
  - DOM id 유지: `icon`, `result`, `detail`, `error`, `error-message`, `retry-btn`
  - DOM id 신규(빈 채로, 다음 태스크에서 채움): `humidity-wind`, `comment`, `timeline`
  - `function showScene(category: 'sunny'|'cloudy'|'rainy'|'snowy'): void`
  - 기존 로직 함수 전부 그대로 유지: `THRESHOLD_PERCENT`, `WINDOW_START_HOUR`, `WINDOW_END_HOUR`, `getTodayWindowPrecipProbabilities`, `needsUmbrella`, `getCurrentWeatherCode`, `weatherCodeToCategory`, `getUserLocation`, `fetchWeather`

- [ ] **Step 1: `index.html` 전체 내용을 아래로 교체**

```html
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>아침 우산 알리미</title>
<style>
  * { box-sizing: border-box; }
  body {
    margin: 0;
    min-height: 100vh;
    font-family: -apple-system, "Apple SD Gothic Neo", "Malgun Gothic", sans-serif;
  }
  .page {
    min-height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    background: #f0f1f3;
    padding: 1.5rem;
  }
  .app-card {
    position: relative;
    width: 380px;
    max-width: 92vw;
    border-radius: 24px;
    overflow: hidden;
    box-shadow: 0 12px 32px rgba(0,0,0,0.15);
    background: #e9edf1;
  }

  /* 배경 애니메이션 레이어 */
  .bg-layer { position: absolute; inset: 0; z-index: 1; overflow: hidden; }
  .scene { position: absolute; inset: 0; display: none; }
  .scene.active { display: block; }
  #scene-sunny { background: linear-gradient(180deg, #fff3d6, #ffe3b0); }
  #scene-cloudy { background: linear-gradient(180deg, #d9dee3, #eef1f4); }
  #scene-rainy { background: linear-gradient(180deg, #cfd8e3, #e9edf1); }
  #scene-snowy { background: linear-gradient(180deg, #dceaf5, #f3f8fc); }

  .sun-face {
    position: absolute; top: 24px; left: 50%;
    width: 50px; height: 50px; margin-left: -25px;
    border-radius: 50%; background: #ffb84d;
    animation: bounceSoft 2.2s ease-in-out infinite;
  }
  .sun-face::before, .sun-face::after {
    content: ''; position: absolute; top: 18px; width: 5px; height: 5px;
    border-radius: 50%; background: #7a4a1a;
  }
  .sun-face::before { left: 14px; }
  .sun-face::after { left: 30px; }
  @keyframes bounceSoft { 0%, 100% { transform: translateY(0); } 50% { transform: translateY(-10px); } }

  .puff-cloud {
    position: absolute; width: 40px; height: 16px;
    background: #fff; border-radius: 20px; opacity: 0.85;
    animation: driftAcross 9s linear infinite;
  }
  @keyframes driftAcross { from { transform: translateX(-60px); } to { transform: translateX(340px); } }

  .cloud-char {
    position: absolute; top: 60px; left: 50%;
    width: 64px; height: 32px; margin-left: -32px;
    background: #fff; border-radius: 30px;
    z-index: 2;
    animation: cloudSway 3s ease-in-out infinite;
  }
  .cloud-char::before {
    content: ''; position: absolute; top: -14px; left: 14px;
    width: 30px; height: 30px; background: #fff; border-radius: 50%;
  }
  .cloud-char .eye {
    position: absolute; top: 10px; width: 5px; height: 5px;
    border-radius: 50%; background: #6b7280;
  }
  .cloud-char .eye.l { left: 20px; }
  .cloud-char .eye.r { left: 38px; }
  @keyframes cloudSway { 0%, 100% { transform: translateY(0); } 50% { transform: translateY(-6px); } }

  .raindrop {
    position: absolute; top: 56px;
    width: 2px; height: 14px;
    background: #6b93c7;
    z-index: 1;
    animation: rainFall 1s linear infinite;
  }
  @keyframes rainFall {
    from { transform: translateY(0); opacity: 0; }
    15% { opacity: 0.9; }
    90% { opacity: 0.9; }
    to { transform: translateY(220px); opacity: 0; }
  }

  .snowflake {
    position: absolute; top: 56px;
    width: 8px; height: 8px; font-size: 10px; color: #fff; text-align: center;
    z-index: 1;
    animation: snowFall 4s linear infinite;
  }
  @keyframes snowFall {
    from { transform: translate(0, 0) rotate(0deg); opacity: 0; }
    10% { opacity: 1; }
    50% { transform: translate(12px, 110px) rotate(180deg); }
    90% { opacity: 1; }
    to { transform: translate(-6px, 220px) rotate(360deg); opacity: 0; }
  }

  /* 텍스트 콘텐츠 레이어 */
  .content-layer {
    position: relative; z-index: 3;
    padding: 1.5rem 1.5rem 0.5rem;
    text-align: center;
  }
  .tabs { display: flex; gap: 0.5rem; margin-bottom: 1rem; }
  .tab {
    flex: 1;
    padding: 0.5rem 0;
    border: none; border-radius: 10px;
    background: rgba(255,255,255,0.6);
    font-size: 0.9rem; cursor: pointer; color: #333;
  }
  .tab.active { background: #333; color: #fff; font-weight: 600; }
  #icon { font-size: 5rem; }
  #result { font-size: 2rem; font-weight: 700; margin: 0.5rem 0; }
  #detail { font-size: 1.1rem; color: #333; }
  #humidity-wind { font-size: 0.85rem; color: #444; margin-top: 0.4rem; }
  #comment { font-size: 0.85rem; color: #444; margin-top: 0.6rem; }
  #error { display: none; font-size: 1.2rem; margin-top: 1rem; }
  #retry-btn {
    margin-top: 1rem; padding: 0.6rem 1.2rem; font-size: 1rem;
    border: none; border-radius: 8px; background: #333; color: white; cursor: pointer;
  }

  .timeline {
    position: relative; z-index: 3;
    text-align: center; font-size: 1.1rem; letter-spacing: 0.3rem;
    padding: 0.75rem 0 1.25rem;
  }
</style>
</head>
<body>
  <div class="page">
    <div class="app-card">
      <div class="bg-layer">
        <div class="scene active" id="scene-sunny">
          <div class="puff-cloud" style="top:14px; animation-delay:0s"></div>
          <div class="puff-cloud" style="top:50px; opacity:0.5; transform:scale(0.7); animation-delay:-4s"></div>
          <div class="sun-face"></div>
        </div>
        <div class="scene" id="scene-cloudy">
          <div class="cloud-char"><div class="eye l"></div><div class="eye r"></div></div>
        </div>
        <div class="scene" id="scene-rainy">
          <div class="raindrop" style="left:15%; animation-delay:0s"></div>
          <div class="raindrop" style="left:30%; animation-delay:0.2s"></div>
          <div class="raindrop" style="left:45%; animation-delay:0.4s"></div>
          <div class="raindrop" style="left:60%; animation-delay:0.1s"></div>
          <div class="raindrop" style="left:75%; animation-delay:0.35s"></div>
          <div class="raindrop" style="left:90%; animation-delay:0.25s"></div>
          <div class="cloud-char"><div class="eye l"></div><div class="eye r"></div></div>
        </div>
        <div class="scene" id="scene-snowy">
          <div class="snowflake" style="left:15%; animation-delay:0s">❄</div>
          <div class="snowflake" style="left:35%; animation-delay:1s">❄</div>
          <div class="snowflake" style="left:50%; animation-delay:0.5s">❄</div>
          <div class="snowflake" style="left:65%; animation-delay:1.7s">❄</div>
          <div class="snowflake" style="left:82%; animation-delay:0.8s">❄</div>
          <div class="cloud-char"><div class="eye l"></div><div class="eye r"></div></div>
        </div>
      </div>
      <div class="content-layer">
        <div class="tabs">
          <button id="tab-umbrella" class="tab active" type="button">☂️ 우산</button>
          <button id="tab-parasol" class="tab" type="button">🌂 양산</button>
        </div>
        <div id="icon">⏳</div>
        <div id="result">날씨 확인 중...</div>
        <div id="detail"></div>
        <div id="humidity-wind"></div>
        <div id="comment"></div>
        <div id="error">
          <p id="error-message"></p>
          <button id="retry-btn" type="button">다시 시도</button>
        </div>
      </div>
      <div id="timeline" class="timeline"></div>
    </div>
  </div>
  <script>
    const THRESHOLD_PERCENT = 40;
    const WINDOW_START_HOUR = 6;
    const WINDOW_END_HOUR = 21;

    const SCENES = ['sunny', 'cloudy', 'rainy', 'snowy'];
    function showScene(category) {
      SCENES.forEach((c) => {
        document.getElementById('scene-' + c).classList.toggle('active', c === category);
      });
    }

    function getTodayWindowPrecipProbabilities(hourlyTimes, hourlyPrecipProb, startHour = WINDOW_START_HOUR, endHour = WINDOW_END_HOUR) {
      const todayStr = hourlyTimes[0].slice(0, 10);
      const result = [];
      for (let i = 0; i < hourlyTimes.length; i++) {
        const [datePart, timePart] = hourlyTimes[i].split('T');
        const hour = parseInt(timePart.slice(0, 2), 10);
        if (datePart === todayStr && hour >= startHour && hour <= endHour) {
          result.push(hourlyPrecipProb[i]);
        }
      }
      return result;
    }

    function needsUmbrella(precipProbs, threshold = THRESHOLD_PERCENT) {
      if (precipProbs.length === 0) return false;
      return Math.max(...precipProbs) >= threshold;
    }

    function getCurrentWeatherCode(hourlyTimes, hourlyWeatherCode) {
      const now = new Date();
      const pad = (n) => String(n).padStart(2, '0');
      const nowHourStr = `${now.getFullYear()}-${pad(now.getMonth() + 1)}-${pad(now.getDate())}T${pad(now.getHours())}:00`;
      let idx = hourlyTimes.indexOf(nowHourStr);
      if (idx === -1) idx = 0;
      return hourlyWeatherCode[idx];
    }

    function weatherCodeToCategory(code) {
      if (code === 0 || code === 1) return 'sunny';
      if ([2, 3, 45, 48].includes(code)) return 'cloudy';
      if ([71, 73, 75, 77, 85, 86].includes(code)) return 'snowy';
      return 'rainy';
    }

    function getUserLocation() {
      return new Promise((resolve, reject) => {
        if (!navigator.geolocation) {
          reject(new Error('이 브라우저는 위치 확인을 지원하지 않아요'));
          return;
        }
        navigator.geolocation.getCurrentPosition(
          (position) => resolve({ lat: position.coords.latitude, lon: position.coords.longitude }),
          () => reject(new Error('위치 정보가 필요해요')),
          { timeout: 10000 }
        );
      });
    }

    async function fetchWeather(lat, lon) {
      const url = `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&hourly=precipitation_probability,weathercode&daily=temperature_2m_max,temperature_2m_min&timezone=auto`;
      let response;
      try {
        response = await fetch(url);
      } catch (e) {
        throw new Error('날씨 정보를 가져올 수 없어요. 잠시 후 다시 시도해주세요.');
      }
      if (!response.ok) {
        throw new Error('날씨 정보를 가져올 수 없어요');
      }
      const data = await response.json();
      return {
        hourlyTimes: data.hourly.time,
        hourlyPrecipProb: data.hourly.precipitation_probability,
        hourlyWeatherCode: data.hourly.weathercode,
        todayMax: data.daily.temperature_2m_max[0],
        todayMin: data.daily.temperature_2m_min[0],
      };
    }

    function renderResult({ umbrella, max, min, maxPrecipProb, category }) {
      document.getElementById('error').style.display = 'none';
      const iconEl = document.getElementById('icon');
      const resultEl = document.getElementById('result');
      const detailEl = document.getElementById('detail');

      if (umbrella) {
        iconEl.textContent = '☂️';
        resultEl.textContent = '우산 챙기세요';
      } else {
        iconEl.textContent = '☀️';
        resultEl.textContent = '우산 필요없어요';
      }
      detailEl.textContent = `오늘 ${Math.round(min)}° / ${Math.round(max)}°, 강수확률 최대 ${Math.round(maxPrecipProb)}%`;

      showScene(category);
    }

    function renderError(message) {
      document.getElementById('icon').textContent = '⚠️';
      document.getElementById('result').textContent = '';
      document.getElementById('detail').textContent = '';
      document.getElementById('humidity-wind').textContent = '';
      document.getElementById('comment').textContent = '';
      document.getElementById('error-message').textContent = message;
      document.getElementById('error').style.display = 'block';
    }

    async function init() {
      document.getElementById('error').style.display = 'none';
      document.getElementById('icon').textContent = '⏳';
      document.getElementById('result').textContent = '날씨 확인 중...';
      document.getElementById('detail').textContent = '';
      document.getElementById('humidity-wind').textContent = '';
      document.getElementById('comment').textContent = '';
      try {
        const { lat, lon } = await getUserLocation();
        const weather = await fetchWeather(lat, lon);
        const precipProbs = getTodayWindowPrecipProbabilities(weather.hourlyTimes, weather.hourlyPrecipProb);
        const umbrella = needsUmbrella(precipProbs);
        const maxPrecipProb = precipProbs.length ? Math.max(...precipProbs) : 0;
        const code = getCurrentWeatherCode(weather.hourlyTimes, weather.hourlyWeatherCode);
        const category = weatherCodeToCategory(code);
        renderResult({ umbrella, max: weather.todayMax, min: weather.todayMin, maxPrecipProb, category });
      } catch (err) {
        const known = err instanceof Error && /[가-힣]/.test(err.message || '');
        renderError(known ? err.message : '문제가 발생했어요. 잠시 후 다시 시도해주세요.');
      }
    }

    document.getElementById('retry-btn').addEventListener('click', init);
    window.addEventListener('load', init);
  </script>
</body>
</html>
```

- [ ] **Step 2: 코드 정독 검증**

브라우저가 없는 환경이므로 파일을 처음부터 끝까지 다시 읽으며 다음을 확인한다:
- `scene-sunny`~`scene-snowy` 4개 모두 존재하고, `showScene`이 정확히 그 id들(`scene-` + category)을 조작하는지
- `renderResult`가 더 이상 `document.body.classList`를 건드리지 않고 `showScene(category)`만 호출하는지
- `#icon`, `#result`, `#detail`, `#error`, `#error-message`, `#retry-btn` id가 기존과 동일하게 유지되어 기존 로직 함수들이 그대로 동작하는지
- `#humidity-wind`, `#comment`, `#tab-umbrella`, `#tab-parasol`, `#timeline`이 HTML에 존재하지만 아직 JS에서 값이 채워지지 않는 것(다음 태스크에서 채움)이 의도된 상태인지

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Replace static background with card layout and animated weather scenes"
```

---

### Task 2: `fetchWeather` 확장 (UV·습도·바람) + 헬퍼 함수 일반화

**Files:**
- Modify: `index.html` (`<script>` 블록)

**Interfaces:**
- Consumes: Task 1의 `fetchWeather`, `getCurrentWeatherCode`, `getTodayWindowPrecipProbabilities`
- Produces:
  - `function getCurrentHourIndex(hourlyTimes: string[]): number`
  - `function getTodayWindowValues(hourlyTimes, hourlyValues, startHour = WINDOW_START_HOUR, endHour = WINDOW_END_HOUR): number[]` (기존 `getTodayWindowPrecipProbabilities`를 이름만 일반화한 것 — 구현은 동일)
  - `fetchWeather`가 반환하는 객체에 `hourlyUv`, `hourlyHumidity`, `hourlyWind` 필드 추가

- [ ] **Step 1: `getTodayWindowPrecipProbabilities`를 `getTodayWindowValues`로 이름 변경**

```javascript
function getTodayWindowPrecipProbabilities(hourlyTimes, hourlyPrecipProb, startHour = WINDOW_START_HOUR, endHour = WINDOW_END_HOUR) {
```
를
```javascript
function getTodayWindowValues(hourlyTimes, hourlyPrecipProb, startHour = WINDOW_START_HOUR, endHour = WINDOW_END_HOUR) {
```
로 바꾼다 (함수 몸통은 그대로 둔다). 그리고 `init()` 안의 호출부:
```javascript
const precipProbs = getTodayWindowPrecipProbabilities(weather.hourlyTimes, weather.hourlyPrecipProb);
```
를
```javascript
const precipProbs = getTodayWindowValues(weather.hourlyTimes, weather.hourlyPrecipProb);
```
로 바꾼다.

- [ ] **Step 2: `getCurrentHourIndex` 추가 + `getCurrentWeatherCode`가 이를 사용하도록 리팩터링**

`getCurrentWeatherCode` 함수 전체를 아래로 교체:

```javascript
function getCurrentHourIndex(hourlyTimes) {
  const now = new Date();
  const pad = (n) => String(n).padStart(2, '0');
  const nowHourStr = `${now.getFullYear()}-${pad(now.getMonth() + 1)}-${pad(now.getDate())}T${pad(now.getHours())}:00`;
  const idx = hourlyTimes.indexOf(nowHourStr);
  return idx === -1 ? 0 : idx;
}

function getCurrentWeatherCode(hourlyTimes, hourlyWeatherCode) {
  return hourlyWeatherCode[getCurrentHourIndex(hourlyTimes)];
}
```

- [ ] **Step 3: `fetchWeather`에 UV·습도·바람 요청 추가**

`fetchWeather` 함수 전체를 아래로 교체:

```javascript
async function fetchWeather(lat, lon) {
  const url = `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&hourly=precipitation_probability,weathercode,uv_index,relative_humidity_2m,wind_speed_10m&daily=temperature_2m_max,temperature_2m_min&timezone=auto`;
  let response;
  try {
    response = await fetch(url);
  } catch (e) {
    throw new Error('날씨 정보를 가져올 수 없어요. 잠시 후 다시 시도해주세요.');
  }
  if (!response.ok) {
    throw new Error('날씨 정보를 가져올 수 없어요');
  }
  const data = await response.json();
  return {
    hourlyTimes: data.hourly.time,
    hourlyPrecipProb: data.hourly.precipitation_probability,
    hourlyWeatherCode: data.hourly.weathercode,
    hourlyUv: data.hourly.uv_index,
    hourlyHumidity: data.hourly.relative_humidity_2m,
    hourlyWind: data.hourly.wind_speed_10m,
    todayMax: data.daily.temperature_2m_max[0],
    todayMin: data.daily.temperature_2m_min[0],
  };
}
```

- [ ] **Step 4: 실제 Node.js 실행으로 순수 함수 검증**

`getCurrentHourIndex`와 `getTodayWindowValues`만 스크래치 디렉터리의 임시 `.js` 파일로 추출해(프로젝트에는 커밋하지 않음) 아래를 실행하고 실제 출력을 확인한다:

```javascript
console.log(getTodayWindowValues(
  ['2026-07-31T05:00', '2026-07-31T06:00', '2026-07-31T21:00', '2026-07-31T22:00'],
  [1, 2, 3, 4]
)); // [2, 3] 기대 (기존 getTodayWindowPrecipProbabilities와 동일한 동작이어야 함)

const idx = getCurrentHourIndex(['2020-01-01T00:00']); // 실제 존재하지 않는 시간대 -> fallback
console.log(idx); // 0 기대
```

Expected: 두 출력 모두 주석의 기대값과 일치.

- [ ] **Step 5: 코드 정독 검증**

`index.html`을 다시 읽으며: `getTodayWindowPrecipProbabilities`라는 이름이 더 이상 파일 어디에도 남아있지 않은지(호출부 포함), `fetchWeather`의 반환 객체 6개 필드(`hourlyTimes`, `hourlyPrecipProb`, `hourlyWeatherCode`, `hourlyUv`, `hourlyHumidity`, `hourlyWind`, `todayMax`, `todayMin` — 총 8개)가 모두 있는지 확인한다.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Extend fetchWeather with UV/humidity/wind and generalize window/index helpers"
```

---

### Task 3: 우산/양산 탭 전환 + 양산(UV) 판단 로직

**Files:**
- Modify: `index.html` (`<script>` 블록)

**Interfaces:**
- Consumes: Task 2의 `getTodayWindowValues`, `getCurrentHourIndex`, `getCurrentWeatherCode`, `weatherCodeToCategory`, `needsUmbrella`, `fetchWeather`(신규 필드 포함), Task 1의 `showScene`, `renderError`
- Produces:
  - `const UV_THRESHOLD = 6`
  - `function needsParasol(uvValues: number[], threshold = UV_THRESHOLD): boolean`
  - `function computeUmbrellaView(weather): { icon, label, detail, category }`
  - `function computeParasolView(weather): { icon, label, detail, category }`
  - `function renderView(view): void`
  - `function setActiveTab(tab: 'umbrella'|'parasol'): void`
  - `let activeTab`, `let currentView` (모듈 스코프 상태)
  - `renderResult` 함수는 제거되고 `computeUmbrellaView` + `renderView`로 대체됨

- [ ] **Step 1: `needsParasol` 추가**

`needsUmbrella` 함수 바로 아래에 추가:

```javascript
const UV_THRESHOLD = 6;

function needsParasol(uvValues, threshold = UV_THRESHOLD) {
  if (uvValues.length === 0) return false;
  return Math.max(...uvValues) >= threshold;
}
```

- [ ] **Step 2: `renderResult`를 `computeUmbrellaView` + `computeParasolView` + `renderView`로 교체**

기존 `renderResult` 함수 전체를 지우고 그 자리에 아래 코드로 교체:

```javascript
function computeUmbrellaView(weather) {
  const precipProbs = getTodayWindowValues(weather.hourlyTimes, weather.hourlyPrecipProb);
  const umbrella = needsUmbrella(precipProbs);
  const maxPrecipProb = precipProbs.length ? Math.max(...precipProbs) : 0;
  const code = getCurrentWeatherCode(weather.hourlyTimes, weather.hourlyWeatherCode);
  const category = weatherCodeToCategory(code);
  return {
    icon: umbrella ? '☂️' : '☀️',
    label: umbrella ? '우산 챙기세요' : '우산 필요없어요',
    detail: `오늘 ${Math.round(weather.todayMin)}° / ${Math.round(weather.todayMax)}°, 강수확률 최대 ${Math.round(maxPrecipProb)}%`,
    category,
  };
}

function computeParasolView(weather) {
  const uvValues = getTodayWindowValues(weather.hourlyTimes, weather.hourlyUv);
  const parasol = needsParasol(uvValues);
  const maxUv = uvValues.length ? Math.max(...uvValues) : 0;
  const category = parasol ? 'sunny' : 'cloudy';
  return {
    icon: parasol ? '🌂' : '⛅',
    label: parasol ? '양산 챙기세요' : '양산 필요없어요',
    detail: `오늘 ${Math.round(weather.todayMin)}° / ${Math.round(weather.todayMax)}°, UV 지수 최대 ${maxUv.toFixed(1)}`,
    category,
  };
}

function renderView(view) {
  document.getElementById('error').style.display = 'none';
  document.getElementById('icon').textContent = view.icon;
  document.getElementById('result').textContent = view.label;
  document.getElementById('detail').textContent = view.detail;
  showScene(view.category);
}
```

- [ ] **Step 3: 탭 상태 관리 + 클릭 이벤트 추가**

`init()` 함수 바로 위에 추가:

```javascript
let activeTab = 'umbrella';
let currentView = null;

function setActiveTab(tab) {
  activeTab = tab;
  document.getElementById('tab-umbrella').classList.toggle('active', tab === 'umbrella');
  document.getElementById('tab-parasol').classList.toggle('active', tab === 'parasol');
  if (currentView) {
    renderView(tab === 'umbrella' ? currentView.umbrella : currentView.parasol);
  }
}
```

- [ ] **Step 4: `init()`을 새 뷰 계산 방식으로 교체**

`init()` 함수 전체를 아래로 교체:

```javascript
async function init() {
  document.getElementById('error').style.display = 'none';
  document.getElementById('icon').textContent = '⏳';
  document.getElementById('result').textContent = '날씨 확인 중...';
  document.getElementById('detail').textContent = '';
  document.getElementById('humidity-wind').textContent = '';
  document.getElementById('comment').textContent = '';
  try {
    const { lat, lon } = await getUserLocation();
    const weather = await fetchWeather(lat, lon);
    currentView = {
      umbrella: computeUmbrellaView(weather),
      parasol: computeParasolView(weather),
    };
    renderView(activeTab === 'umbrella' ? currentView.umbrella : currentView.parasol);
  } catch (err) {
    const known = err instanceof Error && /[가-힣]/.test(err.message || '');
    renderError(known ? err.message : '문제가 발생했어요. 잠시 후 다시 시도해주세요.');
  }
}
```

- [ ] **Step 5: 탭 버튼 이벤트 리스너 추가**

파일 맨 아래, 기존 `document.getElementById('retry-btn').addEventListener(...)` 줄 바로 아래에 추가:

```javascript
document.getElementById('tab-umbrella').addEventListener('click', () => setActiveTab('umbrella'));
document.getElementById('tab-parasol').addEventListener('click', () => setActiveTab('parasol'));
```

- [ ] **Step 6: 실제 Node.js 실행으로 `needsParasol` 검증**

```javascript
console.log(needsParasol([2, 4, 7], 6)); // true 기대
console.log(needsParasol([2, 4, 5], 6)); // false 기대
console.log(needsParasol([])); // false 기대
```

Expected: 세 출력 모두 주석과 일치.

- [ ] **Step 7: 코드 정독 검증**

`index.html`을 다시 읽으며: `renderResult`라는 이름이 파일에 더 이상 남아있지 않은지, `init()`이 `currentView`를 채운 뒤 `activeTab`에 맞는 뷰로 `renderView`를 호출하는지, 탭 버튼 클릭 시 `setActiveTab`이 `currentView`가 아직 `null`인 경우(데이터 로딩 전에 탭을 눌렀을 때) 에러 없이 버튼 활성 상태만 바뀌고 조용히 넘어가는지(`if (currentView)` 가드 확인) 점검한다.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "Add umbrella/parasol tab switching with UV-based parasol logic"
```

---

### Task 4: 습도·바람 + 한줄 코멘트 + 시간대별 타임라인

**Files:**
- Modify: `index.html` (`<script>` 블록)

**Interfaces:**
- Consumes: Task 2의 `getCurrentHourIndex`, `weatherCodeToCategory`, Task 3의 `computeUmbrellaView`, `computeParasolView`, `renderView`, `init`
- Produces:
  - `function getOneLineComment(category): string`
  - `function buildTimeline(hourlyTimes, hourlyWeatherCode, hours = TIMELINE_HOURS): string[]`
  - `function renderTimeline(hourlyTimes, hourlyWeatherCode): void`
  - `computeUmbrellaView`/`computeParasolView` 반환 객체에 `humidityWind`, `comment` 필드 추가
  - `renderView`가 `humidityWind`/`comment`도 표시하도록 확장

- [ ] **Step 1: 코멘트 사전 + 타임라인 상수/함수 추가**

`needsParasol` 함수 바로 아래에 추가:

```javascript
const COMMENTS = {
  sunny: '나들이하기 좋은 날씨예요',
  cloudy: '선선하게 흐린 하루예요',
  rainy: '우산 깜빡하지 마세요!',
  snowy: '눈길 조심하세요',
};

function getOneLineComment(category) {
  return COMMENTS[category] || '';
}

const TIMELINE_HOURS = [6, 9, 12, 15, 18, 21];
const CATEGORY_EMOJI = { sunny: '☀️', cloudy: '⛅', rainy: '🌧️', snowy: '❄️' };

function buildTimeline(hourlyTimes, hourlyWeatherCode, hours = TIMELINE_HOURS) {
  const todayStr = hourlyTimes[0].slice(0, 10);
  return hours.map((h) => {
    const pad = String(h).padStart(2, '0');
    const target = `${todayStr}T${pad}:00`;
    const idx = hourlyTimes.indexOf(target);
    if (idx === -1) return '–';
    return CATEGORY_EMOJI[weatherCodeToCategory(hourlyWeatherCode[idx])];
  });
}

function renderTimeline(hourlyTimes, hourlyWeatherCode) {
  document.getElementById('timeline').textContent = buildTimeline(hourlyTimes, hourlyWeatherCode).join(' ');
}
```

- [ ] **Step 2: `computeUmbrellaView`에 습도/바람/코멘트 추가**

`computeUmbrellaView` 함수 전체를 아래로 교체:

```javascript
function computeUmbrellaView(weather) {
  const precipProbs = getTodayWindowValues(weather.hourlyTimes, weather.hourlyPrecipProb);
  const umbrella = needsUmbrella(precipProbs);
  const maxPrecipProb = precipProbs.length ? Math.max(...precipProbs) : 0;
  const code = getCurrentWeatherCode(weather.hourlyTimes, weather.hourlyWeatherCode);
  const category = weatherCodeToCategory(code);
  const idx = getCurrentHourIndex(weather.hourlyTimes);
  return {
    icon: umbrella ? '☂️' : '☀️',
    label: umbrella ? '우산 챙기세요' : '우산 필요없어요',
    detail: `오늘 ${Math.round(weather.todayMin)}° / ${Math.round(weather.todayMax)}°, 강수확률 최대 ${Math.round(maxPrecipProb)}%`,
    humidityWind: `습도 ${Math.round(weather.hourlyHumidity[idx])}% · 바람 ${weather.hourlyWind[idx].toFixed(1)}m/s`,
    comment: getOneLineComment(category),
    category,
  };
}
```

- [ ] **Step 3: `computeParasolView`에 습도/바람/코멘트 추가**

`computeParasolView` 함수 전체를 아래로 교체:

```javascript
function computeParasolView(weather) {
  const uvValues = getTodayWindowValues(weather.hourlyTimes, weather.hourlyUv);
  const parasol = needsParasol(uvValues);
  const maxUv = uvValues.length ? Math.max(...uvValues) : 0;
  const category = parasol ? 'sunny' : 'cloudy';
  const idx = getCurrentHourIndex(weather.hourlyTimes);
  return {
    icon: parasol ? '🌂' : '⛅',
    label: parasol ? '양산 챙기세요' : '양산 필요없어요',
    detail: `오늘 ${Math.round(weather.todayMin)}° / ${Math.round(weather.todayMax)}°, UV 지수 최대 ${maxUv.toFixed(1)}`,
    humidityWind: `습도 ${Math.round(weather.hourlyHumidity[idx])}% · 바람 ${weather.hourlyWind[idx].toFixed(1)}m/s`,
    comment: getOneLineComment(category),
    category,
  };
}
```

- [ ] **Step 4: `renderView`가 습도/바람/코멘트도 표시하도록 확장**

`renderView` 함수 전체를 아래로 교체:

```javascript
function renderView(view) {
  document.getElementById('error').style.display = 'none';
  document.getElementById('icon').textContent = view.icon;
  document.getElementById('result').textContent = view.label;
  document.getElementById('detail').textContent = view.detail;
  document.getElementById('humidity-wind').textContent = view.humidityWind;
  document.getElementById('comment').textContent = view.comment;
  showScene(view.category);
}
```

- [ ] **Step 5: `renderError`와 `init()`에 타임라인 초기화/렌더 반영**

`renderError` 함수 전체를 아래로 교체:

```javascript
function renderError(message) {
  document.getElementById('icon').textContent = '⚠️';
  document.getElementById('result').textContent = '';
  document.getElementById('detail').textContent = '';
  document.getElementById('humidity-wind').textContent = '';
  document.getElementById('comment').textContent = '';
  document.getElementById('timeline').textContent = '';
  document.getElementById('error-message').textContent = message;
  document.getElementById('error').style.display = 'block';
}
```

`init()` 함수 전체를 아래로 교체 (Task 3에서 만든 버전에 `timeline` 초기화 한 줄과 `renderTimeline` 호출 한 줄만 추가된 것):

```javascript
async function init() {
  document.getElementById('error').style.display = 'none';
  document.getElementById('icon').textContent = '⏳';
  document.getElementById('result').textContent = '날씨 확인 중...';
  document.getElementById('detail').textContent = '';
  document.getElementById('humidity-wind').textContent = '';
  document.getElementById('comment').textContent = '';
  document.getElementById('timeline').textContent = '';
  try {
    const { lat, lon } = await getUserLocation();
    const weather = await fetchWeather(lat, lon);
    currentView = {
      umbrella: computeUmbrellaView(weather),
      parasol: computeParasolView(weather),
    };
    renderTimeline(weather.hourlyTimes, weather.hourlyWeatherCode);
    renderView(activeTab === 'umbrella' ? currentView.umbrella : currentView.parasol);
  } catch (err) {
    const known = err instanceof Error && /[가-힣]/.test(err.message || '');
    renderError(known ? err.message : '문제가 발생했어요. 잠시 후 다시 시도해주세요.');
  }
}
```

- [ ] **Step 6: 실제 Node.js 실행으로 `buildTimeline`/`getOneLineComment` 검증**

```javascript
console.log(getOneLineComment('rainy')); // '우산 깜빡하지 마세요!' 기대
console.log(getOneLineComment('unknown')); // '' 기대

console.log(buildTimeline(
  ['2026-07-31T06:00', '2026-07-31T09:00', '2026-07-31T12:00', '2026-07-31T15:00', '2026-07-31T18:00', '2026-07-31T21:00'],
  [0, 2, 61, 61, 3, 0]
)); // ['☀️','⛅','🌧️','🌧️','⛅','☀️'] 기대

console.log(buildTimeline(['2026-07-31T06:00'], [0], [6, 9])); // ['☀️','–'] 기대 (09시 데이터 없음)
```

Expected: 세 출력 모두 주석과 일치.

- [ ] **Step 7: 코드 정독 검증**

`index.html`을 다시 읽으며: `#humidity-wind`, `#comment`, `#timeline`이 로딩 시작/에러 시 모두 비워지는지, 정상 렌더 시 `renderTimeline`이 `renderView`보다 먼저 호출되는지(순서 자체는 결과에 영향 없지만 일관성 확인 목적) 점검한다.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "Add humidity/wind, one-line comment, and hourly timeline"
```

---

### Task 5: README 업데이트 + 통합 점검

**Files:**
- Modify: `README.md`

**Interfaces:**
- Consumes: 완성된 `index.html` 전체
- Produces: 없음 (문서 + 점검)

- [ ] **Step 1: `README.md` 내용을 아래로 교체**

```markdown
# 아침 우산 알리미

오늘 우산(또는 양산)을 챙겨야 할지 한눈에 알려주는 웹페이지입니다.

## 사용 방법

페이지를 열고 위치 권한을 허용하면, 현재 위치 기준으로 날씨 정보를 보여줍니다.

- **☂️ 우산 탭**: 오전 6시~오후 9시 사이 강수확률이 40% 이상이면 "우산 챙기세요", 아니면 "우산 필요없어요"
- **🌂 양산 탭**: 같은 시간대 UV 지수가 6 이상이면 "양산 챙기세요", 아니면 "양산 필요없어요"

두 탭 모두 오늘의 최고/최저 기온, 습도·바람, 날씨에 맞는 한줄 코멘트, 06~21시 시간대별 미니 타임라인을 함께 보여줍니다. 배경은 날씨(맑음/흐림/비/눈)에 따라 실제로 움직이는 일러스트 애니메이션으로 바뀝니다.

## 사용한 서비스

- 날씨 데이터: [Open-Meteo](https://open-meteo.com/) (API 키 불필요)
- 위치: 브라우저 Geolocation API

## 배포

GitHub Pages로 배포되어 있습니다: https://hayley32781.github.io/morning-umbrella-app/
```

- [ ] **Step 2: 통합 점검 체크리스트 (코드 정독)**

`index.html` 전체를 처음부터 끝까지 다시 읽으며 아래를 확인한다:
- [ ] 4개 장면(`scene-sunny`/`cloudy`/`rainy`/`snowy`) 중 정확히 하나에만 `active` 클래스가 남는 로직인지 (`showScene`의 `toggle` 사용 확인)
- [ ] 우산 탭 계산(`computeUmbrellaView`)과 양산 탭 계산(`computeParasolView`)이 서로 다른 배열(`hourlyPrecipProb` vs `hourlyUv`)을 사용하는지
- [ ] 양산 탭의 `category`가 `parasol ? 'sunny' : 'cloudy'`로만 결정되어 비/눈 장면을 절대 쓰지 않는지
- [ ] 탭 전환(`setActiveTab`)이 네트워크 재요청 없이 이미 계산된 `currentView`만 다시 그리는지
- [ ] 에러 발생 시(`renderError`) 습도·바람·코멘트·타임라인이 모두 빈 문자열로 초기화되는지
- [ ] `retry-btn` 클릭이 여전히 `init()` 전체를 재실행하는지(위치 재요청부터 다시 시작)

Expected: 모든 항목 확인 완료.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "Update README for v2 features and finish integration check"
```

---

### Task 6: GitHub Pages 재배포

**Files:**
- 없음 (배포 작업, 코드 변경 없음)

**Interfaces:**
- Consumes: 완성된 `index.html`, `README.md`
- Produces: 갱신된 배포 (기존 URL 유지: https://hayley32781.github.io/morning-umbrella-app/)

> **주의:** 이미 공개된 저장소에 업데이트를 푸시하는 작업이다. 실행 전 사용자에게 푸시해도 되는지 확인받는다.

- [ ] **Step 1: 원격 브랜치 상태 확인**

```bash
git fetch origin
git status
```

Expected: 로컬 `main`이 `origin/main`보다 앞서 있고(커밋 6개 추가됨), 충돌 없음.

- [ ] **Step 2: 사용자 확인 후 푸시**

```bash
git push origin main
```

Expected: 푸시 성공. GitHub Pages가 자동으로 재빌드를 시작한다(기존과 동일한 저장소/설정이므로 별도 설정 불필요).

- [ ] **Step 3: 빌드 완료 대기 후 배포 확인**

```bash
gh api repos/hayley32781/morning-umbrella-app/pages/builds/latest
```

Expected: `"status":"built"`이 될 때까지 몇 초 간격으로 재확인. 완료되면 브라우저로 https://hayley32781.github.io/morning-umbrella-app/ 을 열어 실제로 카드 레이아웃, 애니메이션, 탭 전환이 정상 동작하는지 최종 확인한다.

---
