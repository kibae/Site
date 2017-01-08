# PostgreSQL 작업기 #1
- Cashtree 서비스의 가장 중요한 테이블 중 하나인 **"리워드 획득 이력 DB"** 이전 작업에 대한 기록입니다.
- 총 500GB 정도이며 월별로 자동 파티셔닝 되어 있습니다.
    - INSERT: 매우 빈번함, 초당 수백 레코드
    - UPDATE: 없음
    - SELECT: 빈번함
- 사용 중인 버전은 9.5, 9.6 입니다.
- 십수 년 PostgreSQL을 쓰며 얻은 노하우들이기 때문에, 최신 기능에 대한 반영이 안 되어 있을수 있습니다.
- 무정지 DB 샤딩 및 이전에서 가장 중요한 것은 OLD/NEW DB 셋이 동일한 내용을 유지하게 하는 것입니다. 사실 DBMS의 종류와 상관없습니다.
- 아래부터 음슴체

## 작업 목표
- 테이블을 2개의 다른 물리적 서버로 분리한다. 샤딩 키는 uid
- 테이블 데이터 이전 시 아래의 데이터 변환을 수행한다.
    - detail이라는 VARCHAR(100) 필드의 내용을 별도의 테이블에 저장하고 그 키값을 저장하게 한다.
    - user_earning -> user_earning_data , user_earning_detail 테이블로 분리
    - before_free_cash(BIGINT) , before_work_cash(BIGINT) 두 개 필드의 값을 합하여 before_cash(INT) 필드로 저장한다.
    - after_free_cash(BIGINT) , after_work_cash(BIGINT) 두 개 필드의 값을 합하여 after_cash(INT) 필드로 저장한다.
- **서비스 중지가 없을 것**
- **어플리케이션 코드 변경이 거의 없을 것**

## OLD DB 구조
- PostgreSQL의 테이블 상속 기능을 이용해 파티셔닝을 하고 있음
- raw 라는 이름의 schema에 실제 데이터들이 저장되고 user_earning 테이블은 데이터 구조만 정의됨

