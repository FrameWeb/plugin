---
name: frameweb-script
description: "Epic Framework Form Script 작성 전문 스킬. Web Worker 샌드박스에서 실행되는 폼 스크립트(registerFunction) 작성, 이벤트 핸들러 구현, sandboxAPI(form/workset/grid/util/popup/print/macro) 활용. 트리거: form_script, 폼 스크립트, registerFunction, registerNativeFunction, sandboxAPI, beforeSave, afterSave, afterOpen, afterNew, beforeDelete, onFormLoad, grid.getData, form.getValue, form.setValue, workset.executeGet, workset.save, 스크립트 작성, 이벤트 핸들러, onChange, onClick, onKeyDown. 폼 스크립트 관련 코드를 작성하거나 디버깅할 때 반드시 이 스킬을 참조할 것."
user-invocable: true
---

# Form Script 스킬

Epic Framework Form Script는 Web Worker 샌드박스에서 실행되는 사용자 스크립트 시스템.
`registerFunction`으로 함수 등록 -> 이벤트 발생 시 자동 호출.

## 0. 스킬 위치 — 폼 디자이너 탭 스킬 6종 중 하나

이 스킬은 폼 디자이너 Script 탭(폼 이벤트 핸들러 form_script) 전용이다. 탭 작업 스킬 6종 — `frameweb-canvas` / `frameweb-bindchain` / `frameweb-binding` / `frameweb-script` / `frameweb-sandbox` / `frameweb-state` — 중 하나이며, 탭과 1:1로 대응한다. 6종은 포지(Forge, Cloud Dev AI 영역) 이식 대상으로 묶여 있다.

`frameweb-sandbox`(폼 전용 AI 도우미 작동 규칙)와 혼동하지 말 것. 이 스킬은 컴포넌트 이벤트에 붙는 실행 스크립트(form_script)를 다룬다.

---

## 1. 함수 등록 패턴

```javascript
// Worker 함수 (기본 - 샌드박스 안전)
registerFunction('함수명', async function(/* args */) {
  try {
    // form, workset, grid, util, popup API 사용 가능
  } catch (error) {
    console.error('[폼ID] 함수명 error:', error.message);
  }
});

// Native 함수 (메인 스레드 - DOM 접근 필요 시)
registerNativeFunction('함수명', async function() {
  // DOM 직접 접근 가능, setTimeout 가능
  // 보안/안정성은 개발자 책임
});

// 유틸리티 (registerFunction 밖 - API 사용 불가)
function helper(args) { /* 순수 로직 */ }
```

**실행 우선순위**: 같은 이름이면 Native 우선 -> Worker fallback

---

## 2. 이벤트 네이밍 규칙

### 컴포넌트 이벤트: `{fcmpId}_{eventName}`

```javascript
// 버튼 클릭: btn_save -> btn_save_onClick
registerFunction('btn_save_onClick', async function() { ... });

// 입력 변경: emp_nm -> emp_nm_onChange
registerFunction('emp_nm_onChange', async function(value) { ... });

// 트리뷰: menutree -> menutree_onChange
registerFunction('menutree_onChange', async function(value) {
  // value = 선택된 노드 cd (string)
});
```

- 런타임이 `{fcmpId}_{eventName}` 함수를 자동 검색/호출
- 미등록이면 정상 스킵 (hasScriptFunction 체크)

### 생명주기 이벤트 (예약 함수명)

| 함수명 | 시점 | args | 설명 |
|--------|------|------|------|
| `afterOpen` | 폼 Open 완료 후 | 없음 | 초기 데이터 로딩, UI 설정 |
| `beforeSave` | 저장 직전 | 없음 | 유효성 검증, 데이터 전처리 |
| `afterSave` | 저장 완료 후 | 없음 | 선택행 복원, 후처리 |
| `afterNew` | New 버튼 후 | 없음 | 초기값 설정 |
| `beforeDelete` | 삭제 직전 | 없음 | 삭제 확인, 조건 검증 |
| `afterDelete` | 삭제 완료 후 | 없음 | 후처리 |
| `onFormLoad` | 스크립트 초기화 후 | 없음 | 최초 1회 실행 |

