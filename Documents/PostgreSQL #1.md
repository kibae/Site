# PostgreSQL 작업기
- Cashtree 서비스의 가장 중요한 테이블 중 두가지에 대한 작업 기록입니다.
- 십 수년 PostgreSQL을 쓰며 얻은 노하우들이기 때문에, 최신 기능에 대한 반영이 안되어 있을수 있습니다.
- 사용 중인 버전은 9.5, 9.6 입니다.

# 작업  1. 리워드 획득 이력
- Cashtree의 가장 중요한 테이블 중 하나인 리워드 획득 이력 테이블입니다. 초 당 수백개의 레코드가 쌓이고 있고 파티셔닝 되어 있습니다.
- 데이터의 양이 크지만 한 번 입력된 레코드는 변경되지 않는다는 작업 상 장점(?)이 있습니다.

## 목표
- 테이블을 2개의 다른 물리적 서버로 분리한다.
- 테이블 데이터 이전 시 아래의 데이터 변환을 수행한다.
    - detail 이라는 VARCHAR(100) 필드의 내용을 별도의 테이블에 저장하고 그 키값을 저장하게 한다.
    - user_earning -> user_earning_data , user_earning_detail 테이블로 분리
    - before_free_cash(BIGINT) , before_work_cash(BIGINT) 두 개 필드의 값을 합하여 before_cash(INT) 필드로 저장한다.
    - after_free_cash(BIGINT) , after_work_cash(BIGINT) 두 개 필드의 값을 합하여 after_cash(INT) 필드로 저장한다.
- 서비스 중지가 없을것

## 기존 테이블 구조
- PostgreSQL의 테이블 상속 기능을 이용해 파티셔닝을 하고 있다.
- raw 라는 이름의 schema에 실제 데이터들이 저장되고 user_earning 테이블은 데이터 구조만 정의되어 있다.

