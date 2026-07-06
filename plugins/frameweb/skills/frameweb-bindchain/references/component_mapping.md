# ROCA → Epic 컴포넌트 매핑 사전

> cmpmst 63개 컴포넌트 분석, 팔레트 노출 + 활성 32개 기준
> bnd-design이 FRMCMP INSERT 시 참고

---

## 1. ROCA → Epic 매핑 테이블

| ROCA 컴포넌트 | Epic cmp_id | 비고 |
|---|---|---|
| Frame9.eText | **EPIC_TEXT_INPUT** | inputType으로 text/number/email 분기 |
| Frame9.eCombo | **EPIC_SELECT** | optionSource에 POPCOD 코드 또는 직접 목록 |
| Frame9.eDate | **EPIC_DATEPICKER** | dateType: date/month |
| Frame9.eGrid | **EPIC_HANDSONTABLE** | showToolbar/showFooter |
| Frame9.eMemo | **EPIC_TEXT_INPUT** | inputType: "text", 높이 200+ |
| Frame9.ePanel | **CMP_PANEL** | is_container=true |
| Frame9.eCheck | **EPIC_CHECKBOX** | inlineLabel로 텍스트 |
| Frame9.eRadio | **EPIC_MULTICHECK** | mode: "radio" |
| Frame9.eNumber | **EPIC_TEXT_INPUT** | inputType: "number" |
| Frame9.eButton | **EPIC_BUTTON** | variant: default/secondary/outline/destructive |
| Frame9.ePicture | **EPIC_IMAGE** | editable, entityType/entityField |
| Frame9.eLabel | **EPIC_TEXT_INPUT** | readonly: true (라벨 대용) |
| XtraTabControl | **CMP_TABS** | 자식으로 CMP_TAB_PAGE |
| XtraTabPage | **CMP_TAB_PAGE** | parent_id가 CMP_TABS |
| SimpleButton | **EPIC_BUTTON** | 동일 |
| SplitContainer | **CMP_SPLIT_CONTAINER** | orientation: horizontal/vertical |

### Epic 전용 (ROCA에 없음)

| cmp_id | 용도 |
|---|---|
| EPIC_MULTISELECT | 다중선택 드롭다운 |
| EPIC_FTP_GRID | 파일 첨부 그리드 |
| EPIC_TREEVIEW | 트리뷰 |
| EPIC_MONACO_EDITOR | SQL/코드 에디터 |
| EPIC_CAROUSEL | 캐러셀 슬라이드 |
| CMP_CHART_LINEBAR | 차트 |
| COWORK_CHAT | AI 코워크 채팅 |
| BARCODE_SCANNER | 바코드 스캐너 |

---

## 2. FRMCMP INSERT 예시 (properties 패턴)

### 입력 컴포넌트

```sql
-- 텍스트 입력
('EPIC_TEXT_INPUT', '{"placeholder": "이름을 입력하세요"}'::jsonb)

-- 숫자 입력
('EPIC_TEXT_INPUT', '{"inputType": "number", "placeholder": "0"}'::jsonb)

-- 읽기 전용 (라벨 대용)
('EPIC_TEXT_INPUT', '{"readOnly": true}'::jsonb)

-- 셀렉트 (POPCOD 코드)
('EPIC_SELECT', '{"optionSource": "MSSTAT", "emptyOptionText": "-- 선택 --"}'::jsonb)

-- 셀렉트 (직접 목록)
('EPIC_SELECT', '{"optionSource": "전체, DEV, PLN, DES, HR, QA"}'::jsonb)

-- 체크박스
('EPIC_CHECKBOX', '{"inlineLabel": "재직중", "defaultValue": true}'::jsonb)

-- 날짜
('EPIC_DATEPICKER', '{"placeholder": "YYYY-MM-DD"}'::jsonb)

-- 멀티셀렉트
('EPIC_MULTISELECT', '{"optionSource": "빨강, 파랑, 초록"}'::jsonb)

-- 라디오/멀티체크
('EPIC_MULTICHECK', '{"mode": "radio", "optionSource": "개발, 디자인, 기획"}'::jsonb)

-- 버튼
('EPIC_BUTTON', '{"variant": "default", "text": "실행"}'::jsonb)
-- variant: "default"(파랑), "secondary"(회색), "outline"(테두리), "destructive"(빨강)
```

### 그리드

```sql
-- 기본 CRUD 그리드
('EPIC_HANDSONTABLE', 0,0, 1200,500, 'fill',
 '{"showFooter": true, "showToolbar": {"add": true, "save": true, "delete": true, "search": true, "refresh": true}}'::jsonb)

-- 읽기 전용 그리드
('EPIC_HANDSONTABLE', 0,0, 300,600, 'fill',
 '{"showFooter": true, "showToolbar": {"search": true, "refresh": true}}'::jsonb)
```