---

## 3. sandboxAPI 레퍼런스

### form - 컴포넌트 값/상태

```javascript
await form.getValue(fcmpId)           // -> string|string[] (컴포넌트 값)
await form.getLabel(fcmpId)           // -> string (Select: 옵션 label, Input: value)
await form.setValue(fcmpId, value)     // 값 설정 (UI + formData 동시 반영)
await form.setEnabled(fcmpId, bool)   // 활성/비활성
await form.setVisible(fcmpId, bool)   // 표시/숨김
await form.getData()                  // -> object (전체 폼 데이터)
await form.validate()                 // -> boolean (유효성 검증)
```

> **없는 API 주의**: `form.setLabel(fcmpId, text)` **없음**. 라벨(컴포넌트 옆 텍스트) 동적 변경 불가.
> 글자수 표시 같은 동적 라벨이 필요하면 별도 표시용 컴포넌트(BNK_LABEL 등) 추가 후 `setValue`로 텍스트 갱신하거나, 그리드 셀 안에 계산값을 넣는 방식으로 우회.

### workset - 데이터 CRUD

```javascript
await workset.executeGet(wrkId)       // 데이터 조회 (WRKGET 파라미터 기반)
await workset.save(wrkIds?)           // 저장 (save_sq 순, 생략시 전체)
await workset.delete(wrkIds?)         // 삭제 (역순)
await workset.refreshGrid(wrkId)      // GridSet 새로고침
await workset.open()                  // 폼 Open 액션 실행
await workset.executeDataSet(wrkId)   // DataSet 조회
await workset.setValue(wrkId, field, value) // WorkSet 필드값 직접 설정
await workset.markAsNew(wrkId)            // FieldSet을 NEW로 마킹
await workset.generateNumber(tableName, columnName, prefix?, digitCount?)
                                      // -> string (자동채번)
```

### grid - Grid 데이터 조작

```javascript
await grid.getData(wrkId)             // -> Record<string,unknown>[] (전체 행)
await grid.getSelectedRow(wrkId)      // -> Record<string,unknown> | null
await grid.selectRow(wrkId, rowIndex) // 행 인덱스로 선택
await grid.selectRow(wrkId, colName, value) // 컬럼값으로 행 검색 선택
await grid.findRow(wrkId, colName, findWord) // -> number (행 인덱스, -1=없음)
await grid.addRow(wrkId, rowData?)    // 행 추가
await grid.deleteRow(wrkId, rowIndex) // 행 삭제
await grid.setColumns(wrkId, columns) // 그리드 컬럼 정의 동적 변경
await grid.loadData(wrkId, rows)      // 그리드에 행 일괄 주입 (기존 행 대체)
```

#### grid.setColumns / loadData (동적 컬럼 재구성)

CSV 업로드처럼 사용자 데이터 형태에 따라 그리드 컬럼이 달라져야 하는 경우 사용.

```javascript
const columns = [
  { data: 'phone',     title: 'Phone',     width: 120 },
  { data: 'recipient', title: 'Recipient', width: 120 },
  { data: 'language',  title: 'Lang',      width: 60 },
  // ...동적으로 만든 변수 컬럼들
  ...vars.map(v => ({ data: v, title: v, width: 120 })),
  { data: 'message_completed', title: 'Message', width: 400, readOnly: true }
];
await grid.setColumns('ws_recipients', columns);
await grid.loadData('ws_recipients', rows); // rows = [{phone:'010...', recipient:'홍길동', ...}, ...]
```

순서: 항상 `setColumns` 먼저, `loadData` 뒤. 반대로 하면 새 컬럼에 데이터 안 들어감.

### util - 유틸리티

```javascript
await util.alert(message)             // 알림 대화상자 (Worker→메인 스레드 경유)
await util.confirm(message)           // -> boolean (확인/취소)
util.log(...args)                     // 콘솔 로그
util.toast(message, level?)           // 토스트 메시지 (level: 'success'|'error'|'info'|'warning')
await util.fetchApi(url, options?)    // API 호출 (Worker에서 fetch 불가 → 메인 스레드 경유)
```

