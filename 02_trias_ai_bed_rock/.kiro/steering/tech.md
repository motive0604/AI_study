# TRIAS AI 분석 뷰어 — 기술 명세

## 기술 스택

| 항목 | 내용 |
|------|------|
| 런타임 | 브라우저 (서버 없음, 빌드 도구 없음) |
| 언어 | HTML + Vanilla JS (ES2020+) |
| 차트 | Chart.js 4.4.0 (jsdelivr CDN) |
| 엑셀 파싱 | SheetJS xlsx-latest (cdn.sheetjs.com CDN) |
| AWS 인증 | aws4fetch@1.0.20 (esm.sh CDN, 동적 import) |
| AI | AWS Bedrock — Claude Sonnet 4.5, Amazon Nova Pro/Lite |
| 스타일 | 인라인 CSS (외부 프레임워크 없음) |

## 빌드 / 실행

빌드 도구가 없다. 브라우저에서 `trias_standalone.html` 파일을 직접 열면 실행된다.

```
# 개발 시 로컬 서버 없이 파일 직접 실행
# (CORS 이슈 없음 — 모든 외부 호출은 브라우저에서 직접 수행)
start trias_standalone.html
```

## AWS Bedrock 연동

| 항목 | 값 |
|------|-----|
| 인증 | AWS SigV4 (aws4fetch) |
| IAM 권한 | `bedrock:InvokeModel` |
| 엔드포인트 | `bedrock-runtime.{region}.amazonaws.com/model/{modelId}/invoke` |
| 지원 리전 | us-east-1, us-west-2, ap-northeast-1 |
| 키 저장 | 브라우저 메모리만 (localStorage 미사용) |

### 지원 모델 및 요청 스키마 분기

```js
// Claude: anthropic_version + max_tokens + system
// Nova:   messages + inferenceConfig.maxNewTokens
const isNova = modelId.includes('nova')
```

## 주요 외부 라이브러리

```html
<!-- Chart.js: 번들 UMD, 전역 Chart 객체 -->
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>

<!-- SheetJS: XLSX 전역 객체 -->
<script src="https://cdn.sheetjs.com/xlsx-latest/package/dist/xlsx.full.min.js"></script>

<!-- aws4fetch: callBedrock() 최초 호출 시 동적 로드 -->
const { AwsClient } = await import("https://esm.sh/aws4fetch@1.0.20")
```

## 코딩 컨벤션

- 변수/함수: `camelCase`
- 상수: `UPPER_SNAKE_CASE` (`DATA`, `BCOLORS`, `BLOCKS`, `SYSTEM_PROMPT`)
- CSS 클래스: 짧은 시맨틱명 (`sb-card`, `ai-btn`, `meas-item`), BEM 미사용
- 들여쓰기: 2스페이스
- 세미콜론: 생략 (ASI 의존)
- 함수 선언: `function` 키워드 (호이스팅 활용)
- 비동기: `async/await` + `try/catch`

## 확장 시 주의사항

- **Chart 재생성**: `chCurve?.destroy()` 반드시 호출 후 재생성 (메모리 누수 방지)
- **모델 추가**: `#sel-model` `<option>` 추가 + `callBedrock()` 내 `isNova` 분기 확인
- **CORS**: 브라우저 → Bedrock 직접 호출은 CORS 차단 가능 → FastAPI 프록시 권장
- **단일 파일 원칙**: CSS/JS는 인라인 유지. FastAPI 연동 버전 전환 시 별도 정적 파일로 분리
- **SheetJS 파싱**: `XLSX.read()` → 시트별 `sheet_to_json()` → `DATA` 객체 변환 파이프라인
