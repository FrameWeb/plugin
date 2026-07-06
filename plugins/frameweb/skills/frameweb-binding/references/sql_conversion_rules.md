# MSSQL → PostgreSQL 변환 규칙

> ROCA WorkSet SQL(18,211개) 분석 기반
> Epic BNDQUERY 작성 시 참고

---

## 1. 함수 변환 (빈도순)

```
MSSQL                        PostgreSQL                   빈도
────────────────────────────────────────────────────────────────
getdate()                    now()                        30.3%
isnull(a, b)                 coalesce(a, b)               28.6%
convert(type, val)           val::type  또는  cast(val as type)  5.1%
cast(val as type)            cast(val as type) 또는 val::type    1.5%
dateadd(unit, n, date)       date + interval 'n unit'     1.5%
left(str, n)                 left(str, n)                 1.2% (동일)
substring(str, s, l)         substring(str from s for l)  0.9%
len(str)                     length(str)                  0.8%
datediff(unit, d1, d2)       extract(epoch from d2-d1)    0.6%
                             또는 date_part('unit', d2-d1)
right(str, n)                right(str, n)                0.5% (동일)
rtrim(str)                   rtrim(str)                   0.4% (동일)
stuff(str, s, l, rep)        overlay(str placing rep from s for l)  0.3%
ltrim(str)                   ltrim(str)                   0.1% (동일)
charindex(sub, str)          position(sub in str)         0.1%
@@identity                   currval('seq_name')          0.0%
```

### 자주 쓰는 변환 예시

```sql
-- MSSQL
SELECT isnull(a.dept_nm, '') FROM hra200 a WHERE a.cdt <= getdate()

-- PostgreSQL
SELECT coalesce(a.dept_nm, '') FROM hra200 a WHERE a.cdt <= now()
```

```sql
-- MSSQL
SELECT convert(varchar(10), a.hire_dt, 121)

-- PostgreSQL
SELECT to_char(a.hire_dt, 'YYYY-MM-DD')
```

```sql
-- MSSQL
SELECT dateadd(month, -3, getdate()), datediff(day, a.fr_dt, a.to_dt)

-- PostgreSQL
SELECT now() - interval '3 months', extract(day from a.to_dt - a.fr_dt)
```

---

## 2. 문법 변환

### SELECT TOP N → LIMIT N

```sql
-- MSSQL
SELECT TOP 1 z.up FROM sdp100 z WHERE z.itm_id = a.itm_id ORDER BY z.fr_dt DESC

-- PostgreSQL
SELECT z.up FROM sdp100 z WHERE z.itm_id = a.itm_id ORDER BY z.fr_dt DESC LIMIT 1
```

### UPDATE ... FROM → UPDATE ... FROM (PostgreSQL도 지원하지만 문법 다름)

```sql
-- MSSQL
UPDATE a
SET    a.title = @title, a.mdt = getdate()
FROM   eaa200 a
WHERE  a.stream_id = @stream_id_old

-- PostgreSQL
UPDATE eaa200 a
SET    title = :title, mdt = now()
WHERE  a.stream_id = :stream_id_old
```

**주의**: PostgreSQL UPDATE FROM은 SET에서 `a.col` 대신 `col`만 사용.

### IF EXISTS ... BEGIN ... END → DO $$ 블록 또는 WHERE EXISTS

```sql
-- MSSQL
IF EXISTS (SELECT 0 FROM sda100 WHERE pro_no = @pro_no AND pro_rev > @pro_rev)
    RETURN

-- PostgreSQL (BNDQUERY에서)
DO $$
BEGIN
  IF EXISTS (SELECT 1 FROM sda100 WHERE pro_no = :pro_no AND pro_rev > :pro_rev) THEN
    RAISE EXCEPTION '이미 증가된 차수가 존재합니다';
  END IF;
  -- ...실제 CUD 로직
END $$;
```

### DECLARE + SET → DO $$ 블록

