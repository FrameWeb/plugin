---
name: frameweb-drawio
description: >
  FrameWeb 다이어그램 관리자(frameweb.mupai.studio/diagrams) 전용 스킬. 말로 설명이
  어려운 개념(아키텍처·데이터 흐름·ERD·절차)을 draw.io 다이어그램으로 직접 그려
  저장하고, 기존 다이어그램을 읽고 수정한다. 저장소는 diagram_mst, API 는
  /api/diagrams CRUD. Trigger: drawio, draw.io, 다이어그램, 도식, 흐름도, 구조도,
  ERD 그려, 그림으로 설명, 그림 그려, diagram_mst, frm_diagram, DiagramManagerPage,
  /diagrams, 다이어그램 관리자, preview_svg, frameweb-drawio.
user-invocable: true
---

# FrameWeb draw.io — 그림으로 설명하는 배려

말로 설명해도 개념이 잘 전달되지 않을 때, draw.io 다이어그램을 직접 그려
`https://frameweb.mupai.studio/diagrams` 에 저장하고 링크를 안내한다.
사용자가 "그려 달라"고 말하기 전에도, 구조·흐름·관계 설명이 길어지면
먼저 그림을 제안하거나 그려서 보여준다. 그것이 이 스킬의 존재 이유다.

## 언제 그리는가

- 아키텍처·데이터 흐름·호출 순서 등 **관계가 3개 이상 얽힌 설명**을 할 때.
- 테이블 관계(ERD), 상태 전이, 배포 구조처럼 **말보다 도식이 빠른 주제**일 때.
- 사용자가 같은 개념을 두 번 이상 되물을 때 — 글을 더 쓰지 말고 그림으로 바꾼다.
- 사용자가 명시적으로 "그려줘 / 다이어그램으로 / 도식으로" 라고 할 때.

그린 뒤에는 반드시 한 줄로 안내한다:
"다이어그램 관리자(https://frameweb.mupai.studio/diagrams)에 '{제목}'으로 저장했습니다."

## 저장 위치와 화면

- 본체 테이블: 시스템 DB `diagram_mst` (bis_id 파티션, `is_active=TRUE` 만 노출).
- 폼 연결 매핑: `frm_diagram` (본체 하나를 여러 폼에 연결 가능).
- 화면: 개발자(Dev) 화면 `/diagrams` = `DiagramManagerPage`. 목록에서 제목을 클릭하면
  draw.io embed 편집기(`DiagramEditorOverlay`)가 풀스크린으로 열린다.
- 화면 목록은 **현재 선택된 비즈니스(bis_id)의 다이어그램만** 보여준다.
  잘못된 bis_id 로 저장하면 사용자 화면에 나타나지 않는다.

## API — 인증과 CRUD

베이스 URL 은 프로덕션 `https://frameweb.mupai.studio` (로컬 개발이면
`http://localhost:8181`). `/diagrams` 는 개발자 화면이므로 Dev 계정으로 로그인한다.

```bash
BASE=https://frameweb.mupai.studio
TOK=$(curl -s -X POST $BASE/api/auth/login -H 'Content-Type: application/json' \
  -d "{\"username\":\"${FRAMEWEB_LOGIN_USER}\",\"password\":\"${FRAMEWEB_LOGIN_PW}\"}" \
  | python3 -c "import json,sys;print(json.load(sys.stdin)['access_token'])")
```

모든 호출에 `Authorization: Bearer $TOK` + `X-Bis-Id: {bis_id}` 헤더를 붙인다.
bis_id 는 사용자가 지금 작업 중인 비즈니스다. 모르면 먼저 확인한다 — 헤더를
빠뜨리면 `'default'` 로 저장되어 사용자 화면에 보이지 않는다.

| 메서드 | 경로 | 동작 |
|--------|------|------|
| GET | `/api/diagrams` | 목록 `{items:[...]}` — diagram_id, title, diagram_type, updated_at, has_preview, linked_forms |
| POST | `/api/diagrams` | 생성 `{title, diagram_type, drawio_xml?, preview_svg?}` → diagram_id 반환 |
| GET | `/api/diagrams/{id}` | 단건 — `drawio_xml` 포함 (읽기·수정의 시작점) |
| PUT | `/api/diagrams/{id}` | 부분 수정 — payload 에 넣은 키만 갱신 |
| DELETE | `/api/diagrams/{id}` | **본체 완전 삭제** — 폼 연결도 CASCADE 로 함께 삭제 |

