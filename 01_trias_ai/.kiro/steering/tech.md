# TRIAS AI 분석 뷰어 — 기술 명세

## 프로젝트 개요

| 항목 | 내용 |
|------|------|
| 목적 | IVD 체외진단 데이터 분석 (TRIAS-3 기기) |
| 개발자 | 1인 (팀장) |
| Frontend | HTML / Vanilla JS, Chart.js, SheetJS |
| Backend | FastAPI (Python) |
| AI | AWS Bedrock — Claude Sonnet |

---

## 아키텍처

현재 standalone 버전은 단일 HTML 파일(`trias_standalone.html`)로 구성되어 있으며, 빌드 도구·번들러·서버 없이 브라우저에서 직접 실행된다.
향후 FastAPI 백엔드와 연동하는 구조로 확장 예정이다.

```
[브라우저 - Frontend]
  ├── HTML / Vanilla JS       단일 파일 SPA
  ├── Chart.js 4.4.0          (jsdelivr CDN, <script src>)
  ├── SheetJS (xlsx)          엑셀 파일 파싱 / 데이터 업로드용
  └── aws4fetch@1.0.20        (esm.sh, dynamic import — callBedrock() 호출 시점)

[Backend - FastAPI / Python]  ← 확장 예정
  ├── 데이터 수신 및 전처리
  ├── Bedrock API 프록시 (CORS 우회)
  └── 분석 결과 저장/조회

[AI - AWS Bedrock]
  └── Claude Sonnet           SigV4 인증, InvokeModel API
```

---

## 전역 상태

```js
const DATA      // 전체 측정 데이터 (하드코딩 JSON)
const BCOLORS   // 블록별 색상 {Fast1, Fast2, Fast3, Fast4, Final}
const BLOCKS    // ["Fast1","Fast2","Fast3","Fast4","Final"]
let selIdx      // 현재 선택된 측정 인덱스 (0-based)
let chCurve     // Chart.js 커브 인스턴스
let chTrend     // Chart.js 추이 인스턴스
let abortCtrl   // AbortController — 중복 AI 호출 취소
```

---

## DATA 구조

```js
DATA = {
  "기기명(시트명)": [
    {
      col: Number,          // 열 번호
      blocks: {
        Fast1: BlockData,
        Fast2: BlockData,
        Fast3: BlockData,
        Fast4: BlockData,
        Final: BlockData
      }
    },
    ...
  ]
}

BlockData = {
  date:      "YYYY.MM.DD HH:MM",
  ratio2:    Number,        // T2 / Control 면적 비율
  ctrl_area: Number,        // Control 라인 면적
  t2_area:   Number,        // T2 라인 면적
  det_FluA:  Number,        // FluA 검출값
  det_FluB:  Number,        // FluB 검출값
  item1:     "POSITIVE"|"NEGATIVE"|"INVALID"|null,  // FluA 판정
  item2:     "POSITIVE"|"NEGATIVE"|"INVALID"|null,  // FluB 판정
  curve50:   Number[50],    // 광학 신호 커브 (50포인트, 10ms 간격)
  stats: {
    baseline: Number,
    t2_peak:  Number,
    t2_pos:   Number,       // T2 피크 위치 (포인트 인덱스)
    c_peak:   Number,
    c_pos:    Number,       // Control 피크 위치
    snr_t2:   Number,
    snr_c:    Number
  }
}
```

---

## 주요 함수

### `init()`
- 앱 진입점 — `DOMContentLoaded` 시 호출
- 기기(시트) 드롭다운 생성
- 이벤트 리스너 등록 (sel-sheet, sel-block, sel-model)
- `refresh()` 호출하여 초기 렌더링

### `refresh()`
- `renderList()` + `renderCurve()` + `renderTrend()` 순서로 호출

### `renderList()`
- 현재 선택 시트·블록 기준으로 측정 목록 렌더링
- `selIdx`에 해당하는 항목에 `.sel` 클래스 적용
- 항목 클릭 → `selIdx` 변경 → `renderList()` + `renderCurve()` 재호출

### `renderCurve()`
- 선택된 측정(`selIdx`)의 Fast1~Final 커브 오버레이
- `chCurve` 기존 인스턴스 파괴 후 재생성
- X축: 0~490ms (10ms 간격 × 50포인트)
- Y축: 광학 신호 강도

### `renderTrend()`
- 현재 시트의 모든 측정을 시계열로 시각화
- 선택 블록 기준 `ratio2`(좌축) + `ctrl_area`(우축) 이중 Y축 차트
- `chTrend` 기존 인스턴스 파괴 후 재생성

### `buildPrompt(type)`
| type | 포함 데이터 |
|------|------------|
| `"curve"` | 선택 측정의 모든 블록 커브·stats·판정 결과 |
| `"trend"` | 현재 시트 전체 측정의 ratio2, ctrl_area, 판정 시계열 |
| `"compare"` | 모든 시트(기기)의 Final 블록 요약 비교 |
| `"anomaly"` | 전체 시트·블록의 ratio2, ctrl_area, det_FluA/B, item1/2 |