#### util.fetchApi 사용법

Worker sandbox에서 `fetch()`는 직접 사용 불가. 반드시 `util.fetchApi`를 사용한다.
인증 헤더는 자동 포함되지 않으므로 `credentials: 'include'`로 쿠키 인증에 의존한다.

```javascript
// GET
const data = await util.fetchApi("/api/some/endpoint");

// POST (body는 객체로 전달 — 자동 JSON.stringify)
const result = await util.fetchApi("/api/binding/DSP100/dsp_chain_start/execute/sql", {
  method: "POST",
  body: { crudm: "C", params: { issue_id: 3 } }
});

// 커스텀 헤더
const result = await util.fetchApi("/api/some/endpoint", {
  method: "POST",
  headers: { "x-bis-id": "PACKAGE" },
  body: { key: "value" }
});
```

### Worker sandbox 제약사항

| 항목 | 가능/불가 | 대안 |
|------|----------|------|
| `window` | ❌ 불가 | 없음 (Worker는 DOM 접근 차단) |
| `localStorage` | ❌ 불가 | 없음 |
| `fetch()` | ❌ 불가 | `util.fetchApi()` 사용 |
| `alert()` | ❌ 불가 | `util.alert()` 사용 |
| `confirm()` | ❌ 불가 | `util.confirm()` 사용 |
| `console.log()` | ✅ 가능 | `[FormScript]` 접두사로 F12 콘솔 출력 |
| `form/grid/workset/util` | ✅ 가능 | 스코프에 직접 노출 (api.util 아님) |
| `registerNativeFunction` | ⚠️ V2PreviewPage만 | V5PreviewPage/FormPage에서는 no-op |

> **★ registerFunction 콜백의 인자는 `(api)` 가 아니다.**
> `fn(...args)` 형태로 호출됨. `form/grid/util` 등은 스코프에 이미 노출되어 있으므로 직접 사용.
> ```javascript
> // ✅ 올바른 패턴
> registerFunction("play_onClick", async (selectedData) => {
>   await util.alert("선택: " + selectedData.id);
> });
>
> // ❌ 잘못된 패턴 (api 객체는 전달되지 않음)
> registerFunction("play_onClick", async (api) => {
>   api.util.alert("...");  // TypeError: api.util is undefined
> });
> ```

### popup - 팝업

```javascript
await popup.open(frmId, params, title) // 팝업 열기
await popup.close(result?, data?)      // 팝업 닫기 (결과 반환)
```

### print - 인쇄

```javascript
await print.generate(templateName, inputs) // PDF 생성
await print.openDesigner()                  // 템플릿 디자이너 열기
```

### macro - 매크로

```javascript
await macro.value(key)                // 매크로 값 조회
await macro.resolve(text)             // 텍스트 내 매크로 해석
```

---

## 4. 컴포넌트별 이벤트/인자

> **최신 데이터는 CMPMST.ui_events 조회**:
> ```sql
> SELECT cmp_id, ui_events, component_apis FROM cmpmst WHERE cmp_id = '{cmp_id}';
> ```

### 입력 컴포넌트

| cmp_id | event | args | trigger |
|--------|-------|------|---------|
| EMAX_TEXT_INPUT | onChange | `value: string` | 타이핑 시 |
| EMAX_SELECT | onChange | `value: string` (option value) | 선택 변경 시 |
| EMAX_CHECKBOX | onChange | `value: string` (1/0) | 체크 토글 시 |
| EMAX_DATEPICKER | onChange | `value: string` (YYYY-MM-DD) | 날짜 선택 시 |
| EPIC_TEXT_INPUT | onChange | `value: string` | 타이핑 시 |
| EPIC_SELECT | onChange | `value: string` | 선택 변경 시 |
| EPIC_CHECKBOX | onChange | `value: string` (1/0) | 체크 토글 시 |
| EPIC_DATEPICKER | onChange | `value: string` (YYYY-MM-DD) | 날짜 선택 시 |

