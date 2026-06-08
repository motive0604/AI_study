# Design: xlsx 파일 업로드 및 시각화

## 변경 범위

`trias_standalone.html` 단일 파일만 수정한다. 신규 파일 생성 없음.

---

## 1. CDN 추가

`<head>` 의 Chart.js `<script>` 태그 아래에 SheetJS를 추가한다.

```html
<script src="https://cdn.sheetjs.com/xlsx-latest/package/dist/xlsx.full.min.js"></script>
```

---

## 2. CSS 추가

기존 인라인 `<style>` 끝에 업로드 관련 스타일을 추가한다.

```css
/* 업로드 영역 */
.upload-zone{border:2px dashed #c5d5ff;border-radius:8px;padding:14px 10px;text-align:center;cursor:pointer;transition:.15s;background:#f8f9ff}
.upload-zone:hover{border-color:#6366f1;background:#f0f1ff}
.upload-zone input[type=file]{display:none}
.upload-zone .uz-icon{font-size:22px;margin-bottom:4px}
.upload-zone .uz-label{font-size:11px;color:#6366f1;font-weight:700}
.upload-zone .uz-sub{font-size:10px;color:#999;margin-top:2px}
.upload-msg{font-size:11px;margin-top:6px;padding:5px 8px;border-radius:5px;display:none}
.upload-msg.ok{background:#dcfce7;color:#166534;display:block}
.upload-msg.err{background:#fee2e2;color:#991b1b;display:block}
.upload-msg.loading{background:#f0f4ff;color:#3b5bdb;display:block}
.btn-reset{background:#f1f5f9;color:#64748b;border:1px solid #e2e8f0;width:100%;padding:6px;border-radius:6px;font-size:11px;cursor:pointer;font-family:inherit}
.btn-reset:hover{background:#e2e8f0}
```

---

## 3. HTML 구조 추가

사이드바의 기존 첫 번째 sb-card("📋 데이터 선택") **위에** 업로드 카드를 삽입한다.

```html
<div class="sb-card">
  <div class="sb-title">📂 데이터 업로드</div>
  <div class="sb-body">
    <label class="upload-zone" id="upload-zone">
      <input type="file" id="file-input" accept=".xlsx,.xls">
      <div class="uz-icon">📊</div>
      <div class="uz-label">xlsx 파일 클릭 또는 드롭</div>
      <div class="uz-sub">TRIAS-3 측정 데이터 (.xlsx)</div>
    </label>
    <div class="upload-msg" id="upload-msg"></div>
    <button class="btn-reset" id="btn-reset" style="display:none" onclick="resetData()">↩ 초기 데이터로 되돌리기</button>
  </div>
</div>
```

---

## 4. JavaScript 설계

### 4-1. 전역 변수 추가

```js
const INIT_DATA = JSON.parse(JSON.stringify(DATA)) // 하드코딩 DATA 백업 (깊은 복사)
```

`init()` 첫 줄에 추가. `DATA`가 선언된 직후 바로 백업한다.

### 4-2. 이벤트 바인딩 — `init()` 내부에 추가

```js
// 파일 선택
document.getElementById("file-input").addEventListener("change", e => {
  if(e.target.files[0]) parseXlsx(e.target.files[0])
})

// 드래그&드롭
const zone = document.getElementById("upload-zone")
zone.addEventListener("dragover",  e => { e.preventDefault(); zone.style.borderColor="#6366f1" })
zone.addEventListener("dragleave", ()=> { zone.style.borderColor="" })
zone.addEventListener("drop", e => {
  e.preventDefault(); zone.style.borderColor=""
  if(e.dataTransfer.files[0]) parseXlsx(e.dataTransfer.files[0])
})
```

### 4-3. `parseXlsx(file)` 함수

```
parseXlsx(file)
  ├── 1. setUploadMsg("loading", "파싱 중...")
  ├── 2. FileReader.readAsArrayBuffer(file)
  ├── 3. XLSX.read(buffer, {type:"array"})
  ├── 4. sheetNames.forEach → sheetToData(wb, sheetName)
  ├── 5. 결과가 비어있으면 setUploadMsg("err", ...)
  ├── 6. DATA = 파싱 결과
  ├── 7. rebuildSheetSelect()
  ├── 8. selIdx = 0 ; refresh()
  └── 9. setUploadMsg("ok", `${n}개 시트 로드 완료`)
```