### `callBedrock(prompt)` — `async`
1. AWS Key/Secret/Region/Model ID DOM에서 읽기
2. `import("https://esm.sh/aws4fetch@1.0.20")` 동적 로드
3. `AwsClient` 인스턴스 생성 (service: `"bedrock"`)
4. 모델별 요청 body 분기:
   - **Claude**: `anthropic_version`, `max_tokens: 1500`, `system: SYSTEM_PROMPT`
   - **Nova**: `messages`, `inferenceConfig.maxNewTokens: 1500`
5. `POST https://bedrock-runtime.{region}.amazonaws.com/model/{modelId}/invoke`
6. 응답 파싱:
   - Claude: `data.content[0].text`
   - Nova: `data.output.message.content[0].text`

### `run(type)` — `async`
1. 기존 `abortCtrl` 취소 후 새 `AbortController` 생성
2. 로딩 바 표시
3. `buildPrompt(type)` → `callBedrock()` 호출
4. 결과를 `md()` 마크다운 파서로 변환하여 DOM 삽입
5. `type === "anomaly"` 이면 `renderAnomalyTable()` 추가 호출
6. 오류 시 인라인 에러 메시지 표시

### `md(text)`
- 경량 마크다운 → HTML 변환 (별도 라이브러리 없음)
- 지원 규칙: `## → h2`, `### → h3`, `**bold**`, `- 항목`, 빈줄 → `<p>`

### `renderAnomalyTable(anomalies)`
- AI가 반환한 이상치 목록을 파싱하여 테이블로 렌더링
- `.anomaly-panel` 요소 표시/숨김 제어

---

## AWS Bedrock 연동

| 항목 | 값 |
|------|-----|
| 인증 방식 | AWS SigV4 (aws4fetch 라이브러리) |
| IAM 권한 | `bedrock:InvokeModel` |
| API 엔드포인트 | `bedrock-runtime.{region}.amazonaws.com` |
| 지원 리전 | us-east-1 / us-west-2 / ap-northeast-1 |
| 키 저장 | 브라우저 메모리만 (localStorage 미사용) |
| 스트리밍 | 미사용 (단건 invoke) |

---

## UI 레이아웃

```
<header>  로고 + 앱 제목
<div.layout>  CSS Grid (270px 사이드바 | 1fr 메인)
  <div.sidebar>
    sb-card: 데이터 선택 (시트 + 블록 드롭다운)
    sb-card: 측정 목록 (스크롤 리스트)
    sb-card: AWS Bedrock 설정 (Key/Secret/Region/Model)
    sb-card: 추가 질문 (textarea)
    sb-card: AI 분석 실행 버튼 4종
  <div.main>
    div.loading        2px 그라데이션 진행 바
    div.panel          AI 분석 결과
    div.chart-row      커브 차트 | 추이 차트 (1:1 그리드)
    div.anomaly-panel  이상치 테이블 (anomaly 실행 시 표시)
```

---

## 코딩 컨벤션

- 변수/함수: camelCase
- 상수: UPPER_SNAKE_CASE (`DATA`, `BCOLORS`, `BLOCKS`, `SYSTEM_PROMPT`)
- CSS: BEM 불사용 — 짧은 시맨틱 클래스명 (`sb-card`, `ai-btn`, `meas-item` 등)
- 들여쓰기: 2스페이스
- 세미콜론: 생략 (ASI 의존)
- 함수 선언: `function` 키워드 (호이스팅 활용)
- async/await 사용, `try/catch`로 에러 처리

---

## 확장 시 주의사항

- **데이터 추가**: `DATA` 객체에 시트를 추가하면 드롭다운에 자동 반영됨. SheetJS로 엑셀 파일을 파싱하여 동적으로 `DATA`를 채우는 방식으로 확장 가능
- **모델 추가**: `#sel-model` `<option>` 추가 + `callBedrock()` 내 `isNova` 분기 확인
- **차트 파괴**: `chCurve?.destroy()` 를 반드시 호출한 뒤 재생성할 것 (메모리 누수 방지)
- **CORS**: 브라우저 → Bedrock 직접 호출은 CORS 정책으로 차단될 수 있음. FastAPI 백엔드를 프록시로 활용하는 것이 권장 방향
- **FastAPI 연동 시**: `callBedrock()` 를 FastAPI 엔드포인트 호출로 교체. AWS 자격증명은 서버 환경변수로 이동하여 브라우저 노출 제거
- **SheetJS 도입 시**: `<input type="file">` → `XLSX.read()` → `DATA` 객체 변환 파이프라인 구성 필요
- **단일 파일 원칙**: 현재 standalone 구조에서 CSS/JS는 인라인 유지. FastAPI 연동 버전 전환 시 별도 정적 파일로 분리
