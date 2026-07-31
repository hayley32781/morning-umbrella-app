# 아침 우산 알리미 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 브라우저 위치를 기반으로 오늘 우산이 필요한지 알려주는 단일 HTML 페이지를 만들고 GitHub Pages에 배포한다.

**Architecture:** 빌드 도구 없는 순수 HTML/CSS/JS 단일 파일(`index.html`). 브라우저 Geolocation API로 위치를 얻고, 키 없이 쓸 수 있는 Open-Meteo API로 날씨를 가져와, 시간대(06:00~21:00) 강수확률 최댓값을 임계값과 비교해 우산 필요 여부를 판단하고 화면에 렌더링한다.

**Tech Stack:** HTML5, CSS3, Vanilla JavaScript (ES2017+ `async/await`), Open-Meteo Forecast API, `navigator.geolocation`, GitHub Pages.

## Global Constraints

- 파일은 `index.html` 하나로 구성한다. 빌드 도구, 프레임워크, npm 패키지를 사용하지 않는다.
- 날씨 데이터는 Open-Meteo Forecast API(`https://api.open-meteo.com/v1/forecast`)를 사용한다. API 키는 필요 없다.
- 위치는 `navigator.geolocation.getCurrentPosition()`으로 자동 감지한다. 수동 지역 검색/입력 UI는 만들지 않는다.
- 우산 필요 여부 판단 시간대는 **오전 6시(06:00) ~ 오후 9시(21:00)**, 이 시간대의 시간별 강수확률 중 **최댓값이 40% 이상**이면 "우산 필요"로 판단한다. 두 값(시간대, 임계값)은 코드 상단 상수로 분리해 조정 가능하게 한다.
- 자동 테스트 코드(pytest, jest 등)는 작성하지 않는다 — spec에서 시간 관계상 명시적으로 생략하기로 했다. 각 작업 뒤에는 브라우저로 직접 열어 눈으로 확인하는 "수동 확인" 절차로 대체한다.
- 최종 결과물은 GitHub Pages로 배포해 공개 URL을 통해 접근 가능해야 한다.
- 참고 spec: `docs/superpowers/specs/2026-07-31-morning-umbrella-app-design.md`

---

### Task 1: 정적 뼈대 (HTML + CSS)

**Files:**
- Create: `index.html`

**Interfaces:**
- Consumes: 없음 (첫 작업)
- Produces: 다음 DOM 요소 id들 — `icon`, `result`, `detail`, `error`, `error-message`, `retry-btn`. `body`에 날씨별 배경색을 위한 CSS 클래스 `sunny`, `cloudy`, `rainy`, `snowy` (기본값 `cloudy`).

- [ ] **Step 1: `index.html` 파일 생성 (HTML 구조 + CSS)**

```html
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>아침 우산 알리미</title>
<style>
  :root {
    --bg-sunny: #fdf6e3;
    --bg-cloudy: #e9edf1;
    --bg-rainy: #cfd8e3;
    --bg-snowy: #eef3f7;
  }
  * { box-sizing: border-box; }
  body {
    margin: 0;
    min-height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    font-family: -apple-system, "Apple SD Gothic Neo", "Malgun Gothic", sans-serif;
    background: var(--bg-cloudy);
    transition: background-color 0.4s ease;
  }
  body.sunny { background: var(--bg-sunny); }
  body.cloudy { background: var(--bg-cloudy); }
  body.rainy { background: var(--bg-rainy); }
  body.snowy { background: var(--bg-snowy); }
  .card { text-align: center; padding: 2rem; }
  #icon { font-size: 5rem; }
  #result { font-size: 2rem; font-weight: 700; margin: 0.5rem 0; }
  #detail { font-size: 1.1rem; color: #444; }
  #error { display: none; font-size: 1.2rem; margin-top: 1rem; }
  #retry-btn {
    margin-top: 1rem;
    padding: 0.6rem 1.2rem;
    font-size: 1rem;
    border: none;
    border-radius: 8px;
    background: #333;
    color: white;
    cursor: pointer;
  }
</style>
</head>
<body class="cloudy">
  <div class="card">
    <div id="icon">⏳</div>
    <div id="result">날씨 확인 중...</div>
    <div id="detail"></div>
    <div id="error">
      <p id="error-message"></p>
      <button id="retry-btn">다시 시도</button>
    </div>
  </div>
  <script>
    // 이후 작업에서 여기에 로직을 추가한다
  </script>
</body>
</html>
```

- [ ] **Step 2: 수동 확인**

