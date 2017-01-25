# PostgreSQL 작업기 #3
- Cashtree 서비스에 추가된 출석부 기능 구현
- 출석부 체크인 및 데이터 조회를 하나의 함수로 구현하여, 조회 및 삽입 시점 차이로 인한 데이터 동기화 문제를 해결한다.

## 규칙
- 하루에 1회 출석 체크인한다
- 하루라도 체크인을 빼먹으면 리셋된다
- 최대 연속 10일까지 가능하므로, 11일 연속 체크인 한 경우 리셋되고 1일차로 바뀜
- 출석 체크 후 1시간 단위로 룰렛을 돌려서 리워드를 받을수 있음. 룰렛에 대한 데이터 구조는 생략

## 테이블 구조
- 유저 당 1개의 row가 생성되므로 샤딩하지 않는다

```SQL
--      ___ ___  ___       __             __   ___
--  /\   |   |  |__  |\ | |  \  /\  |\ | /  ` |__
-- /~~\  |   |  |___ | \| |__/ /~~\ | \| \__, |___
CREATE TABLE attendance (
  uid             BIGINT                   NOT NULL, -- User ID
  today           DATE                     NOT NULL, -- 마지막 체크인 날짜
  checkin         DATE [10]                NOT NULL DEFAULT '{}' :: DATE [], -- 연속된 체크인 날짜들 배열
  tm              TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(), -- 마지막 체크인 시간
  PRIMARY KEY (uid)
);

-- 체크인하며 데이터를 가져오는 함수. attendance 테이블의 row를 리턴한다
CREATE OR REPLACE FUNCTION attendance_checkin(BIGINT, DATE)
  RETURNS attendance AS $$
DECLARE
  row attendance;
BEGIN
  UPDATE attendance
  SET
    -- 오늘 날짜를 설정
    today           = $2,
    -- 체크인 하려는 날짜가 기존 날짜와 다르면 tm 갱신
    tm              = (CASE WHEN today = $2 THEN tm ELSE now() END),
    -- 체크인 하려는 날짜가 기존 날짜와 다르면 checkin 배열에 오늘 날짜 추가하되
    -- 마지막 체크인 기록 날짜 배열 요소와 1일 이상 차이가 나거나
    -- 체크인 기록 날짜 배열 기존 요소 수가 10 이상이면 배열을 비우고 마지막 날짜만 추가
    checkin         = (CASE WHEN today = $2 THEN checkin ELSE (
        CASE WHEN (ARRAY_LENGTH(checkin, 1) IS NOT NULL AND
                   ARRAY_LENGTH(checkin, 1) > 0 AND
                   ($2 :: TIMESTAMP - checkin [ARRAY_LENGTH(checkin, 1)] :: TIMESTAMP) > '1 DAY' :: INTERVAL) OR
                  (ARRAY_LENGTH(checkin, 1) >= 10)
          THEN ARRAY [$2] :: DATE []
        ELSE ARRAY_APPEND(checkin, $2) END
      ) END
    )
  WHERE uid = $1 RETURNING * INTO row;

  -- 기존 row가 없어서 UPDATE 결과가 없다면 추가한다
  IF NOT FOUND
  THEN
    INSERT INTO attendance (uid, today, checkin) VALUES ($1, $2, ARRAY [$2] :: DATE [])
    RETURNING *
      INTO row;
  END IF;

  -- row를 리턴
  RETURN row;
END;
$$
LANGUAGE plpgsql;
```

## 체크인, 데이터 가져오기
- 이 쿼리를 이용해서 데이터를 조회한다. 체크인이 안되어 있으면 체크인이 되고, 되어 있으면 변화 없는 데이터를 준다
- ```$1::BIGINT``` User ID
- ```$2::DATE``` 오늘(체크인 할) 날짜

```SQL
SELECT *, ARRAY_LENGTH(checkin, 1) AS days FROM attendance_checkin($1::BIGINT,$2::DATE)
```

