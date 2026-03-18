# 💧 Water_Watch

> 전국 하수 경로 탐색 · 수질 등급 시각화 · 침수 위험 알림 통합 플랫폼  
> 순수 정적 HTML — 서버 없음 · 호스팅 비용 $0

[![Live Demo](https://img.shields.io/badge/Live-GitHub%20Pages-00e5c0?style=flat-square&logo=github)](https://Fullcharge13.github.io/Water_Watch)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue?style=flat-square)](LICENSE)
[![Cost](https://img.shields.io/badge/Hosting-$0%2Fmo-brightgreen?style=flat-square)](https://pages.github.com)

---

## 📸 스크린샷

| 전국 탐색기 | 수질·침수 대시보드 |
|---|---|
| 픽셀 도트 맵 + 시도별 경로 탐색 | BOD 등급 시각화 + 강수 위험도 |

---

## ✨ 주요 기능

| 기능 | 파일 | 비용 |
|------|------|------|
| 전국 하수 경로 탐색 (230+ 시군구) | `index.html` | $0 |
| 픽셀 도트 한반도 지도 | `sewage_national.html` | $0 |
| 수질 등급 시각화 (5대 수계 BOD/DO/TN/TP) | `water_dashboard.html` | $0 |
| 침수 위험 알림 (강수량 × 관로 노후도) | `water_dashboard.html` | $0 |
| AI 교육 퀴즈 — 레벨별 문제 생성 | `seweredu_quiz.html` | ~$3/월 |

---

## 🗂 프로젝트 구조

```
Water_Watch/
├── index.html               # 메인 진입점 — 전국 하수 경로 탐색기
├── water_dashboard.html     # 수질 등급 + 침수 위험 통합 대시보드
├── sewage_national.html     # 전국 픽셀 도트 맵 (17개 시도)
├── seoul_sewage_mvp.html    # 서울 특화 MVP
├── seweredu_quiz.html       # AI 교육 퀴즈 (Claude API 연동)
├── docs/
│   ├── api_guide.md         # 공공 API 연동 가이드
│   └── data_sources.md      # 데이터 출처 목록
├── .gitignore
├── LICENSE
└── README.md
```

---

## 🚀 빠른 시작

### 로컬 실행

```bash
# 클론
git clone https://github.com/your-username/Water_Watch.git
cd Water_Watch

# 서버 없이 바로 열기
open index.html

# 또는 간단한 로컬 서버
python3 -m http.server 8000
# → http://localhost:8000
```

### API 키 설정 (선택)

각 HTML 파일 상단의 상수를 교체하면 실데이터로 전환됩니다.

```javascript
// water_dashboard.html
const KMA_KEY   = 'YOUR_기상청_API_KEY';   // 강수량 실시간
const WQ_KEY    = 'YOUR_환경부_API_KEY';   // 수질 측정
const KAKAO_KEY = 'YOUR_카카오_REST_KEY';  // 주소 → 좌표

// seweredu_quiz.html
const CLAUDE_API_KEY = 'YOUR_ANTHROPIC_KEY'; // AI 퀴즈
```

> API 키는 절대 커밋하지 마세요 — `.gitignore`에 `config.local.js` 등록 필요

---

## 🔧 기술 스택

- **Frontend** : 순수 HTML5 / CSS3 / Vanilla JS (프레임워크 없음)
- **지도 렌더링** : Canvas API 자체 픽셀 도트 드로잉
- **경로 탐색** : 순수 JS DiGraph BFS (NetworkX 로직 포팅)
- **AI 퀴즈** : Anthropic Claude API (`claude-sonnet-4-20250514`)
- **배포** : GitHub Pages (정적 파일, 무료)

---

## 📡 연동 공공 데이터 API

| 기관 | API | 용도 | 비용 |
|------|-----|------|------|
| 환경부 | 물환경정보시스템 수질측정망 | BOD·DO·TN·TP | 무료 |
| 기상청 | 동네예보 조회서비스 | 강수량 실시간·예보 | 무료 |
| Open-Meteo | Forecast API | 강수 예측 (키 불필요) | 무료 |
| 한국환경공단 | 공공하수처리시설 현황 | 시설 위치·용량 | 무료 |
| 카카오 | 주소 검색 API | 도로명→시군구 변환 | 월 300만 콜 무료 |

---

## 🗺 커버리지

- 전국 **17개 시도** / **230개 이상 시군구**
- **5대 수계** : 한강 · 낙동강 · 금강 · 영산강 · 섬진강
- 서울 **4개 물재생센터** 관할구역 상세 매핑

---

## 📅 릴리즈 이력

| 버전 | 날짜 | 주요 변경 |
|------|------|-----------|
| v1.0.0 | 2026-03 | 초기 릴리즈 — 전국 탐색기 + 수질·침수 대시보드 |

---

## 🤝 기여 방법

```bash
# 1. Fork 후 브랜치 생성
git checkout -b feat/새기능명

# 2. 수정 후 커밋
git commit -m "feat: 설명"

# 3. Pull Request
```

---

## 📄 라이선스

MIT License — 공공데이터 기반, 자유롭게 활용 가능  
데이터 출처: 환경부 · 한국환경공단 · 기상청 · 서울물재생시설공단

---

## 📬 문의

Issues 탭에 남겨주세요.