`index.html`을 더블클릭해 브라우저로 연다.
Expected: 화면 가운데에 "⏳ 날씨 확인 중..." 카드가 옅은 회색(cloudy) 배경 위에 표시된다. 에러 영역과 버튼은 보이지 않는다.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Add static skeleton for umbrella reminder page"
```

---

### Task 2: 위치 가져오기 + 오류 표시

**Files:**
- Modify: `index.html` (`<script>` 블록)

**Interfaces:**
- Consumes: Task 1의 DOM id들 (`icon`, `result`, `error`, `error-message`, `retry-btn`)
- Produces:
  - `function getUserLocation(): Promise<{lat: number, lon: number}>`
  - `function renderError(message: string): void`
  - `async function init(): Promise<void>` (이 단계에서는 위치까지만 확인하는 임시 버전)

- [ ] **Step 1: `<script>` 블록 내용을 아래로 교체**

`<script>` ~ `</script>` 사이의 주석을 지우고 아래 코드로 채운다:

```html
<script>
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

  function renderError(message) {
    document.getElementById('icon').textContent = '⚠️';
    document.getElementById('result').textContent = '';
    document.getElementById('detail').textContent = '';
    document.getElementById('error-message').textContent = message;
    document.getElementById('error').style.display = 'block';
  }

  async function init() {
    document.getElementById('error').style.display = 'none';
    document.getElementById('icon').textContent = '⏳';
    document.getElementById('result').textContent = '날씨 확인 중...';
    document.getElementById('detail').textContent = '';
    try {
      const { lat, lon } = await getUserLocation();
      // 임시: 다음 작업에서 fetchWeather로 교체된다
      document.getElementById('result').textContent = `위치 확인됨: ${lat.toFixed(2)}, ${lon.toFixed(2)}`;
    } catch (err) {
      renderError(err.message || '문제가 발생했어요. 잠시 후 다시 시도해주세요.');
    }
  }

  document.getElementById('retry-btn').addEventListener('click', init);
  window.addEventListener('load', init);
</script>
```

- [ ] **Step 2: 수동 확인 — 위치 허용 시나리오**

브라우저에서 `index.html`을 새로고침하고 위치 권한 요청에 "허용"을 누른다.
Expected: 화면에 "위치 확인됨: 37.56, 126.98" 형태의 문구(실제 좌표)가 표시된다.

- [ ] **Step 3: 수동 확인 — 위치 거부 시나리오**

브라우저 주소창 옆 자물쇠 아이콘에서 위치 권한을 "차단"으로 바꾼 뒤 새로고침하거나, 권한 팝업에서 "차단"을 누른다.
Expected: "⚠️" 아이콘과 "위치 정보가 필요해요" 문구, "다시 시도" 버튼이 표시된다. 버튼을 누르면 다시 권한 요청이 뜬다 (또는 브라우저 설정에 따라 즉시 재실패).

테스트 후 위치 권한을 다시 "허용"으로 되돌려 놓는다.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add geolocation fetch and error UI"
```

---

### Task 3: 날씨 데이터 가져오기 (Open-Meteo)

**Files:**
- Modify: `index.html` (`<script>` 블록)

**Interfaces:**
- Consumes: Task 2의 `getUserLocation()`, `renderError()`, `init()`
- Produces:
  - `async function fetchWeather(lat: number, lon: number): Promise<{hourlyTimes: string[], hourlyPrecipProb: number[], hourlyWeatherCode: number[], todayMax: number, todayMin: number}>`

- [ ] **Step 1: `fetchWeather` 함수 추가**

`getUserLocation` 함수 바로 아래에 추가:

```javascript
  async function fetchWeather(lat, lon) {
    const url = `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&hourly=precipitation_probability,weathercode&daily=temperature_2m_max,temperature_2m_min&timezone=auto`;
    const response = await fetch(url);
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
```

- [ ] **Step 2: `init()`의 임시 표시 코드를 `fetchWeather` 호출로 교체**

`init()` 안의 `// 임시: ...` 줄과 그 아래 문구 표시 줄을 아래로 교체:

```javascript
      const { lat, lon } = await getUserLocation();
      const weather = await fetchWeather(lat, lon);
      // 임시: 다음 작업에서 판단 로직 결과로 교체된다
      document.getElementById('result').textContent =
        `오늘 최고 ${weather.todayMax}° / 최저 ${weather.todayMin}°`;
```

- [ ] **Step 3: 수동 확인**

브라우저에서 새로고침 후 위치 허용.
Expected: "오늘 최고 XX° / 최저 XX°" 형태로 실제 오늘 기온이 표시된다 (브라우저 개발자 도구 Network 탭에서 `api.open-meteo.com` 요청이 200 OK로 성공한 것도 확인).

- [ ] **Step 4: 수동 확인 — 네트워크 실패 시나리오**

