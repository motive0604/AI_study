# Requirements: xlsx 파일 업로드 및 시각화

## 개요

사용자가 TRIAS-3 기기에서 출력된 xlsx 측정 데이터 파일을 업로드하면, SheetJS로 파싱하여 기존 `DATA` 구조로 변환하고 Chart.js로 시각화한다.
하드코딩된 `DATA` 없이도 실제 측정 파일을 바로 분석할 수 있도록 한다.

---

## 요구사항

### REQ-1: xlsx 파일 업로드 UI
- **REQ-1.1** 사이드바 최상단에 "📂 데이터 업로드" sb-card를 추가한다.
- **REQ-1.2** `<input type="file" accept=".xlsx,.xls">` 버튼을 제공한다.
- **REQ-1.3** 파일 선택 즉시 자동으로 파싱을 시작한다 (별도 확인 버튼 불필요).
- **REQ-1.4** 파싱 중 로딩 상태를 사용자에게 표시한다.
- **REQ-1.5** 파싱 성공/실패 결과를 인라인 메시지로 표시한다.

### REQ-2: SheetJS 파싱
- **REQ-2.1** SheetJS(`xlsx` 라이브러리)를 CDN으로 로드한다. (`https://cdn.sheetjs.com/xlsx-latest/package/dist/xlsx.full.min.js`)
- **REQ-2.2** xlsx 파일의 각 시트를 TRIAS-3 기기 1대로 인식한다 (시트명 = 기기명).
- **REQ-2.3** 시트 내 행/열 구조를 파싱하여 기존 `BlockData` 스키마에 맞게 변환한다.
- **REQ-2.4** 파싱 불가 시트는 건너뛰고 나머지를 처리한다.
- **REQ-2.5** 파싱 결과를 기존 `DATA` 전역 변수에 병합(merge)하거나 교체(replace)할지 선택할 수 있다.

### REQ-3: xlsx 컬럼 매핑 (TRIAS-3 출력 형식 기준)
- **REQ-3.1** 다음 컬럼을 인식하여 매핑한다:
  - `date` / `Date` → `BlockData.date`
  - `ratio2` / `Ratio2` → `BlockData.ratio2`
  - `ctrl_area` / `CtrlArea` → `BlockData.ctrl_area`
  - `t2_area` / `T2Area` → `BlockData.t2_area`
  - `det_FluA` / `DetFluA` → `BlockData.det_FluA`
  - `det_FluB` / `DetFluB` → `BlockData.det_FluB`
  - `item1` / `Item1` → `BlockData.item1`
  - `item2` / `Item2` → `BlockData.item2`
  - `curve50` 컬럼 50개 (`P0`~`P49` 또는 `curve_0`~`curve_49`) → `BlockData.curve50[]`
  - `baseline`, `t2_peak`, `t2_pos`, `c_peak`, `c_pos`, `snr_t2`, `snr_c` → `BlockData.stats`
- **REQ-3.2** 컬럼명은 대소문자를 구분하지 않고 매핑한다.
- **REQ-3.3** 필수 컬럼(`date`, `ratio2`, `ctrl_area`)이 없으면 해당 시트를 건너뛴다.

### REQ-4: 데이터 반영 및 시각화
- **REQ-4.1** 파싱 완료 후 기기(시트) 드롭다운을 자동으로 업데이트한다.
- **REQ-4.2** 업로드한 첫 번째 시트를 자동 선택한다.
- **REQ-4.3** `refresh()` 를 호출하여 커브 차트 및 추이 차트를 즉시 업데이트한다.
- **REQ-4.4** 업로드 후 AI 분석 버튼이 정상 동작해야 한다.

### REQ-5: 하드코딩 DATA와의 공존
- **REQ-5.1** 파일 미업로드 시에는 기존 하드코딩 `DATA`로 동작한다.
- **REQ-5.2** 파일 업로드 시 `DATA`를 파싱 결과로 교체한다 (초기 하드코딩 DATA는 유지하지 않음).
- **REQ-5.3** "초기 데이터로 되돌리기" 버튼을 제공한다.
