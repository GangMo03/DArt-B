# SQL_MASTER 3주차 정규과제

📌SQL MASTER 정규과제는 매주 정해진 분량의 『*데이터 분석을 위한 SQL 레시피*』 를 읽고 학습하는 것입니다. 이번 주는 아래의 **SQL_MASTER_3rd_TIL**에 나열된 분량을 읽고 공부하시면 됩니다.

아래 실습을 수행하며 학습 내용을 직접 적용해보세요. 단순히 결과를 재현하는 것이 아니라, SQL을 직접 작성하는 과정에서 개념을 스스로 정리하는 것이 중요합니다.

필요한 경우 교재와 추가 자료를 참고하여 이해를 보완하시기 바랍니다.

## SQL_MASTER_3rd_TIL

### 5장 사용자를 파악하기 위한 데이터 추출
#### 1. 사용자 전체의 특징과 경향 찾기


## Study Schedule

| 주차  | 공부 범위     | 완료 여부 |
| ----- | ------------- | --------- |
| 1주차 | p.52~136   | ✅         |
| 2주차 | p.138~184  | ✅         |
| 3주차 | p.186~232 | ✅         |
| 4주차 | p.233~321 | 🍽️         |
| 5주차 | p.324~406 | 🍽️         |
| 6주차 | p.408~464 | 🍽️         |
| 7주차 | p.466~566 | 🍽️         |

<br>

<!-- 여기까진 그대로 둬 주세요-->


# 실습

## 0. 실습 규칙

1. 샘플 데이터 생성 코드는 **08_SQL_MASTER_Template/src** 경로에 장별로 정리되어 있습니다.
2. 아래 목차에 맞춰 해당 코드를 실행하여 샘플 데이터를 생성한 후, 각 장에서 요구하는 쿼리를 직접 작성해보시기 바랍니다.
3. 작성한 쿼리의 **실행 결과 화면도 함께 제출**해 주세요.
4. 단순히 교재의 예시 코드를 그대로 작성하는 것이 아니라, **제시된 로직을 충분히 이해한 뒤 교재를 보지 않고 스스로 쿼리를 구성**해보는 것을 권장합니다.
5. 교재 예시는 PostgreSQL, Hive, BigQuery 등 다양한 DBMS 기준으로 제시되어 있기 때문에, **MySQL이 아닌 다른 SQL 환경을 사용하여 실습을 진행해도 무방합니다.**
6. 다만, 사용 중인 DBMS에 맞는 문법으로 적절히 변환하여 작성하시기 바랍니다.

## 1. 사용자 전체의 특징과 경향 찾기

### 1-1 사용자의 액션 수 집계하기

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
-- 액션 수와 비율을 계산하는 쿼리
WITH stats AS (          
  SELECT  
    COUNT(DISTINCT session) AS total_uu 
  FROM `3rd_week.action_log`
)
SELECT
  l.action,
  COUNT(DISTINCT l.session) AS action_uu,
  COUNT(1) AS action_count,
  s.total_uu,
  100.0 * COUNT(DISTINCT l.session) / s.total_uu AS usage_rate, -- 전체 사용자 대비 액션 사용률(%): <액션 UU> / <전체 UU>
  1.0 * COUNT(1) / COUNT(DISTINCT l.session) AS count_per_user  -- 해당 액션을 수행한 유저 1인당 평균 실행 횟수: <액션 수> / <액션 UU>
FROM `3rd_week.action_log` AS l
CROSS JOIN stats AS s                                           -- 로그 전체의 유니크 사용자 수를 모든 레코드에 결합하기
GROUP BY
  l.action,
  s.total_uu    
```

<!-- 이 부분을 지우고 실행 결과 화면을 제출해주세요. -->

### 1-2 연령별 구분 집계하기

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
-- 로그인 상태에 따라 액션 수 등을 따로 집계하는 쿼리
WITH action_log_with_status AS (
  SELECT
    session,
    user_id, 
    action,
    CASE
      WHEN COALESCE(user_id, '') <> '' THEN 'login'
      ELSE 'guest'
    END AS login_status  
  FROM `3rd_week.action_log` 
)

-- 2. ROLLUP을 통한 계층별 소계/총계 집계
SELECT
  -- ROLLUP에 의해 합산되어 NULL로 표시되는 행을 'all'로 치환
  COALESCE(action, 'all')       AS action,
  COALESCE(login_status, 'all') AS login_status,
  COUNT(DISTINCT session)       AS action_uu,  
  COUNT(1)                      AS action_count 

FROM action_log_with_status  

-- BigQuery 표준 SQL은 ROLLUP(컬럼1, 컬럼2) 문법을 사용
GROUP BY ROLLUP(action, login_status)                                 -- (action, status)별 상세 -> action별 소계 -> 전체 총계 생성
ORDER BY
  action, login_status;
```

<!-- 이 부분을 지우고 실행 결과 화면을 제출해주세요. -->