```sql
-- MSSQL (R SQL에서 빈번, 13.3%)
DECLARE @_ex_rt decimal(10,2) = 1
IF isnumeric(@ex_rt) = 1
    SET @_ex_rt = @ex_rt
SELECT a.*, @_ex_rt as ex_rt FROM ...

-- PostgreSQL (Epic BNDQUERY)
-- 방법 1: WITH CTE로 대체
WITH params AS (
  SELECT CASE WHEN :ex_rt ~ '^\d+\.?\d*$' THEN :ex_rt::numeric ELSE 1 END as ex_rt
)
SELECT a.*, p.ex_rt FROM ... , params p

-- 방법 2: DO $$ + TEMP TABLE
DO $$
DECLARE v_ex_rt numeric := 1;
BEGIN
  IF :ex_rt ~ '^\d+\.?\d*$' THEN v_ex_rt := :ex_rt::numeric; END IF;
  CREATE TEMP TABLE _result AS SELECT a.*, v_ex_rt as ex_rt FROM ...;
END $$;
SELECT * FROM _result;
```

### WITH (NOLOCK) → 제거 (0.2%)

```sql
-- MSSQL
SELECT * FROM hra200 a WITH (NOLOCK) WHERE ...

-- PostgreSQL (그냥 제거)
SELECT * FROM hra200 a WHERE ...
```

---

## 3. ROCA 전용 문법 → Epic 대응

### andif ... endif → Epic {{ }} 선택적 WHERE

ROCA의 `andif ... endif`는 파라미터가 비어있으면 해당 조건을 제거하는 프레임워크 기능.
Epic에서는 `{{ }}` 로 동일한 역할.

```sql
-- ROCA
SELECT * FROM ppz100 a
WHERE a.fac_cd = @fac_cd
andif (a.wc_cd like @wc_cd + '%' or a.wc_nm like @wc_cd + '%') endif

-- Epic
SELECT * FROM ppz100 a
WHERE a.fac_cd = :fac_cd
{{AND (a.wc_cd LIKE :wc_cd || '%' OR a.wc_nm LIKE :wc_cd || '%')}}
```

### <1></1> <2></2> 태그 → 조건부 분기

ROCA에서 `<1>`, `<2>`, `<3>` 태그는 검색 모드에 따라 하나만 활성화되는 조건분기.
Epic에서는 단일 `{{ }}` 또는 CASE WHEN으로 대체.

```sql
-- ROCA
WHERE a.fac_cd = @fac_cd
<1>AND a.wh_cd = @wh_cd</1>
<2>AND a.wh_nm = @wh_cd</2>
<3>AND (a.wh_cd LIKE @wh_cd + '%' OR a.wh_nm LIKE @wh_cd + '%')</3>

-- Epic (보통 <3> 방식으로 통합)
WHERE a.fac_cd = :fac_cd
{{AND (a.wh_cd LIKE :wh_cd || '%' OR a.wh_nm LIKE :wh_cd || '%')}}
```

### <$reg_id>, <$lan_no> → Epic 세션 변수

```sql
-- ROCA
INSERT INTO table (..., cid, cdt) VALUES (..., <$reg_id>, getdate())
exec System_Approval @reg_id, @fr_dt, @to_dt, <$lan_no>

-- Epic
INSERT INTO table (..., created_by, created_at) VALUES (..., :reg_id, now())
-- <$reg_id>는 BNDSAVE에서 source_type='SESSION', source_ref='<$reg_id>'
```

### @ 파라미터 → : 파라미터

```sql
-- ROCA
WHERE a.pro_no = @pro_no AND a.pro_rev = @pro_rev

-- Epic
WHERE a.pro_no = :pro_no AND a.pro_rev = :pro_rev
```

---

## 4. 문자열 연결

```sql
-- MSSQL (+ 연산자)
a.wc_cd LIKE @wc_cd + '%'
a.dept_cd + ' ' + a.dept_nm

-- PostgreSQL (|| 연산자)
a.wc_cd LIKE :wc_cd || '%'
a.dept_cd || ' ' || a.dept_nm
```

---

## 5. 데이터 타입 변환

```
MSSQL                  PostgreSQL
────────────────────────────────────
nvarchar(N)            varchar(N)
varchar(N)             varchar(N) (동일)
int                    integer
bit                    boolean
datetime               timestamp
datetime2              timestamp
smalldatetime          timestamp
decimal(p,s)           numeric(p,s)
money                  numeric(19,4)
text                   text (동일)
ntext                  text
image                  bytea
uniqueidentifier       uuid
```

