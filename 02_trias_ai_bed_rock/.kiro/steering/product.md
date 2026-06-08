# TRIAS AI 분석 뷰어 — 제품 개요

## 제품 설명

TRIAS-3 IVD(체외진단) 장비의 측정 데이터를 시각화하고, AWS Bedrock AI(Claude / Nova)로 자동 분석하는 **단일 파일 웹 앱**이다.

- 외부 서버·빌드 도구 없이 브라우저에서 바로 실행
- TRIAS-3 rawdata xlsx 파일 업로드 → 자동 파싱
- 인플루엔자 A/B(FluA/FluB) 판정 결과 및 광학 신호 커브 시각화
- AWS Bedrock API를 SigV4 인증으로 직접 호출하여 AI 분석 수행

## 핵심 기능

| 기능 | 설명 |
|------|------|
| xlsx 업로드 | SheetJS로 TRIAS-3 rawdata 파싱, DATA 객체 생성 |
| 커브 오버레이 | Fast1~Final 블록의 50포인트 광학 신호 차트 |
| 추이 차트 | ratio2 & ctrl_area 시계열 이중 Y축 차트 |
| AI 이상치 탐지 | 전체 데이터에서 이상 측정 자동 탐지 |
| AI 커브 분석 | 선택 측정의 블록별 커브·판정 비교 분석 |
| AI 추이 분석 | 시계열 ratio2/ctrl_area 패턴 분석 |
| AI 기기 간 비교 | 여러 기기(시트) Final 블록 교차 비교 |

## 도메인 용어

- **블록**: Fast1, Fast2, Fast3, Fast4, Final — 측정 단계
- **ratio2**: T2 면적 / Control 면적 비율 (양성 판정 핵심 지표)
- **ctrl_area**: Control 라인 면적 (유효성 지표)
- **item1 / item2**: FluA / FluB 판정 결과 (POSITIVE / NEGATIVE / INVALID / null)
- **curve50**: 50포인트 광학 신호 배열 (10ms 간격, 총 490ms)