### 1-3 연령별 구분의 특징 추출하기

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH action_log_with_status AS (
  SELECT
    session,
    user_id,
    action,

    -- [회원 여부(member_status) 판정 로직]
    CASE 
      WHEN COALESCE(
        -- 동일 세션 내에서 시간(stamp) 순서대로 첫 행부터 현재 행까지 user_id의 최댓값을 누적 탐색
        -- (한 번이라도 로그인하여 user_id가 찍혔다면 그 이후 행들에도 해당 user_id가 계속 유지되어 전파됨)
        MAX(user_id) OVER(
          PARTITION BY session 
          ORDER BY stamp 
          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ), 
        '' -- 누적된 user_id가 없으면(NULL) 빈 문자열('')로 치환
      ) <> '' THEN 'member' -- 현재 행 시점까지 로그인한 이력이 존재하면 'member'
      ELSE 'none'           -- 아직 로그인한 적이 없다면 'none'
    END AS member_status,

    stamp
  FROM
    `sql-master-507514.3rd_week.action_log`
)
SELECT *
FROM action_log_with_status;
```

<!-- 이 부분을 지우고 실행 결과 화면을 제출해주세요. -->
 
### 1-4 사용자의 방문 빈도 집계하기

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
-- 한 주에 며칠 사용되었는지를 집계하는 쿼리
WITH action_log_with_dt AS (
  SELECT
    user_id,
    action,
    SUBSTR(stamp, 1, 10) AS dt
  FROM
    `sql-master-507514.3rd_week.action_log`
),
action_day_count_per_user AS (
  SELECT
    user_id,
    COUNT(DISTINCT dt) AS action_day_count     -- 해당 기간 내 사용자가 실제로 활동(접속/액션)한 고유 일수 집계
  FROM
  FROM
    action_log_with_dt
  WHERE
    dt BETWEEN '2016-11-01' AND '2016-11-07'
  GROUP BY
    user_id
)
SELECT
  action_day_count,
  COUNT(DISTINCT user_id) AS user_count    -- 해당 활동 일수 구간에 속하는 고유 사용자 수 집계
FROM
  action_day_count_per_user
GROUP BY
  action_day_count
ORDER BY
  action_day_count;
```

<!-- 이 부분을 지우고 실행 결과 화면을 제출해주세요. -->

### 1-5 벤 다이어그램으로 사용자 액션 집계하기

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
-- 사용자들의 액션 플래그를 집계하는 쿼리
WITH user_action_flag AS (
  SELECT
    user_id,
    SIGN(SUM(CASE WHEN action = 'purchase' THEN 1 ELSE 0 END)) AS has_purchase, -- 구매(purchase) 액션 이력 유무 플래그 (1회 이상: 1, 0회: 0)
    SIGN(SUM(CASE WHEN action = 'review'   THEN 1 ELSE 0 END)) AS has_review,   -- 리뷰(review) 작성 이력 유무 플래그 (1회 이상: 1, 0회: 0)
    SIGN(SUM(CASE WHEN action = 'favorite' THEN 1 ELSE 0 END)) AS has_favorite -- 찜/즐겨찾기(favorite) 이력 유무 플래그 (1회 이상: 1, 0회: 0)
  FROM
    `sql-master-507514.3rd_week.action_log`
  GROUP BY
    user_id
)
SELECT *
FROM user_action_flag;
```

<!-- 이 부분을 지우고 실행 결과 화면을 제출해주세요. -->

### 1-6 Decile 분석을 사용해 사용자를 10단계 그룹으로 나누기

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH user_purchase_amount AS (       -- 사용자를 구매 금액이 많은 순서로 정렬
  SELECT 
    user.id,
    SUM(amount) AS purchase_amount 
  FROM
    `sql-master-507514.3rd_week.action_log`
  WHERE 
    action = 'purchase'
  GROUP BY 
    user.id 
),
users_with_decile AS (              -- 정렬된 사용자의 상위에서 10%씩 Decile1부터 Decile10까지의 그룹을 할당
  SELECT
    user.id,
    purchase_amount,
    NTILE(10) OVER (ORDER BY purchase_amount DESC) AS decile 
  FROM
    user_purchase_amount
)
SELECT *
FROM users_with_decile
```

<!-- 이 부분을 지우고 실행 결과 화면을 제출해주세요. -->

### 1-7 RFM 분석으로 사용자를 3가지 관점의 그룹으로 나누기

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
-- 사용자별로 RFM을 집계하는 쿼리
WITH
purchase_log AS (
  SELECT
    user_id,
    amount,    
    SUBSTR(stamp, 1, 10) AS dt   -- SUBSTR로 날짜 부분(YYYY-MM-DD) 10자리 추출
  FROM
    `sql-master-507514.3rd_week.action_log`
  WHERE
    action = 'purchase'
)
, user_rfm AS (
  SELECT
    user_id,
    MAX(dt) AS recent_date,
    DATE_DIFF(CURRENT_DATE(), DATE(TIMESTAMP(MAX(dt))), DAY) AS recency,  -- DATE_DIFF(종료일, 시작일, DAY)를 사용하여 오늘 기준 경과일 수(Recency) 계산
    COUNT(dt) AS frequency,
    SUM(amount) AS monetary
  FROM
    purchase_log
  GROUP BY
    user_id
)
SELECT *
FROM
  user_rfm
;
```

<!-- 이 부분을 지우고 실행 결과 화면을 제출해주세요. -->


### 🎉 수고하셨습니다.