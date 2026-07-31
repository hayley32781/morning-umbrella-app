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

GitHub Pages로 배포되어 있습니다: https://hayley32781.github.io/morning-umbrella-app/
