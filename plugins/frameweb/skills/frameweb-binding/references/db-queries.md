# bnd* DB 쿼리 모음 — epic-cloud 환경

bnd-setup 스킬의 참조 문서. 모든 쿼리는 epic-cloud 환경(${FRAMEWEB_DB_HOST}:${FRAMEWEB_DB_PORT}) 기준.

---

## 1. DB 접속

### psql 직접 (권장)
```bash
# 시스템 DB
psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME}

# 비즈니스 DB
psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d {bis_id}

# 환경변수 (비밀번호 프롬프트 생략)
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "SQL문;"
```

### Python (긴 쿼리, 프로그래밍 필요 시)
```python
python3 -c "
import psycopg2
conn = psycopg2.connect(host='${FRAMEWEB_DB_HOST}', port=${FRAMEWEB_DB_PORT}, user='${FRAMEWEB_DB_USER}', password='${FRAMEWEB_DB_PASSWORD}', dbname='${FRAMEWEB_DB_NAME}')
cur = conn.cursor()
cur.execute('SQL문')
for row in cur.fetchall(): print(row)
cur.close(); conn.close()
"
```

### Backend API (대안)
```bash
# bnd* 설정 조회
curl -s localhost:8181/api/binding/{frm_id}/configs | python3 -m json.tool

# FRMBND 목록
curl -s localhost:8181/api/binding/{frm_id} | python3 -m json.tool

# BNDQUERY 목록
curl -s localhost:8181/api/bindings/{frm_id}/{bnd_id}/queries | python3 -m json.tool
```

---

## 2. 자주 쓰는 조회 쿼리

### 전체 스냅샷
```sql
SELECT 'bisfrm' AS tbl, count(*) FROM bisfrm WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
UNION ALL SELECT 'frmcmp', count(*) FROM frmcmp WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
UNION ALL SELECT 'frmbnd', count(*) FROM frmbnd WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
UNION ALL SELECT 'bndquery', count(*) FROM bndquery WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
UNION ALL SELECT 'bndpull', count(*) FROM bndpull WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
UNION ALL SELECT 'bndsave', count(*) FROM bndsave WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
UNION ALL SELECT 'bndcmp', count(*) FROM bndcmp WHERE frm_id='{폼ID}' AND bis_id='{bis_id}';
```

### frmbnd 설정 조회
```sql
SELECT bnd_id, bnd_type, bound_fcmp_ids, open_sq, trigger_yn, open_trigger, new_yn, save_sq
FROM frmbnd WHERE frm_id='{폼ID}' AND bis_id='{bis_id}' ORDER BY open_sq;
```

### bndquery SQL 전문 조회
```sql
SELECT bnd_id, crudm, execution_order, source_type, query
FROM bndquery WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
ORDER BY bnd_id, crudm, execution_order;
```

### bndpull 파라미터 조회
```sql
SELECT bnd_id, crudm, param_nm, source_type, source_ref, source_column, default_val
FROM bndpull WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
ORDER BY bnd_id, param_nm;
```

### bndsave SET 매핑 조회
```sql
SELECT bnd_id, param_nm, source_type, source_ref, source_column, default_val
FROM bndsave WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
ORDER BY bnd_id, param_nm;
```

### bndcmp 컬럼 매핑 조회
```sql
SELECT bnd_id, fcmp_id, data_field, data_type, col_seq, label, visible, is_required, editor_type
FROM bndcmp WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
ORDER BY bnd_id, col_seq;
```

### frmcmp에서 그리드 컴포넌트 찾기
```sql
SELECT fcmp_id, cmp_id, label, parent_id
FROM frmcmp WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
AND cmp_id IN ('EPIC_HANDSONTABLE', 'EMAX_HANDSONTABLE');
```

### bispop 입력보조 조회
```sql
SELECT pop_id, pop_nm, pop_type FROM bispop WHERE bis_id='{bis_id}' ORDER BY pop_id;
```

### 비즈니스 테이블 컬럼 확인
```sql
-- 비즈니스 DB에서 실행
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_name = '테이블명'
ORDER BY ordinal_position;
```

---

## 3. 수정 쿼리 (INSERT/UPDATE)

### frmbnd 보정
```sql
UPDATE frmbnd SET
  bound_fcmp_ids = '["grid_xxx"]',
  open_sq = 1,
  trigger_yn = false,
  open_trigger = 'OPEN',
  new_yn = 'N',
  save_sq = 1
WHERE frm_id='{폼ID}' AND bis_id='{bis_id}' AND bnd_id='BND_ID';
```

### bndquery INSERT
```sql
INSERT INTO bndquery
  (bis_id, frm_id, bnd_id, crudm, execution_order, source_type, query, description, created_at)
VALUES
  ('{bis_id}', '{폼ID}', 'BND_ID', 'R', 1, 'SQL',
   'SELECT col1, col2 FROM table WHERE co_cd = :co_cd ORDER BY col1',
   '조회', NOW())
ON CONFLICT (bis_id, frm_id, bnd_id, crudm, execution_order) DO NOTHING;
```

