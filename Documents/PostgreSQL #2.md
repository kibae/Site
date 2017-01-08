# PostgreSQL 작업기 #2
- Cashtree 서비스의 가장 중요한 테이블 중 하나인 **"광고 참여 상태 DB"** 이전 작업에 대한 기록입니다.
- 총 100GB 정도이며 월별로 자동 파티셔닝 되어 있습니다.
    - INSERT: 빈번함, 초당 수십 레코드
    - UPDATE: 빈번함
    - SELECT: 매우 빈번함
- 사용 중인 버전은 9.5, 9.6 입니다.
- 십 수년 PostgreSQL을 쓰며 얻은 노하우들이기 때문에, 최신 기능에 대한 반영이 안되어 있을수 있습니다.
- 무정지 DB 샤딩 및 이전에서 가장 중요한 것은 OLD/NEW DB 셋이 동일한 내용을 유지하게 하는 것입니다. 사실 DBMS의 종류와 상관 없습니다.
- 아래부터 음슴체

## 작업 목표
- 테이블을 2개의 다른 물리적 서버로 분리한다. 샤딩 키는 uid
- **서비스 중지가 없을것**
- **어플리케이션 코드 변경이 거의 없을것**
- **계속되는 데이터 변경이 누락되지 않을것**

## OLD DB 구조
- PostgreSQL의 테이블 상속 기능을 이용해 파티셔닝을 하고 있음
- raw 라는 이름의 schema에 실제 데이터들이 저장되고 user_part 테이블은 데이터 구조만 정의됨

```SQL
CREATE SCHEMA raw; -- 실제 데이터가 저장될 schema

CREATE TABLE user_part (
  seq        BIGSERIAL                NOT NULL,
  uid        BIGINT                   NOT NULL,
  -- ... some fields,
  PRIMARY KEY (seq),
  UNIQUE (uid, adid)
);
CREATE INDEX user_part_idx1 ON user_part (adid, tm DESC);
CREATE INDEX user_part_idx2 ON user_part (needreward) WHERE status = 200;

CREATE OR REPLACE FUNCTION trig_user_part_insert()
  RETURNS TRIGGER AS $$
DECLARE
  tb  TEXT;
  sql TEXT;
BEGIN
  BEGIN
    tb := 'user_part_' || to_char(NEW.tm, 'YYYYMM');
    sql := 'INSERT INTO raw.' || tb || ' SELECT ($1).*';

    EXECUTE sql
    USING NEW;

    EXCEPTION
    WHEN undefined_table
      THEN
        DECLARE
          dt1 DATE;
          dt2 DATE;
          ddl TEXT;
        BEGIN
          dt1 := date_trunc('month', NEW.tm);
          dt2 := dt1 + '1 month' :: INTERVAL;

          ddl :='CREATE TABLE raw.' || tb || ' ( CHECK ( tm >= TIMESTAMP WITH TIME ZONE ' || quote_literal(dt1) ||
                ' AND tm < TIMESTAMP WITH TIME ZONE ' || quote_literal(dt2) ||
                ' ), primary key(seq), UNIQUE(uid, adid)) INHERITS (public.user_part)';

          EXECUTE ddl;
          EXECUTE 'CREATE INDEX ' || tb || '_idx1 ON raw.' || tb || '(adid, tm DESC)';
          EXECUTE 'CREATE INDEX ' || tb || '_idx2 ON raw.' || tb || '(needreward) WHERE status = 200';
          EXECUTE sql
          USING NEW;

          EXCEPTION
          WHEN OTHERS
            THEN
              BEGIN
                EXECUTE sql
                USING NEW;
              END;
        END;
    WHEN unique_violation
      THEN
        RETURN NULL;
  END;
  RETURN NULL;
END;
$$
LANGUAGE plpgsql;

CREATE TRIGGER trig_user_part_ins
BEFORE INSERT ON user_part
FOR EACH ROW EXECUTE PROCEDURE trig_user_part_insert();
```

## NEW DB 구조
- 테이블의 구조나 데이터의 변환이 없기 때문에 동일함

## 작업1 : 오래된 데이터 가져오기
- INSERT/UPDATE 작업이 전혀 없는 오래된 테이블들을 먼저 가져옴
- ```\COPY (SELECT ... FROM ... WHERE uid%2=[0 | 1]) TO '/work/src_YYYYMM_0_OR_1';```
    - raw.user_part_YYYYMM 테이블에서 월 별로
    - uid%2 별로 데이터를 dump
- NEW DB에 그대로 restore

## 작업2 : 변경 가능성이 있는 기간의 오래된 데이터 가져오기
- 최근 달(2017년 1월) 이전이어서 INSERT는 없지만 UPDATE는 일어날 수 있는 데이터
- OLD DB에서 postgres_fdw를 이용해 NEW DB 2대를 연결하고, 변경이 일어나는 정보를 NEW DB들에 그대로 UPDATE
- 데이터가 이전되어서 존재하면 UPDATE 될 것이고, 아직 이전되지 않아서 존재하지 않으면 아무 일이 일어나지 않음