```SQL
--       __   ___  __       ___       __               __
-- |  | /__` |__  |__)     |__   /\  |__) |\ | | |\ | / _`
-- \__/ .__/ |___ |  \ ___ |___ /~~\ |  \ | \| | | \| \__>
CREATE SCHEMA raw;
CREATE TABLE user_earning (
  seq              BIGSERIAL                NOT NULL,
  uid              BIGINT                   NOT NULL,
  tm               TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),

  -- ... some fields,

  before_free_cash BIGINT                   NOT NULL DEFAULT 0,
  before_work_cash BIGINT                   NOT NULL DEFAULT 0,
  after_free_cash  BIGINT                   NOT NULL DEFAULT 0,
  after_work_cash  BIGINT                   NOT NULL DEFAULT 0,

  detail           VARCHAR(100),

  PRIMARY KEY (seq),
  UNIQUE (uid, uniq)
);

CREATE OR REPLACE FUNCTION IS_USING(TEXT)
  RETURNS BOOLEAN AS $$
SELECT LEFT($1, 1) = 'U';
$$
LANGUAGE SQL IMMUTABLE;

-- 자동 테이블 파티셔닝을 위한 트리거
-- user_earning 테이블에 데이터를 입력하면 tm 필드의 시간 정보를 이용해
-- raw.user_earning_YYYYMM 테이블에 데이터를 입력한다
-- 테이블이 없으면 user_earning 테이블을 상속하여 자동으로 생성한다
CREATE OR REPLACE FUNCTION trig_user_earning_insert()
  RETURNS TRIGGER AS $$
DECLARE
  tb TEXT;
BEGIN
  BEGIN
    -- 월별로 분할된 테이블 명 생성
    tb := 'user_earning_' || to_char(NEW.tm, 'YYYYMM');

    -- 테이블에 삽입해 본다
    EXECUTE 'INSERT INTO raw.' || tb || ' SELECT ($1).*'
    USING NEW;

    EXCEPTION
    WHEN undefined_table
      THEN
        -- 테이블이 없다는 에러가 발생하면
        DECLARE
          dt1 DATE;
          dt2 DATE;
          ddl TEXT;
        BEGIN
          dt1 := date_trunc('month', NEW.tm);
          dt2 := dt1 + '1 month' :: INTERVAL;

          -- 테이블을 생성하며 CHECK 제약을 만든다.
          -- 이 CHECK 제약으로 인해 테이블에서 데이터 검색 시 WHERE 조건에 tm 필드가 포함되어 있으면
          -- 많은 하위 테이블들이 있어도 해당 범위의 테이블만 스캔하게 된다
          ddl :='CREATE TABLE raw.' || tb || ' ( CHECK ( tm >= TIMESTAMP WITH TIME ZONE ' || quote_literal(dt1) ||
                ' AND tm < TIMESTAMP WITH TIME ZONE ' || quote_literal(dt2) ||
                ' ), primary key(seq), UNIQUE (uid,uniq)) INHERITS (public.user_earning)';

          EXECUTE ddl;

          -- 인덱스들도 생성
          EXECUTE 'CREATE INDEX ' || tb || '_idx1 ON raw.' || tb || '(uid, tm DESC)';
          EXECUTE 'CREATE INDEX ' || tb || '_idx2 ON raw.' || tb || '(uid, tm DESC) WHERE IS_USING(code) = TRUE';

          -- 다시 삽입해 본다
          EXECUTE 'INSERT INTO raw.' || tb || ' SELECT ($1).*'
          USING NEW;

          EXCEPTION
          WHEN OTHERS
            THEN
              BEGIN
                -- 테이블 생성, 데이터 삽입 중에 에러가 발생할 수 있다
                -- DB 전역 락을 걸지 않았기 때문에 다른 세션에서 테이블 생성을 진행 했을 수 있는데
                -- 그럴 경우 데이터 입력만 다시 시도해 본다
                EXECUTE 'INSERT INTO raw.' || tb || ' SELECT ($1).*'
                USING NEW;
              END;
        END;
  END;

  -- user_earning 테이블 대신 서브 테이블에 데이터가 삽입되었기 때문에 RETURN NULL을 하여
  -- user_earning 테이블에는 데이터가 삽입되지 않게 한다
  -- RETURN NEW; 를 하게 되면 데이터가 이중으로 삽입되는 문제가 생기니 주의
  RETURN NULL;
END;
$$
LANGUAGE plpgsql;

-- 데이터가 입력되기 전에 처리되어야 하므로 BEFORE INSERT
CREATE TRIGGER trig_user_earning_ins
BEFORE INSERT ON user_earning
FOR EACH ROW EXECUTE PROCEDURE trig_user_earning_insert();
```

- 데이터가 입력되면 자동으로 하위 테이블들이 생성됨. 요렇게

```
ctree_earning=> \dt
List of relations
 Schema |          Name          | Type  |
--------+------------------------+-------+
 public | user_earning           | table |
 raw    | user_earning_201607    | table |
 raw    | user_earning_201608    | table |
 raw    | user_earning_201609    | table |
 raw    | user_earning_201610    | table |
 raw    | user_earning_201611    | table |
 raw    | user_earning_201612    | table |
 raw    | user_earning_201701    | table |
```


## NEW DB 구조