폼 경유 경로(`/api/forms/{frm_id}/diagrams`)는 별도다:

- `POST /api/forms/{frm_id}/diagrams/link` `{diagram_id}` — 기존 본체를 폼에 연결.
- `DELETE /api/forms/{frm_id}/diagrams/{id}` — **연결만 해제** (본체는 남는다).
- 구현: `backend/api/diagram_router.py`, `backend/api/form_spec_router.py`.

검증 규칙 (서버가 거부하는 것):
- `diagram_type` ∈ `erd | flow | api | process | etc` (기본 `erd`).
- `drawio_xml` ≤ 5MB.
- `preview_svg` 는 `data:image/svg+xml` 로 시작하는 데이터 URL만 허용 (빈 문자열 허용).
- `title` 은 공백 제거 후 비어 있으면 400.

## drawio XML 직접 작성 규칙

편집기는 XML 을 그대로 주고받으므로 **압축 없는 평문 XML** 을 손으로 작성하면 된다.
(draw.io 파일 export 의 base64 압축 형식을 흉내낼 필요 없다.)

```xml
<mxfile host="frameweb">
  <diagram id="d1" name="Page-1">
    <mxGraphModel dx="800" dy="600" grid="1" gridSize="10" page="1"
        pageWidth="1169" pageHeight="826">
      <root>
        <mxCell id="0"/>
        <mxCell id="1" parent="0"/>
        <mxCell id="n1" value="브라우저" vertex="1" parent="1"
            style="rounded=1;whiteSpace=wrap;html=1;fillColor=#dae8fc;strokeColor=#6c8ebf;">
          <mxGeometry x="40" y="40" width="160" height="60" as="geometry"/>
        </mxCell>
        <mxCell id="n2" value="백엔드 API" vertex="1" parent="1"
            style="rounded=1;whiteSpace=wrap;html=1;fillColor=#d5e8d4;strokeColor=#82b366;">
          <mxGeometry x="300" y="40" width="160" height="60" as="geometry"/>
        </mxCell>
        <mxCell id="e1" value="JWT 요청" edge="1" parent="1" source="n1" target="n2"
            style="edgeStyle=orthogonalEdgeStyle;rounded=1;html=1;">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>
      </root>
    </mxGraphModel>
  </diagram>
</mxfile>
```

- `<mxCell id="0"/>` 과 `<mxCell id="1" parent="0"/>` 두 루트 셀은 **반드시 포함**한다.
  빠지면 편집기가 빈 화면을 보여준다.
- 노드(vertex) 필수: `vertex="1" parent="1"` + `<mxGeometry x y width height as="geometry"/>`.
- 선(edge) 필수: `edge="1" parent="1" source target` + `<mxGeometry relative="1" as="geometry"/>`.
- 한국어 라벨은 `whiteSpace=wrap;html=1` 스타일과 함께 넣어야 줄바꿈이 된다.
- XML 특수문자는 이스케이프한다: `&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`.
- 배치 요령: 흐름은 왼쪽→오른쪽 또는 위→아래 한 방향으로. 노드 간격은 최소 100px,
  선이 노드를 가로지르지 않게 x/y 를 잡는다. 노드 10개 초과면 그룹(컨테이너)보다
  페이지를 나누는 편이 읽기 쉽다.
- 자주 쓰는 채움색: 파랑 `#dae8fc/#6c8ebf`(외부·화면), 초록 `#d5e8d4/#82b366`(서버·정상),
  주황 `#ffe6cc/#d79b00`(주의·비동기), 빨강 `#f8cecc/#b85450`(오류·차단), 회색
  `#f5f5f5/#666666`(비활성). 개념 구분 외의 장식은 넣지 않는다.

## 저장 절차

XML 을 JSON 문자열로 직접 이스케이프하지 말고, 파일로 쓴 뒤 python3 로 감싼다.

1. 스크래치패드에 `diagram.xml` 작성 (UTF-8).
2. 로그인 토큰 발급 (위 인증 절).
3. 생성 호출:

```bash
python3 - "$TOK" "{bis_id}" <<'PY'
import json, sys, urllib.request
tok, bis = sys.argv[1], sys.argv[2]
xml = open('diagram.xml', encoding='utf-8').read()
body = json.dumps({"title": "제목", "diagram_type": "flow", "drawio_xml": xml}).encode()
req = urllib.request.Request("https://frameweb.mupai.studio/api/diagrams", data=body,
    method="POST", headers={"Content-Type": "application/json",
    "Authorization": f"Bearer {tok}", "X-Bis-Id": bis})
print(urllib.request.urlopen(req).read().decode())
PY
```

4. 응답의 `diagram_id` 를 확보하고, 사용자에게 `/diagrams` 링크와 제목을 안내한다.

## 읽기·수정 절차

1. `GET /api/diagrams` 목록에서 제목으로 대상 `diagram_id` 를 찾는다.
2. `GET /api/diagrams/{id}` 로 `drawio_xml` 을 받아 파일로 저장하고 구조를 읽는다.
3. XML 을 고친 뒤 `PUT /api/diagrams/{id}` 로 `drawio_xml` 만 보낸다.
   payload 에 넣은 키만 갱신되므로 title/preview_svg 는 건드리지 않으면 보존된다.
4. 수정 후 `preview_svg` 는 옛 그림으로 남는다. 미리보기 갱신은 사용자가 편집기에서
   한 번 저장하면 자동으로 되므로, 수정 보고에 "썸네일은 편집기에서 저장 시 갱신"을 덧붙인다.

## preview_svg — 목록 썸네일

- 선택 사항이다. 없으면 목록 카드에 썸네일이 비어 있을 뿐 동작엔 지장 없다.
- 넣으려면 `data:image/svg+xml;base64,...` 데이터 URL 로 보낸다 (prefix 검증 있음).
- 손으로 그린 다이어그램이라면 간단한 축약 SVG 를 직접 만들어 base64 인코딩해 동봉해도
  되고, 생략한 뒤 편집기 저장에 맡겨도 된다. 정밀 재현에 시간을 쓰지 않는다.

## 함정

- **DELETE 두 경로의 의미가 다르다.** `/api/diagrams/{id}` 는 본체 완전 삭제(폼 연결까지
  CASCADE), `/api/forms/{frm_id}/diagrams/{id}` 는 그 폼과의 연결만 해제. 사용자가
  "폼에서 빼 달라"고 하면 후자다. 본체 삭제는 복구 불가이므로 실행 전에 확인한다.
- **X-Bis-Id 누락 = 유령 저장.** 헤더 없이 저장하면 `'default'` 파티션에 들어가
  사용자 화면에 안 보인다. 저장 후 `GET /api/diagrams` 로 같은 헤더 조합으로 재조회해
  목록에 있는지 확인한다.
- **PUT 은 부분 갱신.** `"preview_svg": null` 처럼 키를 명시적으로 넣으면 그 값으로
  덮어쓴다. 지울 의도가 없으면 키 자체를 payload 에서 뺀다.
- **XML 을 셸 인라인 JSON 에 붙여넣지 않는다.** 따옴표·한글 이스케이프가 깨지기 쉽다.
  반드시 파일 → python3 `json.dumps` 경로로 보낸다.
- 5MB 초과 XML 은 413 으로 거부된다. 손으로 그리는 설명용 다이어그램에서 넘을 일은
  없지만, 기존 다이어그램에 이어 붙일 때는 크기를 확인한다.

## 검증

1. **저장 확인** — `GET /api/diagrams` (같은 X-Bis-Id) 목록에 제목이 있는지.
2. **본문 확인** — `GET /api/diagrams/{id}` 의 `drawio_xml` 이 보낸 XML 과 같은지.
3. **화면 확인** — 사용자에게 `https://frameweb.mupai.studio/diagrams` 에서 제목 클릭으로
   열리는지 확인을 요청한다 (편집기 렌더까지는 API 로 검증할 수 없다 — 미실행 항목으로 보고).

## 참조

- `backend/api/diagram_router.py` — 본체 CRUD (이 스킬의 주 경로)
- `backend/api/form_spec_router.py` — 폼 경유 경로 + 검증 헬퍼 단일 소스
- `frontend/src/pages/DiagramManagerPage.tsx` — /diagrams 화면
- `frontend/src/formDesigner/ui/DiagramEditorOverlay.tsx` — draw.io embed 편집기