### 4-4. `sheetToData(wb, sheetName)` 함수

```
sheetToData(wb, sheetName)
  ├── XLSX.utils.sheet_to_json(sheet, {defval:null}) → rows[]
  ├── 헤더 정규화: 모든 키를 소문자로 변환
  ├── 필수 컬럼 검사 (date, ratio2, ctrl_area) → 없으면 null 반환
  ├── rows.forEach → buildBlockData(row)
  │     ├── col: row 인덱스
  │     └── blocks: Fast1~Final 블록 분리
  │           → 블록 구분 컬럼이 있으면 사용, 없으면 모든 row를 Final로 처리
  └── [{col, blocks}, ...] 반환
```

**블록 구분 전략**
- xlsx에 `block` 또는 `stage` 컬럼이 있으면 그 값으로 분류
- 없으면 동일 날짜(`date`) 내 순서(1st→Fast1, 2nd→Fast2 ... last→Final)로 자동 분류
- 블록이 1개뿐이면 `Final`로 취급

**curve50 파싱**
- `p0`~`p49` (대소문자 무관) 컬럼이 있으면 배열로 조합
- `curve_0`~`curve_49` 형식도 지원
- 없으면 `curve50: []` 로 처리 (커브 차트는 빈 상태로 표시)

### 4-5. `rebuildSheetSelect()` 함수

```js
function rebuildSheetSelect(){
  const sel = document.getElementById("sel-sheet")
  sel.innerHTML = ""
  Object.keys(DATA).forEach(s => {
    const o = document.createElement("option")
    o.value = s; o.textContent = s
    sel.appendChild(o)
  })
}
```

### 4-6. `resetData()` 함수

```js
function resetData(){
  Object.keys(DATA).forEach(k => delete DATA[k])
  Object.assign(DATA, JSON.parse(JSON.stringify(INIT_DATA)))
  rebuildSheetSelect()
  selIdx = 0; refresh()
  setUploadMsg("", "")
  document.getElementById("btn-reset").style.display = "none"
  document.getElementById("file-input").value = ""
}
```

### 4-7. `setUploadMsg(type, text)` 헬퍼

```js
function setUploadMsg(type, text){
  const el = document.getElementById("upload-msg")
  el.className = "upload-msg" + (type ? " "+type : "")
  el.textContent = text
  document.getElementById("btn-reset").style.display =
    (type === "ok") ? "block" : "none"
}
```

---

## 5. DATA 변경 방식

`DATA`를 `const`에서 `let`으로 변경하지 않고, 객체 내용을 교체하는 방식을 사용한다.

```js
// ✅ 기존 참조를 유지하면서 내용만 교체
Object.keys(DATA).forEach(k => delete DATA[k])
Object.assign(DATA, parsed)
```

`buildPrompt()` 등 `DATA`를 참조하는 함수들이 별도 수정 없이 그대로 동작한다.

---

## 6. 처리 흐름 다이어그램

```
[파일 선택 / 드롭]
      │
      ▼
parseXlsx(file)
      │
      ├─ FileReader.readAsArrayBuffer
      │         │
      │         ▼
      │   XLSX.read() → workbook
      │         │
      │         ▼
      │   sheetNames.forEach
      │         │
      │         ▼
      │   sheetToData() → [{col, blocks}]
      │         │
      │   (필수 컬럼 없으면 skip)
      │
      ├─ DATA 교체 (Object.keys delete + assign)
      │
      ├─ rebuildSheetSelect()
      │
      └─ selIdx=0 → refresh()
              │
              ├─ renderList()
              ├─ renderCurve()
              └─ renderTrend()
```

---

## 7. 에러 처리

| 상황 | 처리 |
|------|------|
| xlsx가 아닌 파일 선택 | `accept` 속성으로 1차 방어, 파싱 실패 시 err 메시지 |
| 필수 컬럼 없는 시트 | 해당 시트 skip, 나머지 처리 후 경고 메시지 포함 |
| 모든 시트 파싱 실패 | `DATA` 교체 안 함, err 메시지 표시 |
| curve50 컬럼 없음 | `curve50: []` 처리, 커브 차트 빈 상태 |
| 파일 읽기 실패 | FileReader.onerror 핸들링, err 메시지 |