### OLD DB 준비

```SQL
-- OLD DB
CREATE EXTENSION postgres_fdw;

-- shard1 접속 준비
CREATE SERVER fdw_ctree_ad_part1
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (dbname 'DBNAME', host 'HOSTNAME_OF_SHARD1');

CREATE USER MAPPING FOR USERNAME
SERVER fdw_ctree_ad_part1
OPTIONS ( USER 'USERNAME', PASSWORD 'PASSWORD');

-- shard2 접속 준비
CREATE SERVER fdw_ctree_ad_part2
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (dbname 'DBNAME', host 'HOSTNAME_OF_SHARD2');

CREATE USER MAPPING FOR USERNAME
SERVER fdw_ctree_ad_part2
OPTIONS ( USER 'USERNAME', PASSWORD 'PASSWORD');

-- shard1의 테이블을 binding
CREATE FOREIGN TABLE user_part1 (
  seq        BIGSERIAL,
  uid        BIGINT,
  -- ... some fields,
) SERVER fdw_ctree_ad_part1
OPTIONS (table_name 'user_part', use_remote_estimate 'true', updatable 'true'
);

-- shard2의 테이블을 binding
CREATE FOREIGN TABLE user_part2 (
  seq        BIGSERIAL,
  uid        BIGINT,
  -- ... some fields,
) SERVER fdw_ctree_ad_part2
OPTIONS (table_name 'user_part', use_remote_estimate 'true', updatable 'true'
);

-- 트리거 함수
CREATE OR REPLACE FUNCTION trig_user_part_update()
  RETURNS TRIGGER AS $$
BEGIN
  IF new.uid % 2 = 0
  THEN
    UPDATE user_part1
    SET status = new.status, install = new.install, needreward = new.needreward, complete = new.complete
    WHERE seq = new.seq;
  ELSE
    UPDATE user_part2
    SET status = new.status, install = new.install, needreward = new.needreward, complete = new.complete
    WHERE seq = new.seq;
  END IF;
  RETURN NEW;
END;
$$
LANGUAGE plpgsql;
```

### NEW DB 준비
- UPDATE 내역을 실시간으로 전달받기 때문에, 기존처럼 dump -> restore 방식으로 데이터를 가져올 수 없음
- dump 후 restore 하는 사이에 변경되는 데이터가 누락되기 때문
- postgres_fdw를 이용해 NEW DB에서 OLD DB의 데이터를 가져올 수 있게 한다

```SQL
CREATE EXTENSION postgres_fdw;

CREATE SERVER fdw_ctree_ad
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (dbname 'DBNAME', host 'HOSTNAME_OF_OLD');

CREATE USER MAPPING FOR USERNAME
SERVER fdw_ctree_ad
OPTIONS ( USER 'USERNAME', PASSWORD 'PASSWORD');

CREATE FOREIGN TABLE fdw_user_part (
  seq        BIGSERIAL,
  uid        BIGINT,
  -- ... some fields,
) SERVER fdw_ctree_ad
OPTIONS (table_name 'user_part', use_remote_estimate 'true', fetch_size '10000'
);
```

### Data 이전
- 마지막 달의 테이블 전까지 월 별로 아래의 작업을 반복(예: 201610)
    - OLD DB에 트리거 생성
    	- ```CREATE TRIGGER trig_user_part_upd AFTER UPDATE ON raw.user_part_201610 FOR EACH ROW EXECUTE PROCEDURE trig_user_part_update();```
        - UPDATE 되는 내역이 NEW DB들에 반영되기 시작한다.
	- OLD DB의 raw.user_part_201610 파티션 테이블에서 min(PK), max(PK) 값을 구한다
    - NEW DB에 데이터 입력. OLD DB -> NEW DB
        - seq 필드가 PK 이므로 seq로 정렬해서 10,000개 정도씩 가져와서 삽입
	    - ```INSERT INTO raw.user_part_201610 SELECT * FROM fdw_user_part WHERE uid%2=0 AND seq > $1 ... ORDER BY seq LIMIT 10000```
        - ```INSERT INTO raw.user_part_201610 SELECT * FROM fdw_user_part WHERE uid%2=1 AND seq > $1 ... ORDER BY seq LIMIT 10000```

## 작업3 : 빈번하게 입력되고 수정되는 최신 데이터 가져오기
- 서버 프로그램이 **"특정 시간"**부터 OLD DB에 데이터 입력 성공 시 NEW DB에도 insert 한다
    - IF (데이터 입력 && time() >= strtotime('YYYY-MM-DD)) NEW DB에 데이터 입력
- **"특정 시간"**이 지나면 그 시간까지의 데이터를 **작업2**의 방법을 써서 import
- 데이터 검증
- 서버 프로그램에서 DB서버의 endpoint 를 변경

# 끗. 참 쉽쥬?