### 트리뷰

| cmp_id | event | args | trigger |
|--------|-------|------|---------|
| EMAX_TREEVIEW | onChange | `value: string` (노드 cd) | 노드 클릭 시 |

### 그리드

| cmp_id | event | args | trigger |
|--------|-------|------|---------|
| EMAX_HANDSONTABLE | onChange | `value: Record \| Record[]` | 선택/편집/추가/삭제 시 |

### 버튼

| cmp_id | event | args | trigger |
|--------|-------|------|---------|
| EMAX_BUTTON | onClick | (없음) | 버튼 클릭 시 |
| CMP_BUTTON | onClick | (없음) | 버튼 클릭 시 |
| EPIC_BUTTON | onClick | (없음) | 버튼 클릭 시 |

### BNK_* (BNK 조직 테마용 컴포넌트 시리즈)

| cmp_id | event | args | trigger |
|--------|-------|------|---------|
| BNK_INPUT | onChange | `value: string` | 타이핑 시 |
| BNK_TEXTAREA | onChange | `value: string` | 타이핑 시 |
| BNK_SELECT | onChange | `value: string` (option value) | 선택 변경 시 |
| BNK_HANDSONTABLE | onChange | `value: Record \| Record[]` | 선택/편집/추가/삭제 시 |
| BNK_BUTTON | onClick | (없음) | 버튼 클릭 시 |

#### BNK_INPUT 우측 search 아이콘 → BISPOP 자동 호출

`properties.buttonType: "search"` + `properties.buttonPosition: "right"` 가 설정된 BNK_INPUT은 입력칸 우측에 돋보기 아이콘이 자동 표시되고, **클릭하면 properties.optionSource(=BISPOP pop_id)에 등록된 lookup 팝업이 자동 호출됨**.

흐름:
```
사용자 — 돋보기 클릭
  → handlePopupOpen(popId=optionSource, fcmpId)
    → GET /api/popup/{popId} (BISPOP 메타 조회)
    → LookupPopupModal 그리드 표시
    → 행 선택 + [선택] 클릭
    → popio OUT 매핑(또는 cmpio)에 따라 form field에 자동 setValue
```

**중요**: form_script에서 `popup.open()` 명시 호출 **불필요**. 메커니즘 자체가 컴포넌트에 포함되어 있음.

연관 DB 자원 (BISPOP/popsql/popcmp/popio) 신설/수정은 epic-component 스킬 범위.

#### BNK_INPUT을 hidden 필드로 사용 (자동 계산 컬럼 보관용)

`visible: false`로 화면 숨김 + properties에 `value` 초기값 명시 (보통 `""` 또는 빈 배열 직렬화 `"[]"`).

```sql
INSERT INTO frmcmp (bis_id, frm_id, fcmp_id, cmp_id, x, y, width, height, visible, label, properties)
VALUES ('bnk', 'SMS200', 'inp_tpl_variables', 'BNK_INPUT',
        0, 0, 1, 1, false, 'Variables (auto)',
        jsonb_build_object(
          'value', '[]',
          'required', false,
          'disabled', true,
          'showLabel', false,
          'placeholder', ''
        ));
```

**주의** (hidden 필드의 흔한 주의점):
- properties가 NULL이면 BindingSet 검증이 hidden 필드를 빈값으로 보고 차단할 수 있음 → 초기값 명시 의무
- `disabled: true` 권장 (디자이너 화면에서 실수로 편집 차단)
- `frmbnd.bound_fcmp_ids`에 hidden 필드 ID 포함해야 bndsave가 자동 저장
- bndquery C/U SQL에 컬럼 추가 + JSONB 타입이면 `CAST(:field AS jsonb)`

---

## 5. 실전 패턴

### afterSave 선택행 복원

```javascript
registerFunction('afterSave', async function() {
  const selectedRow = await grid.getSelectedRow('G10');
  const key = selectedRow?.emp_cd;
  await workset.refreshGrid('G10');
  if (key) {
    await grid.selectRow('G10', 'emp_cd', key);
  }
});
```

