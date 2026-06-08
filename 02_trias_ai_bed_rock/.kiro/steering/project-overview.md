# TRIAS AI 분석 뷰어 — 프로젝트 개요

## 프로젝트 설명

`trias_standalone.html` 은 **TRIAS IVD(체외진단) 장비**의 측정 데이터를 시각화하고 AWS Bedrock AI로 분석하는 **단일 파일 웹 애플리케이션**이다.
외부 서버 없이 브라우저에서 직접 실행되며, AWS Bedrock API를 직접 호출한다.

## 기술 스택

| 항목 | 내용 |
|------|------|
| 런타임 | 브라우저 (서버 없음) |
| 차트 | Chart.js 4.4.0 (CDN) |
| AI API | AWS Bedrock — Claude Sonnet, Amazon Nova |
| AWS 인증 | aws4fetch@1.0.20 (esm.sh CDN, SigV4 서명) |
| 스타일 | 인라인 CSS (외부 프레임워크 없음) |
| 언어 | HTML + Vanilla JS (ES2020+) |

## 도메인 지식

- **TRIAS**: 면역크로마토그래피 방식 IVD 장비 (FluA / FluB 인플루엔자 검출)
- **측정 블록**: Fast1, Fast2, Fast3, Fast4, Final (순서대로 실행되는 분석 단계)
- **핵심 지표**:
  - `ratio2`: T2 영역 / Control 영역 비율 — 양성 판정 기준값
  - `ctrl_area`: Control 라인 면적 — 유효성 지표
  - `t2_area`: T2 라인 면적
  - `det_FluA` / `det_FluB`: Flu 검출값
  - `curve50`: 50포인트 광학 신호 커브 배열
  - `snr_t2` / `snr_c`: Signal-to-Noise Ratio
- **판정 결과**: `item1` (FluA), `item2` (FluB) — POSITIVE / NEGATIVE / INVALID / null

## 파일 구조

```
trias_standalone.html
├── <style>          인라인 CSS — 레이아웃, 컴포넌트 스타일
├── HTML 마크업      사이드바(입력) + 메인 영역(결과/차트)
└── <script>
    ├── DATA         하드코딩된 측정 데이터 (JSON)
    ├── init()       앱 초기화, 드롭다운 렌더링
    ├── renderList() 기기별 측정 목록 표시
    ├── renderCurve() Chart.js 커브 오버레이 차트
    ├── renderTrend() ratio2 & ctrl_area 추이 차트
    ├── buildPrompt() 분석 타입별 AI 프롬프트 생성
    ├── callBedrock() aws4fetch SigV4 서명 → Bedrock API 호출
    ├── run()        분석 실행 진입점 (anomaly/curve/trend/compare)
    └── renderAnomalyTable() 이상치 탐지 결과 테이블 렌더링
```

## AI 분석 유형

| 버튼 | 타입 | 설명 |
|------|------|------|
| ⚠️ 이상치 자동 탐지 | `anomaly` | 전체 측정에서 이상 패턴 탐지 |
| 📈 커브 분석 | `curve` | 선택한 측정의 Fast1~Final 커브 비교 |
| 📊 추이 분석 | `trend` | 시계열 ratio2 & ctrl_area 추이 |
| 🔄 기기 간 비교 | `compare` | 여러 기기(시트)의 데이터 교차 비교 |

## 지원 AI 모델

- `us.anthropic.claude-sonnet-4-5` — Claude Sonnet 4.5 (기본)
- `us.amazon.nova-pro-v1:0` — Amazon Nova Pro
- `us.amazon.nova-lite-v1:0` — Amazon Nova Lite (빠름)

Nova 모델은 Claude와 요청/응답 스키마가 다르므로 `callBedrock()` 내부에서 `isNova` 플래그로 분기 처리한다.