### bndpull INSERT
```sql
INSERT INTO bndpull
  (bis_id, frm_id, bnd_id, crudm, param_nm, source_type, source_ref, source_column, default_val, created_at)
VALUES
  ('{bis_id}', '{폼ID}', 'BND_ID', 'R', 'co_cd', 'SESSION', '<$co_cd>', NULL, NULL, NOW())
ON CONFLICT (bis_id, frm_id, bnd_id, crudm, param_nm) DO NOTHING;
```

### bndsave INSERT
```sql
INSERT INTO bndsave
  (bis_id, frm_id, bnd_id, param_nm, source_type, source_ref, source_column, default_val, created_at)
VALUES
  ('{bis_id}', '{폼ID}', 'BND_ID', 'col1', 'GRIDSET', 'BND_ID', 'col1', NULL, NOW())
ON CONFLICT (bis_id, frm_id, bnd_id, param_nm) DO NOTHING;
```

### bndsave SESSION 매핑 (공통)
```sql
-- reg_id
INSERT INTO bndsave (bis_id, frm_id, bnd_id, param_nm, source_type, source_ref, created_at)
VALUES ('{bis_id}', '{폼ID}', 'BND_ID', 'reg_id', 'SESSION', '<$reg_id>', NOW())
ON CONFLICT (bis_id, frm_id, bnd_id, param_nm) DO NOTHING;

-- co_cd
INSERT INTO bndsave (bis_id, frm_id, bnd_id, param_nm, source_type, source_ref, created_at)
VALUES ('{bis_id}', '{폼ID}', 'BND_ID', 'co_cd', 'SESSION', '<$co_cd>', NOW())
ON CONFLICT (bis_id, frm_id, bnd_id, param_nm) DO NOTHING;
```

### bndsave _old 매핑
```sql
-- emp_no_old → source_column은 원본 필드명 emp_no
INSERT INTO bndsave (bis_id, frm_id, bnd_id, param_nm, source_type, source_ref, source_column, created_at)
VALUES ('{bis_id}', '{폼ID}', 'BND_ID', 'emp_no_old', 'GRIDSET', 'BND_ID', 'emp_no', NOW())
ON CONFLICT (bis_id, frm_id, bnd_id, param_nm) DO NOTHING;
```

### 하위 GridSet FK 자체참조 수정
```sql
-- 자체 참조 확인
SELECT bnd_id, param_nm, source_ref FROM bndsave
WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
AND source_type = 'GRIDSET' AND source_ref = bnd_id;

-- 마스터 참조로 수정
UPDATE bndsave SET source_ref = '마스터_bnd_id'
WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
AND bnd_id = '하위_bnd_id' AND param_nm = 'fk_field'
AND source_type = 'GRIDSET' AND source_ref = bnd_id;
```

---

## 4. SQL 작성 규칙

### PostgreSQL 표준 패턴

| 규칙 | 설명 |
|------|------|
| 바인드 변수 | `:column_name` 형식 |
| 선택적 조건 | `{{AND col = :param}}` (null이면 자동 스킵) |
| 타입 캐스팅 | `:param::timestamp`, `:param::integer` 명시적 |
| ORDER BY | R 쿼리에 반드시 포함 |
| NOW() | 타임스탬프 컬럼 (cdt, mdt, created_at) |
| ON CONFLICT | UPSERT: `INSERT ... ON CONFLICT (pk) DO UPDATE SET ...` |

### _old 파라미터 규칙 (UPDATE vs DELETE)
```sql
-- UPDATE: _old 유지 (SET에 새 값 + WHERE에 원본값)
UPDATE emp SET emp_nm = :emp_nm WHERE emp_no = :emp_no_old

-- DELETE: _old 없이 원본 필드명 (V5가 원본값을 원본 필드명으로 전송)
DELETE FROM emp WHERE emp_no = :emp_no
```

### 삭제 연쇄 (FK 순서)
```sql
-- D execution_order 1: 손자 삭제
DELETE FROM grandchild WHERE fk = :pk;
-- D execution_order 2: 자식 삭제
DELETE FROM child WHERE fk = :pk;
-- D execution_order 3: 본체 삭제
DELETE FROM parent WHERE pk = :pk;
```

### MSSQL → PostgreSQL 변환 (레거시 참조용)

| MSSQL | PostgreSQL |
|-------|-----------|
| GETDATE() | NOW() |
| ISNULL(x,y) | COALESCE(x,y) |
| TOP N | LIMIT N |
| LEN() | LENGTH() |
| CONVERT(type,val) | CAST(val AS type) |
| alias = expr | expr AS alias |
| dbo.fn() | fn() (dbo. 제거) |
| UPDATE a SET...FROM table a | UPDATE table SET...WHERE |
| if/begin/end | {{AND}} 또는 DO블록 |
| DECLARE @var | 서브쿼리 인라인 |
| exec puterror | WHERE 조건 대체 또는 _message 컬럼 |

### _message 컬럼 (R SQL에서 메시지 표시)
```sql
SELECT emp_no, emp_nm, stat_bc,
  CASE WHEN stat_bc = 'HR125400' THEN 'warning' END AS _msg_type,
  CASE WHEN stat_bc = 'HR125400' THEN '퇴직 처리된 사원입니다.' END AS _message
FROM hra100
WHERE dept_cd = :dept_cd
```
- `_msg_type`: info / success / warning / error
- `_message = NULL`인 행은 메시지 없음
- CUD SQL에서는 미지원 → Form Script `toast.*()` 사용