### beforeSave 유효성 검증

```javascript
registerFunction('beforeSave', async function() {
  const empNm = await form.getValue('emp_nm');
  if (!empNm || empNm.trim() === '') {
    util.alert('사원명을 입력하세요.');
    return false; // 저장 중단
  }
  // return 없거나 true -> 저장 진행
});
```

### 자동채번 (afterNew)

```javascript
registerFunction('afterNew', async function() {
  const newNo = await workset.generateNumber('emp_mst', 'emp_cd', 'EMP', 6);
  await form.setValue('emp_cd', newNo);
  await form.setEnabled('emp_cd', false);
});
```

### onChange 연쇄 조회

```javascript
registerFunction('dept_cd_onChange', async function(value) {
  if (value) {
    await workset.executeGet('G10');
  }
});
```

### 그리드 행 선택 -> 상세 표시

```javascript
registerFunction('G10_onChange', async function(value) {
  if (value && !Array.isArray(value)) {
    await form.setValue('emp_cd', value.emp_cd);
    await form.setValue('emp_nm', value.emp_nm);
  }
});
```

### confirm 대화상자 (beforeDelete)

```javascript
registerFunction('beforeDelete', async function() {
  const ok = await util.confirm('정말 삭제하시겠습니까?');
  if (!ok) return false;
});
```

### onKeyDown 연쇄 조회 (Enter 키)

```javascript
registerFunction('stk_cd_onKeyDown', async function(e) {
  if (e.key === 'Enter') {
    await workset.executeGet('WRK_PRICE');
  }
});
```

### beforeSave 변수 추출 + hidden 필드 + BindingSet 저장 (자동 계산 컬럼)

본문에서 `{변수}` 같은 동적 토큰을 자동 추출해 별도 컬럼에 저장하고 싶을 때.
사용자가 직접 입력 안 하는 hidden 필드 + `beforeSave` 정규식 + bndsave 묶음.

**준비**:
- `frmcmp`에 hidden 필드 추가 (예: `inp_tpl_variables`, BNK_INPUT visible=false, **properties.value 초기값 `"[]"` 명시 필수** — NULL이면 BindingSet 검증이 빈값으로 차단)
- `frmbnd.bound_fcmp_ids`에 hidden 필드 포함
- bndquery C/U SQL에 컬럼 추가 (JSONB 컬럼이면 `CAST(:field AS jsonb)`)

**스크립트**:
```javascript
const VAR_REGEX = /\{([a-z_][a-z0-9_]*)\}/g;

function extractVars(text) {
  const out = new Set();
  if (!text) return [];
  const re = new RegExp(VAR_REGEX.source, 'g');
  let m;
  while ((m = re.exec(text)) !== null) out.add(m[1]);
  return Array.from(out).sort();
}

registerFunction('beforeSave', async function() {
  const body = (await form.getValue('inp_tpl_body_en')) || '';
  const vars = extractVars(body);
  // hidden 필드에 JSON 문자열로 주입 → BindingSet 저장 SQL의 :tpl_variables 파라미터로 전달
  await form.setValue('inp_tpl_variables', JSON.stringify(vars));
  return true;
});
```

### registerNativeFunction CSV 파일 다운로드/업로드

Web Worker는 DOM 접근 불가 → 파일 다이얼로그/다운로드는 `registerNativeFunction`(메인 스레드 트랙) 사용. 그리드 툴바의 사용자 정의 다운로드/업로드 아이콘 onClick에 연결.