### 특수 컴포넌트

```sql
-- 이미지
('EPIC_IMAGE', '{"editable": true, "objectFit": "contain", "entityType": "hra110.photo", "entityField": "emp_cd"}'::jsonb)

-- 파일 첨부 그리드
('EPIC_FTP_GRID', '{"entityType": "DEMO", "maxFileSize": 10}'::jsonb)

-- SQL 에디터
('EPIC_MONACO_EDITOR', '{"theme": "vs-dark", "language": "sql", "readOnly": false}'::jsonb)
```

---

## 3. 레이아웃 컨테이너 패턴

### SplitContainer (좌우/상하 분할)

```
CMP_SPLIT_CONTAINER (parent_id=NULL, layout_mode='fill')
  ├── CMP_SPLITPANEL (panel_index=0)
  │     └── [자식]
  └── CMP_SPLITPANEL (panel_index=1)
        └── [자식]
```

```sql
-- SplitContainer
('split_root','CMP_SPLIT_CONTAINER', 0,0, 1200,600, 'fill', NULL, NULL,
 '{"orientation":"horizontal", "split_size":300, "split_mode":"pixel", "resizable":true, "divider_size":3}'::jsonb)

-- SplitPanel 0 (왼쪽/위)
('panel_0','CMP_SPLITPANEL', 0,0, 300,600, 'fill', '목록', 'split_root',
 '{"panel_index":0, "showLabel":true, "showBorder":true}'::jsonb)

-- SplitPanel 1 (오른쪽/아래)
('panel_1','CMP_SPLITPANEL', 0,0, 900,600, 'fill', '상세', 'split_root',
 '{"panel_index":1, "showLabel":true, "showBorder":true}'::jsonb)
```

**orientation**: `"horizontal"` (좌우) / `"vertical"` (상하)
**split_mode**: `"pixel"` (고정 px) / `"ratio"` (비율 0~1)

### Tabs (탭 전환)

```
CMP_TABS (layout_mode='fill')
  ├── CMP_TAB_PAGE (tab_index=0, title="탭1")
  │     └── [자식]
  └── CMP_TAB_PAGE (tab_index=1, title="탭2")
        └── [자식]
```

```sql
-- Tabs
('tabs_main','CMP_TABS', 0,0, 1200,600, 'fill', NULL, NULL,
 '{"tab_style":"bordered", "tab_height":40}'::jsonb)

-- TabPage
('tab_0','CMP_TAB_PAGE', 0,0, 1200,570, 'fill', '인사정보', 'tabs_main',
 '{"title":"인사정보", "layout_mode":"fill"}'::jsonb)
```

### 중첩 Split (3단 분할)

```
CMP_SPLIT_CONTAINER (horizontal, 350px)
  ├── CMP_SPLITPANEL 0
  │     └── Grid
  └── CMP_SPLITPANEL 1
        └── CMP_SPLIT_CONTAINER (vertical, 450px)  ← 중첩
              ├── CMP_SPLITPANEL 0 → Image
              └── CMP_SPLITPANEL 1 → Fields
```

---

## 4. 주의사항

### CMP_SPLITPANEL 필수 규칙
- hide_in_palette=true (팔레트에서 안 보임)
- SplitContainer 사용 시 **반드시 2개를 수동 INSERT**
- panel_index 0과 1 모두 필수
- parent_id가 CMP_SPLIT_CONTAINER의 fcmp_id를 가리켜야 함

### layout_mode
```
"fill"      부모 꽉 채움 (컨테이너 자식 기본)
"none"      절대좌표 x,y 사용 (Panel 내부 필드 배치)
"stretch-h" 가로만 채움
```

### EMAX vs EPIC 세대
```
EMAX_*  labelAlign/labelWidth 기반 레이블 정렬 내장 (ERP 스타일)
EPIC_*  최신, properties 체계 깔끔 (신규 폼 권장)
```
동일 기능이 양쪽에 존재 (EMAX_TEXT_INPUT / EPIC_TEXT_INPUT). 신규 폼은 **EPIC_*** 사용.

### optionSource 2가지 패턴
```
POPCOD 코드: "MSSTAT"  →  DB에서 popcod 테이블 조회
직접 나열:   "전체, DEV, PLN"  →  쉼표 구분 파싱
```

### showToolbar 혼용
cmpmst default_props에서는 문자열, forms.sql에서는 객체. 런타임이 둘 다 파싱.