```SQL
--  __             __   __          __
-- /__` |__|  /\  |__) |  \ | |\ | / _`
-- .__/ |  | /~~\ |  \ |__/ | | \| \__>
--       __   ___  __       ___       __               __
-- |  | /__` |__  |__)     |__   /\  |__) |\ | | |\ | / _`
-- \__/ .__/ |___ |  \ ___ |___ /~~\ |  \ | \| | | \| \__>

CREATE SCHEMA raw; -- 실제 데이터가 저장될 schema

-- detail 필드의 내용이 저장될 테이블
CREATE TABLE user_earning_detail (
  id  BIGSERIAL    NOT NULL,
  txt VARCHAR(120) NOT NULL,
  PRIMARY KEY (id),
  UNIQUE (txt)
);

-- detail 스트링을 넣으면 user_earning_detail 에서 찾아서 id 값을 주거나
-- 테이블에 추가 후 id 값을 주는 함수
CREATE OR REPLACE FUNCTION detail_key(TEXT)
  RETURNS BIGINT AS $$
DECLARE
  k BIGINT;
BEGIN
  -- NULL 입력 시 NULL 리턴
  IF $1 IS NULL
  THEN
    RETURN NULL;
  END IF;

  k := NULL;
  
  -- 값을 가져와 보고
  SELECT id
  INTO k
  FROM user_earning_detail
  WHERE txt = $1;
  
  -- 없으면 입력
  IF NOT FOUND
  THEN
    INSERT INTO user_earning_detail (txt) VALUES ($1)
    RETURNING id
      INTO k;
  END IF;
  
  RETURN k;
END;
$$
LANGUAGE plpgsql;

-- user_earning_data 원형 테이블
-- 이 테이블 아래로 파티셔닝 된 테이블들이 추가된다
CREATE TABLE user_earning_data (
  seq         BIGSERIAL                NOT NULL,
  uid         BIGINT                   NOT NULL,
  tm          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),

  -- ... some fields,

  before_cash INT                      NOT NULL DEFAULT 0,
  after_cash  INT                      NOT NULL DEFAULT 0,

  detail      INT,

  PRIMARY KEY (seq),
  UNIQUE (uid, uniq)
);
CREATE INDEX user_earning_data_idx1
  ON user_earning_data (uid, tm DESC);

-- 자동 테이블 파티셔닝을 위한 트리거ENV/Doc/PostgreSQL.Work1.md:280
-- user_earning_data 테이블에 데이터를 입력하면 tm 필드의 시간 정보를 이용해
-- raw.user_earning_data_YYYYMM 테이블에 데이터를 입력한다
-- 테이블이 없으면 user_earning_data 테이블을 상속하여 자동으로 생성한다
CREATE OR REPLACE FUNCTION trig_user_earning_data_insert()
  RETURNS TRIGGER AS $$
DECLARE
  tb  TEXT;
  ins TEXT;
BEGIN
  BEGIN
  	-- 월별로 분할된 테이블 명 생성
    tb := 'user_earning_data_' || to_char(NEW.tm, 'YYYYMM');
    ins := 'INSERT INTO raw.' || tb || ' SELECT ($1).*';

	-- 테이블에 삽입해 본다
    EXECUTE ins
    USING NEW;

    EXCEPTION
    WHEN undefined_table
      -- 테이블이 없다는 에러가 발생하면
      THEN
        DECLARE
          dt1 DATE;
          dt2 DATE;
          ddl TEXT;
        BEGIN
          dt1 := date_trunc('month', NEW.tm);
          dt2 := dt1 + '1 month' :: INTERVAL;

          -- 테이블을 생성하며 CHECK 제약을 만든다.
          -- 이 CHECK 제약으로 인해 테이블에서 데이터 검색 시 WHERE 조건에 tm 필드가 포함되어 있으면
          -- 많은 하위 테이블들이 있어도 해당 범위의 테이블만 스캔하게 된다
          ddl :='CREATE TABLE raw.' || tb || ' ( CHECK ( tm >= TIMESTAMP WITH TIME ZONE ' || quote_literal(dt1) ||
                ' AND tm < TIMESTAMP WITH TIME ZONE ' || quote_literal(dt2) ||
                ' ), primary key(seq), UNIQUE (uid,uniq)) INHERITS (public.user_earning_data)';

          EXECUTE ddl;

          -- 인덱스들도 생성
          EXECUTE 'CREATE INDEX ' || tb || '_idx1 ON raw.' || tb || '(uid, tm DESC)';

          -- 다시 삽입해 본다
          EXECUTE ins
          USING NEW;

          EXCEPTION
          WHEN OTHERS
            THEN
              BEGIN
                -- 테이블 생성, 데이터 삽입 중에 에러가 발생할 수 있다
                -- DB 전역 락을 걸지 않았기 때문에 다른 세션에서 테이블 생성을 진행 했을 수 있는데
                -- 그럴 경우 데이터 입력만 다시 시도해 본다
                EXECUTE ins
                USING NEW;
              END;
        END;
  END;
  -- user_earning_data 테이블 대신 서브 테이블에 데이터가 삽입되었기 때문에 RETURN NULL을 하여
  -- user_earning 테이블에는 데이터가 삽입되지 않게 한다
  -- RETURN NEW; 를 하게 되면 데이터가 이중으로 삽입되는 문제가 생기니 주의
  RETURN NULL;
END;
$$
LANGUAGE plpgsql;

-- 데이터가 입력되기 전에 처리되어야 하므로 BEFORE INSERT
CREATE TRIGGER trig_user_earning_data_ins
BEFORE INSERT ON user_earning_data
FOR EACH ROW EXECUTE PROCEDURE trig_user_earning_data_insert();
```