개발자 도구 Network 탭에서 "오프라인" 모드로 전환 후 새로고침.
Expected: "⚠️" + "날씨 정보를 가져올 수 없어요" 문구가 표시된다. 확인 후 오프라인 모드를 해제한다.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Fetch weather data from Open-Meteo"
```

---

### Task 4: 우산 판단 로직 (순수 함수)

**Files:**
- Modify: `index.html` (`<script>` 블록)

**Interfaces:**
- Consumes: Task 3의 `fetchWeather()` 반환 형태 (`hourlyTimes`, `hourlyPrecipProb`, `hourlyWeatherCode`)
- Produces:
  - `const THRESHOLD_PERCENT = 40`
  - `const WINDOW_START_HOUR = 6`
  - `const WINDOW_END_HOUR = 21`
  - `function getTodayWindowPrecipProbabilities(hourlyTimes, hourlyPrecipProb, startHour = WINDOW_START_HOUR, endHour = WINDOW_END_HOUR): number[]`
  - `function needsUmbrella(precipProbs, threshold = THRESHOLD_PERCENT): boolean`
  - `function getCurrentWeatherCode(hourlyTimes, hourlyWeatherCode): number`
  - `function weatherCodeToCategory(code): 'sunny' | 'cloudy' | 'rainy' | 'snowy'`

- [ ] **Step 1: 상수 + 4개 함수 추가**

`<script>` 블록 맨 앞, `getUserLocation` 함수 위에 추가:

```javascript
  const THRESHOLD_PERCENT = 40;
  const WINDOW_START_HOUR = 6;
  const WINDOW_END_HOUR = 21;

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
```

- [ ] **Step 2: 수동 확인 — 브라우저 콘솔에서 샘플 데이터로 검증**

`index.html`을 열고 개발자 도구 콘솔에서 아래를 붙여넣어 실행:

```javascript
console.log(needsUmbrella([10, 20, 45, 5], 40)); // true 기대
console.log(needsUmbrella([10, 20, 30], 40)); // false 기대
console.log(weatherCodeToCategory(0)); // "sunny" 기대
console.log(weatherCodeToCategory(63)); // "rainy" 기대
console.log(weatherCodeToCategory(73)); // "snowy" 기대
console.log(getTodayWindowPrecipProbabilities(
  ['2026-07-31T05:00', '2026-07-31T06:00', '2026-07-31T21:00', '2026-07-31T22:00', '2026-08-01T06:00'],
  [1, 2, 3, 4, 5]
)); // [2, 3] 기대 (05시, 22시, 다음날은 제외)
```

Expected: 콘솔에 주석에 적힌 기대값이 그대로 출력된다.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Add umbrella decision logic as pure functions"
```

---

### Task 5: 결과 렌더링 + 전체 흐름 연결

**Files:**
- Modify: `index.html` (`<script>` 블록)

**Interfaces:**
- Consumes: Task 2~4의 모든 함수 (`getUserLocation`, `fetchWeather`, `getTodayWindowPrecipProbabilities`, `needsUmbrella`, `getCurrentWeatherCode`, `weatherCodeToCategory`, `renderError`), Task 1의 DOM id들과 CSS 클래스
- Produces:
  - `function renderResult({ umbrella, max, min, maxPrecipProb, category }): void`
  - 최종 완성된 `init()`

- [ ] **Step 1: `renderResult` 함수 추가**

`renderError` 함수 바로 위에 추가:

```javascript
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

    document.body.classList.remove('sunny', 'cloudy', 'rainy', 'snowy');
    document.body.classList.add(category);
  }
```

- [ ] **Step 2: `init()`의 임시 코드를 최종 로직으로 교체**

`init()` 안의 `// 임시: ...` 줄과 그 아래 `document.getElementById('result').textContent = ...` 줄을 아래로 교체:

```javascript
      const { lat, lon } = await getUserLocation();
      const weather = await fetchWeather(lat, lon);
      const precipProbs = getTodayWindowPrecipProbabilities(weather.hourlyTimes, weather.hourlyPrecipProb);
      const umbrella = needsUmbrella(precipProbs);
      const maxPrecipProb = precipProbs.length ? Math.max(...precipProbs) : 0;
      const code = getCurrentWeatherCode(weather.hourlyTimes, weather.hourlyWeatherCode);
      const category = weatherCodeToCategory(code);
      renderResult({ umbrella, max: weather.todayMax, min: weather.todayMin, maxPrecipProb, category });
```

- [ ] **Step 3: 수동 확인 — 전체 흐름**

브라우저에서 새로고침 후 위치 허용.
Expected: "☂️ 우산 챙기세요" 또는 "☀️ 우산 필요없어요"가 표시되고, 그 아래 "오늘 XX° / XX°, 강수확률 최대 XX%"가 보인다. 배경색이 현재 날씨(맑음/흐림/비/눈)에 맞게 바뀐다.

- [ ] **Step 4: 수동 확인 — 임계값 임시 조정으로 반대 케이스 확인**