---

## 6. 저장 프로시저 → PL/pgSQL 함수 또는 DO 블록

ROCA SQL의 26.7%가 `EXEC` 사용.

```sql
-- ROCA
EXEC SDA100_2_SDB100 @pro_no, @pro_rev, <$reg_id>

-- Epic 방법 1: PL/pgSQL 함수
SELECT * FROM sda100_to_sdb100(:pro_no, :pro_rev, :reg_id)

-- Epic 방법 2: DO $$ 블록 (결과 반환 필요 시)
DO $$
BEGIN
  -- 비즈니스 로직 인라인
  INSERT INTO sdb100 SELECT ... FROM sda100 WHERE ...;
END $$;
```

---

## 7. 임시 테이블

```sql
-- MSSQL
SELECT * INTO #temp FROM table WHERE ...
SELECT * FROM #temp

-- PostgreSQL
CREATE TEMP TABLE _temp AS SELECT * FROM table WHERE ...;
SELECT * FROM _temp;
DROP TABLE IF EXISTS _temp;
```

**Epic BNDQUERY 패턴** (dispatch에서 사용):
```sql
DO $$
BEGIN
  DROP TABLE IF EXISTS _dsp_result;
  CREATE TEMP TABLE _dsp_result (...);
  INSERT INTO _dsp_result SELECT ...;
END $$;
SELECT * FROM _dsp_result;
```

---

## 8. CUD SQL 패턴 비교

### INSERT

```sql
-- ROCA
INSERT INTO sda150 (pro_no, pro_rev, pro_sq, ..., cid, cdt)
VALUES (@pro_no, @pro_rev, @pro_sq, ..., <$reg_id>, getdate())

-- Epic
INSERT INTO sda150 (pro_no, pro_rev, pro_sq, ..., created_by, created_at)
VALUES (:pro_no, :pro_rev, :pro_sq, ..., :reg_id, now())
```

### UPDATE

```sql
-- ROCA (UPDATE FROM 패턴)
UPDATE a
SET a.col1 = @col1, a.mid = <$reg_id>, a.mdt = getdate()
FROM table a
WHERE a.pk = @pk_old

-- Epic
UPDATE table
SET col1 = :col1, updated_by = :reg_id, updated_at = now()
WHERE pk = :pk_old
```

### DELETE

```sql
-- ROCA
DELETE FROM child WHERE fk = @fk_old
DELETE FROM parent WHERE pk = @pk_old

-- Epic (execution_order로 순서 보장)
-- execution_order=1: DELETE FROM child WHERE fk = :fk_old
-- execution_order=2: DELETE FROM parent WHERE pk = :pk_old
```

---

## 9. 빈도 요약 (변환 우선순위)

```
빈도    패턴                 난이도    비고
────────────────────────────────────────────────────
30.3%   getdate→now          쉬움     기계적 치환
28.6%   isnull→coalesce      쉬움     기계적 치환
26.7%   exec→DO$$/함수       어려움   로직 분석 필요
24.6%   BEGIN/END→DO$$       중간     블록 구조 변환
18.2%   IF EXISTS→DO$$       중간     조건 분기 변환
13.3%   DECLARE→CTE/DO$$    중간     변수를 CTE로 변환
12.3%   andif→{{ }}          쉬움     Epic 전용 문법으로 1:1
10.2%   TOP→LIMIT            쉬움     기계적 치환
 5.2%   SET @→변수 제거      중간     CTE 또는 DO$$ 내부
 5.1%   convert→::캐스트      쉬움     타입 캐스팅
 1.8%   #temp→TEMP TABLE    중간     문법 차이
 0.2%   NOLOCK→제거          쉬움     그냥 삭제
```

### 자동 변환 가능 (기계적 치환, ~60%)
getdate, isnull, TOP, convert, NOLOCK, andif, @→:, +→||

### 수동 판단 필요 (~40%)
exec(프로시저 인라인), BEGIN/END(블록 구조), DECLARE(변수), IF EXISTS(분기), UPDATE FROM(문법)
