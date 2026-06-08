# TRIAS AI 분석 뷰어 — 프로젝트 구조

## 파일 구조

```
02_trias_ai_bed_rock/
├── trias_standalone.html       # 앱 전체 (CSS + HTML + JS 단일 파일)
└── .kiro/
    ├── steering/               # Kiro 스티어링 문서
    │   ├── product.md
    │   ├── tech.md
    │   ├── structure.md
    │   ├── project-overview.md
    │   └── korean-language.md
    └── specs/                  # Kiro 스펙 문서
        └── xlsx-upload/
```

## trias_standalone.html 내부 구조

단일 파일이지만 논리적으로 세 영역으로 구분된다.

### 1. `<style>` — 인라인 CSS
- 전체 레이아웃, 컴포넌트 스타일 포함
- 외부 CSS 파일 없음

### 2. HTML 마크업
```
<header>                   로고 + 앱 제목
<div.parse-overlay>        xlsx 파싱 중 오버레이 (진행 바)
<div.layout>               CSS Grid: 270px 사이드바 | 1fr 메인
  <div.sidebar>
    .sb-card: 데이터 업로드  (xlsx 드롭존, 진행 바, 초기화 버튼)
    .sb-card: 데이터 선택    (기기/시트 드롭다운, 블록 드롭다운)
    .sb-card: 측정 선택      (커브 분석용 측정 목록)
    .sb-card: AWS 설정       (Key/Secret/Region/Model)
    .sb-card: 추가 질문      (textarea)
    .sb-card: AI 실행 버튼   (이상치/커브/추이/비교 4종)
  <div.main>
    .loading                2px 그라데이션 진행 바
    .panel                  AI 분석 결과 출력
    .chart-row              커브 차트 + 추이 차트 (1:1 그리드)
    .anomaly-panel          이상치 탐지 테이블 (anomaly 시 표시)
```

### 3. `<script>` — 전역 상태 및 함수

**전역 상태**
```js
const DATA        // 측정 데이터 객체 { "기기명": [ {col, blocks:{...}} ] }
const BCOLORS     // 블록별 색상 { Fast1, Fast2, Fast3, Fast4, Final }
const BLOCKS      // ["Fast1","Fast2","Fast3","Fast4","Final"]
let selIdx        // 현재 선택된 측정 인덱스 (0-based)
let chCurve       // Chart.js 커브 인스턴스
let chTrend       // Chart.js 추이 인스턴스
let abortCtrl     // AbortController (중복 AI 호출 취소)
```

**함수 목록**

| 함수 | 역할 |
|------|------|
| `init()` | 앱 진입점, 드롭다운 생성, 이벤트 리스너 등록 |
| `refresh()` | renderList + renderCurve + renderTrend 순차 호출 |
| `renderList()` | 측정 목록 렌더링, `.sel` 클래스 적용 |
| `renderCurve()` | Fast1~Final 커브 오버레이 차트 (chCurve 재생성) |
| `renderTrend()` | ratio2 & ctrl_area 이중 Y축 시계열 차트 (chTrend 재생성) |
| `buildPrompt(type)` | type별 AI 프롬프트 생성 (curve/trend/compare/anomaly) |
| `callBedrock(prompt)` | SigV4 서명 → Bedrock InvokeModel API 호출 |
| `run(type)` | AI 분석 실행 진입점, 로딩 바 제어, 결과 DOM 삽입 |
| `md(text)` | 경량 마크다운 → HTML 변환 (자체 구현) |
| `renderAnomalyTable(anomalies)` | 이상치 테이블 렌더링 |
| `resetData()` | 업로드 데이터 초기화, 하드코딩 DATA로 복원 |

## DATA 객체 구조

```js
DATA = {
  "기기명(시트명)": [
    {
      col: Number,
      blocks: {
        Fast1: BlockData,
        Fast2: BlockData,
        Fast3: BlockData,
        Fast4: BlockData,
        Final: BlockData
      }
    }
  ]
}

BlockData = {
  date, ratio2, ctrl_area, t2_area,
  det_FluA, det_FluB,
  item1, item2,          // POSITIVE | NEGATIVE | INVALID | null
  curve50: Number[50],   // 광학 신호 (10ms × 50pt)
  stats: { baseline, t2_peak, t2_pos, c_peak, c_pos, snr_t2, snr_c }
}
```