콘솔에서 `THRESHOLD_PERCENT`는 `const`라 재할당은 안 되므로, 대신 `needsUmbrella([100], 40)`과 `needsUmbrella([0], 40)`을 콘솔에서 직접 호출해 두 결과 문구 스타일(아이콘/문구/색)이 각각 정상적으로 나오는지 `renderResult`에 직접 값을 넣어 확인:

```javascript
renderResult({ umbrella: true, max: 30, min: 22, maxPrecipProb: 80, category: 'rainy' });
```
```javascript
renderResult({ umbrella: false, max: 30, min: 22, maxPrecipProb: 10, category: 'sunny' });
```

Expected: 두 호출 모두 화면이 각각 "우산 챙기세요"(rainy 배경)와 "우산 필요없어요"(sunny 배경)로 올바르게 바뀐다.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Wire up result rendering and finish end-to-end flow"
```

---

### Task 6: README 작성 + 통합 점검

**Files:**
- Create: `README.md`

**Interfaces:**
- Consumes: 전체 완성된 `index.html`
- Produces: 없음 (문서 + 점검)

- [ ] **Step 1: `README.md` 작성**

```markdown
# 아침 우산 알리미

오늘 우산을 챙겨야 할지 한눈에 알려주는 웹페이지입니다.

## 사용 방법

`index.html`을 브라우저로 열고 위치 권한을 허용하면, 현재 위치 기준으로
오전 6시~오후 9시 사이 강수확률이 40% 이상이면 "우산 챙기세요"를,
아니면 "우산 필요없어요"를 보여줍니다. 오늘의 최고/최저 기온과
최대 강수확률도 함께 표시됩니다.

## 사용한 서비스

- 날씨 데이터: [Open-Meteo](https://open-meteo.com/) (API 키 불필요)
- 위치: 브라우저 Geolocation API

## 배포

GitHub Pages로 배포되어 있습니다: (Task 7에서 URL 추가 예정)
```

- [ ] **Step 2: 통합 점검 체크리스트 실행**

아래 항목을 브라우저에서 순서대로 확인한다:
- [ ] 위치 허용 시 결과가 정상 표시되는가
- [ ] 위치 거부 시 안내 문구 + 다시 시도 버튼이 표시되는가
- [ ] 오프라인 상태에서 API 실패 안내 문구가 표시되는가
- [ ] 최고/최저 기온과 강수확률 숫자가 실제 값과 일치하는가 (참고로 아무 날씨 앱과 비교)
- [ ] 배경색이 날씨 상태에 맞게 바뀌는가

Expected: 모든 항목이 정상 동작.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "Add README and finish integration check"
```

---

### Task 7: GitHub Pages 배포

**Files:**
- 없음 (배포 작업, 코드 변경 없음)

**Interfaces:**
- Consumes: 완성된 `index.html`, `README.md`
- Produces: 공개 URL (예: `https://<사용자명>.github.io/<저장소명>/`)

> **주의:** 이 작업은 GitHub에 새 저장소를 만들고 코드를 공개로 푸시하는, 외부에 공개되는 되돌리기 번거로운 작업이다. 실행 전 사용자에게 저장소 이름과 공개 여부를 확인받는다.

- [ ] **Step 1: GitHub CLI 로그인 상태 확인**

```bash
gh auth status
```

Expected: 로그인된 계정 정보가 출력된다. 로그인이 안 되어 있다면 사용자에게 `gh auth login` 진행 여부를 먼저 확인한다.

- [ ] **Step 2: 사용자에게 저장소 이름 확인 후 GitHub 저장소 생성**

사용자 확인을 받은 이름(예: `morning-umbrella-app`)으로:

```bash
gh repo create <저장소명> --public --source=. --remote=origin
```

- [ ] **Step 3: 커밋 푸시**

```bash
git push -u origin master
```

- [ ] **Step 4: GitHub Pages 활성화**

```bash
gh api repos/:owner/:repo/pages -X POST -f "source[branch]=master" -f "source[path]=/"
```

Expected: 명령이 성공하면 Pages 사이트가 빌드를 시작한다. 몇 분 후 `https://<사용자명>.github.io/<저장소명>/`에서 접속 가능해진다.

- [ ] **Step 5: 배포 확인**

브라우저로 생성된 Pages URL에 접속해 실제 서비스와 동일하게 동작하는지 확인.
Expected: 로컬에서 확인했던 것과 동일하게 우산 결과가 표시된다.

- [ ] **Step 6: README에 배포 URL 반영 + 커밋**

`README.md`의 "(Task 7에서 URL 추가 예정)" 부분을 실제 URL로 교체한다.

```bash
git add README.md
git commit -m "Add deployed Pages URL to README"
git push
```

---