```javascript
registerNativeFunction('onDownloadSample', async () => {
  const vars = await getCurrentVariables(); // form.getValue('inp_tpl_variables') 파싱
  if (vars.length === 0) {
    alert('템플릿을 먼저 선택하세요');
    return;
  }
  const header = ['phone', 'recipient', 'language', ...vars].join(',');
  const csv = '﻿' + header + '\n'; // UTF-8 BOM (Excel 호환)
  const blob = new Blob([csv], { type: 'text/csv;charset=utf-8' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'sample.csv';
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
});

registerNativeFunction('onUploadCsv', async () => {
  const vars = await getCurrentVariables();
  const input = document.createElement('input');
  input.type = 'file';
  input.accept = '.csv,text/csv';
  input.onchange = (e) => {
    const file = e.target.files && e.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = async (ev) => {
      const text = String(ev.target.result || '').replace(/^﻿/, '');
      const lines = text.split(/\r?\n/).filter(l => l.trim());
      const header = lines[0].split(',').map(s => s.trim());
      const required = ['phone', 'recipient', 'language', ...vars];
      const missing = required.filter(k => !header.includes(k));
      if (missing.length > 0) {
        alert('CSV 헤더 누락: ' + missing.join(', '));
        return;
      }
      const rows = lines.slice(1).map(line => {
        const cols = line.split(',').map(s => s.trim());
        const obj = {};
        header.forEach((h, i) => { obj[h] = cols[i] || ''; });
        return obj;
      });
      // 그리드 컬럼 동적 재구성 + 행 주입
      const columns = [
        { data: 'phone',     title: 'Phone',     width: 120 },
        { data: 'recipient', title: 'Recipient', width: 120 },
        { data: 'language',  title: 'Lang',      width: 60 },
        ...vars.map(v => ({ data: v, title: v, width: 120 })),
        { data: 'message_completed', title: 'Message', width: 400, readOnly: true }
      ];
      await grid.setColumns('ws_recipients', columns);
      await grid.loadData('ws_recipients', rows);
      alert('CSV 업로드 완료 — ' + rows.length + '건');
    };
    reader.readAsText(file, 'utf-8');
  };
  input.click();
});
```

**핵심 주의**:
- `registerNativeFunction` 안에서는 `util.alert()` 대신 `window.alert()` 사용 가능 (메인 스레드)
- `registerFunction` 안에서 호출하면 안 됨 — Worker 샌드박스라 DOM 부재
- `grid.setColumns` → `grid.loadData` 순서 지킬 것

---

## 6. 주의사항

### setValue 이중 갱신
`form.setValue()`는 내부적으로 formDataRef + worksetTrackingStore 동시 갱신. 직접 formData를 조작하면 안 됨.

### beforeSave = 전처리만
`return false`만이 저장 중단. 다른 return 값은 저장 진행.

### window.confirm 금지
`window.confirm()` 금지 -> `await util.confirm()` 사용 (Playwright 호환).

### Web Worker 환경
- DOM 접근 불가 (document, window 없음)
- setTimeout/setInterval 불가
- 외부 라이브러리 import 불가
- DOM 필요 시 `registerNativeFunction` 사용

### async/await 필수
form, workset, grid, popup API는 모두 비동기. await 누락 시 Promise 객체 반환.

---

## 7. 스크립트 작성 시 컴포넌트 조회

```sql
SELECT f.fcmp_id, f.cmp_id, f.label,
       c.ui_events, c.component_apis
FROM frmcmp f
JOIN cmpmst c ON f.cmp_id = c.cmp_id
WHERE f.frm_id = '{폼ID}'
ORDER BY f.display_order;
```

---

## 8. 인프라 (새 API 추가 시)

### 3곳 수정 필수

| # | 파일 | 역할 |
|---|------|------|
| 1 | `frontend/src/workers/formScriptWorker.ts` (sandboxAPI) | Worker에서 호출 가능하게 |
| 2 | `frontend/src/services/formScriptBridge.ts` (APIHandlers) | 타입 + 라우팅 |
| 3 | `frontend/src/pages/V5PreviewPage.tsx` + `FormViewPage.tsx` (apiHandlers) | 실제 구현 |

### 호출 경로

```
Script registerFunction
  -> sandboxAPI.form.xxx()
    -> formScriptBridge.handleFormAPI()
      -> V5PreviewPage apiHandlers.form.xxx()
        -> formDataRef / DOM / 외부 API
```
