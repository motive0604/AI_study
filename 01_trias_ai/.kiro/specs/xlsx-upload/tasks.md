# Tasks: xlsx 파일 업로드 및 시각화

## 구현 순서

---

- [x] 1. SheetJS CDN 추가
  - `<head>` 의 Chart.js `<script>` 태그 바로 아래에 SheetJS CDN 태그를 삽입한다
  - `https://cdn.sheetjs.com/xlsx-latest/package/dist/xlsx.full.min.js`
  - **파일**: `trias_standalone.html`

- [x] 2. 업로드 관련 CSS 추가
  - 기존 인라인 `<style>` 블록 끝 (`</style>` 바로 앞)에 스타일을 추가한다
  - 추가 대상: `.upload-zone`, `.upload-msg`, `.btn-reset` 클래스 스타일
  - design.md 섹션 2 참고
  - **파일**: `trias_standalone.html`

- [x] 3. 업로드 UI 카드 추가
  - 사이드바 첫 번째 sb-card("📋 데이터 선택") 바로 위에 "📂 데이터 업로드" sb-card를 삽입한다
  - 포함 요소: `#upload-zone` (label), `#file-input` (hidden input), `#upload-msg`, `#btn-reset`
  - design.md 섹션 3 참고
  - **파일**: `trias_standalone.html`

- [x] 4. INIT_DATA 백업 및 전역 변수 추가
  - `DATA` 상수 선언 직후에 `const INIT_DATA = JSON.parse(JSON.stringify(DATA))` 추가
  - 하드코딩 데이터를 초기화 기준값으로 보존한다
  - **파일**: `trias_standalone.html`

- [x] 5. `setUploadMsg(type, text)` 헬퍼 함수 구현
  - `#upload-msg` 요소의 className과 textContent를 업데이트한다
  - type: `"loading"` | `"ok"` | `"err"` | `""`
  - `"ok"` 일 때만 `#btn-reset` 을 표시한다
  - design.md 섹션 4-7 참고
  - **파일**: `trias_standalone.html`

- [ ] 6. `rebuildSheetSelect()` 함수 구현
  - `#sel-sheet` 드롭다운을 현재 `DATA`의 키 목록으로 재구성한다
  - 기존 option을 모두 제거한 뒤 새로 추가한다
  - design.md 섹션 4-5 참고
  - **파일**: `trias_standalone.html`

- [ ] 7. `sheetToData(wb, sheetName)` 함수 구현
  - `XLSX.utils.sheet_to_json()` 으로 rows 배열 추출
  - 헤더 키를 소문자로 정규화
  - 필수 컬럼(`date`, `ratio2`, `ctrl_area`) 없으면 `null` 반환
  - `block` / `stage` 컬럼 기반 블록 분류, 없으면 동일 date 내 순서로 자동 분류
  - `p0`~`p49` 또는 `curve_0`~`curve_49` 컬럼으로 `curve50` 배열 조합
  - `stats` 객체(`baseline`, `t2_peak`, `t2_pos`, `c_peak`, `c_pos`, `snr_t2`, `snr_c`) 조합
  - 반환: `[{col, blocks:{Fast1,Fast2,Fast3,Fast4,Final}}, ...]`
  - design.md 섹션 4-4 참고
  - **파일**: `trias_standalone.html`

- [ ] 8. `parseXlsx(file)` 함수 구현
  - `setUploadMsg("loading", "파싱 중...")` 호출
  - `FileReader.readAsArrayBuffer` 로 파일 읽기
  - `XLSX.read(buffer, {type:"array"})` 로 workbook 생성
  - `sheetNames.forEach` → `sheetToData()` 호출, null 반환 시트는 skip
  - 결과가 비어있으면 `setUploadMsg("err", ...)` 후 종료
  - `Object.keys(DATA).forEach(k=>delete DATA[k])` + `Object.assign(DATA, result)` 로 DATA 교체
  - `rebuildSheetSelect()` → `selIdx=0` → `refresh()` 호출
  - `setUploadMsg("ok", `${n}개 시트 로드 완료`)` 호출
  - `FileReader.onerror` 핸들링
  - design.md 섹션 4-3 참고
  - **파일**: `trias_standalone.html`

- [ ] 9. `resetData()` 함수 구현
  - `INIT_DATA` 를 이용해 `DATA` 를 원래 하드코딩 상태로 복원
  - `rebuildSheetSelect()` → `selIdx=0` → `refresh()` 호출
  - `#upload-msg` 초기화, `#btn-reset` 숨김, `#file-input` 값 초기화
  - design.md 섹션 4-6 참고
  - **파일**: `trias_standalone.html`

- [ ] 10. `init()` 에 이벤트 바인딩 추가
  - `#file-input` change 이벤트 → `parseXlsx()` 호출
  - `#upload-zone` dragover / dragleave / drop 이벤트 → 드래그&드롭 지원
  - design.md 섹션 4-2 참고
  - **파일**: `trias_standalone.html`

- [ ] 11. `init()` 내 `rebuildSheetSelect()` 로 초기화 통합
  - 기존 `init()` 의 시트 드롭다운 생성 코드를 `rebuildSheetSelect()` 호출로 교체한다
  - 중복 코드 제거
  - **파일**: `trias_standalone.html`

- [ ] 12. 동작 검증
  - 샘플 xlsx 파일 업로드 시 드롭다운에 시트명이 나타나는지 확인
  - 커브 차트와 추이 차트가 업로드 데이터로 갱신되는지 확인
  - AI 분석 버튼이 업로드 데이터를 기반으로 프롬프트를 생성하는지 확인
  - "초기 데이터로 되돌리기" 버튼이 하드코딩 DATA를 복원하는지 확인
  - curve50 컬럼 없는 xlsx 업로드 시 에러 없이 빈 커브 차트가 표시되는지 확인
  - 필수 컬럼 없는 시트가 포함된 xlsx 업로드 시 해당 시트만 skip하는지 확인