## Writable View 생성
- 기존 코드를 최대한 수정하지 않기 위해 user_earning 이라는 뷰 생성
	- SELECT 시 코드 수정 없음
	- INSERT 시 코드 수정 없기 위해 트리거 생성

```sql
-- user_earning
CREATE VIEW user_earning AS
  SELECT
    seq,
    uid,
    tm,
    -- ... some fields,
    cash,
    0           AS before_free_cash,
    before_cash AS before_work_cash,
    0           AS after_free_cash,
    after_cash  AS after_work_cash,
    txt         AS detail
  FROM user_earning_data a LEFT JOIN user_earning_detail b ON a.detail = b.id;

CREATE OR REPLACE FUNCTION trig_user_earning_insert()
  RETURNS TRIGGER AS $$
BEGIN
  IF new.seq IS NULL
  THEN
    new.seq := nextval('user_earning_data_seq_seq' :: REGCLASS);
  END IF;

  IF new.tm IS NULL
  THEN
    new.tm := now();
  END IF;

  INSERT INTO user_earning_data (seq, uid, tm, code, cash, before_cash, after_cash, uniq, detail) VALUES
    (new.seq, new.uid, new.tm, new.code,
     new.cash, new.before_free_cash + new.before_work_cash, new.after_free_cash + new.after_work_cash, new.uniq,
     detail_key(new.detail));
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 뷰니까 INSTEAD OF
CREATE TRIGGER trig_user_earning_ins
INSTEAD OF INSERT ON user_earning
FOR EACH ROW EXECUTE PROCEDURE trig_user_earning_insert();
```

## 오래된 데이터들을 이전
- ```\COPY (SELECT ... FROM raw.user_earning_YYYYMM WHERE uid%2=[0 | 1]) TO '/work/src_YYYYMM_0_OR_1';```
    - raw.user_earning_YYYYMM 테이블에서 월별로
    - before_cash(free+work), after_cash(free+work) 필드 통합하며
    - uid%2 별로 데이터를 dump
- 오래된 데이터들을 restore
    - 준비된 2대의 NEW DB 서버에서 restore
    - migrate_YYYYMM 테이블로 우선 import 후에
    - detail 데이터를 user_earning_detail 테이블에 삽입해서 id를 생성해 둔다
        - ```INSERT INTO user_earning_detail (txt) SELECT DISTINCT detail FROM migrate_YYYYMM;```
    - ```INSERT INTO user_earning_data ... SELECT migrate_YYYYMM JOIN user_earning_detail ...```
- 서버 프로그램이 **"특정 시간"**부터 OLD DB에 데이터 입력 성공 시 NEW DB에도 insert 한다
    - IF (데이터 입력 && time() >= strtotime('YYYY-MM-DD)) NEW DB에 데이터 입력
- **"특정 시간"**이 지나면 그 시간까지의 데이터를 import
- 데이터 검증
- 서버 프로그램에서 DB서버의 endpoint 를 변경

# 끗. 참 쉽쥬?
- 2편 https://github.com/kibae/Site/blob/master/Documents/PostgreSQL%20%232.md
