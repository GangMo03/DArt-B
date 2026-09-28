# SQL_MASTER 4주차 정규과제

📌SQL MASTER 정규과제는 매주 정해진 분량의 『*데이터 분석을 위한 SQL 레시피*』 를 읽고 학습하는 것입니다. 이번 주는 아래의 **SQL_MASTER_4th_TIL**에 나열된 분량을 읽고 공부하시면 됩니다.

아래 실습을 수행하며 학습 내용을 직접 적용해보세요. 단순히 결과를 재현하는 것이 아니라, SQL을 직접 작성하는 과정에서 개념을 스스로 정리하는 것이 중요합니다.

필요한 경우 교재와 추가 자료를 참고하여 이해를 보완하시기 바랍니다.

## SQL_MASTER_4th_TIL

### 5장 사용자를 파악하기 위한 데이터 추출
#### 2. 시계열에 따른 사용자 전체의 상태 변화 찾기
#### 3. 시계열에 따른 사용자의 개별적인 행동 분석하기 


## Study Schedule

| 주차  | 공부 범위     | 완료 여부 |
| ----- | ------------- | --------- |
| 1주차 | p.52~136   | ✅         |
| 2주차 | p.138~184  | ✅         |
| 3주차 | p.186~232 | ✅         |
| 4주차 | p.233~321 | ✅         |
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

## 2. 시계열에 따른 사용자 전체의 상태 변화 찾기

### 2-1 등록 수의 추이와 경향 보기

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql

SELECT
    register_date,
    COUNT(DISTINCT user_id) AS register_count
FROM `4th_week.mst_users`
GROUP BY register_date
ORDER BY register_date;
```

![](images/스크린샷%202026-09-28%20112735.png)

### 2-2 지속률과 정착률 산출하기

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql

WITH action_log_with_mst_users AS (
    SELECT
        u.user_id,
        u.register_date,
        -- 한번 타임스탬프 자료형으로 변환하고 날짜 자료형으로 변환하기
        DATE(TIMESTAMP(a.stamp)) AS action_date,
        MAX(DATE(TIMESTAMP(a.stamp))) OVER() AS latest_date,
        --  등록일 다음날의 날짜 계산하기
        DATE_ADD(CAST(u.register_date AS DATE), INTERVAL 1 DAY) AS next_day_1
    FROM
        `4th_week.mst_users` AS u
    LEFT OUTER JOIN
        `4th_week.action_log` AS a
    ON
        u.user_id = a.user_id
)
SELECT *
FROM
    action_log_with_mst_users
ORDER BY
    register_date;
```

![](images/스크린샷%202026-09-28%20112810.png)

### 2-3 지속과 정착에 영향을 주는 액션 집계하기 

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql

WITH repeat_interval AS (
    SELECT '01 day repeat' AS index_name, 1 AS interval_begin_date, 1 AS interval_end_date
),
action_log_with_index_date AS (
    SELECT
        u.user_id,
        u.register_date,
        DATE(TIMESTAMP(a.stamp)) AS action_date,
        MAX(DATE(TIMESTAMP(a.stamp))) OVER() AS latest_date,
        r.index_name,
        DATE_ADD(CAST(u.register_date AS DATE), INTERVAL r.interval_begin_date DAY) AS index_begin_date,
        DATE_ADD(CAST(u.register_date AS DATE), INTERVAL r.interval_end_date DAY) AS index_end_date
    FROM
        `4th_week.mst_users` AS u
    LEFT OUTER JOIN
        `4th_week.action_log` AS a
    ON
        u.user_id = a.user_id
    CROSS JOIN
        repeat_interval AS r
),
user_action_flag AS (
    SELECT
        user_id,
        register_date,
        index_name,
        SIGN(
            SUM(
                CASE WHEN index_end_date <= latest_date THEN
                    CASE WHEN action_date BETWEEN index_begin_date AND index_end_date THEN 1 ELSE 0 END
                END
            )
        ) AS index_date_action
    FROM
        action_log_with_index_date
    GROUP BY
        user_id, register_date, index_name, index_begin_date, index_end_date
),
mst_actions AS (
    SELECT 'view' AS action
    UNION ALL SELECT 'comment' AS action
    UNION ALL SELECT 'follow' AS action
),
mst_user_actions AS (
    SELECT
        u.user_id,
        u.register_date,
        a.action
    FROM
        `4th_week.mst_users` AS u
    CROSS JOIN
        mst_actions AS a
)
SELECT *
FROM mst_user_actions
ORDER BY user_id, action;
```

![](images/스크린샷%202026-09-28%20112841.png)
 
### 2-4 액션 수에 따른 정착률 집계하기 

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql

WITH repeat_interval AS (
    SELECT '14 day retention' AS index_name, 8 AS interval_begin_date, 14 AS interval_end_date
),
action_log_with_index_date AS (
    SELECT
        u.user_id,
        u.register_date,
        DATE(TIMESTAMP(a.stamp)) AS action_date,
        MAX(DATE(TIMESTAMP(a.stamp))) OVER() AS latest_date,
        r.index_name,
        DATE_ADD(CAST(u.register_date AS DATE), INTERVAL r.interval_begin_date DAY) AS index_begin_date,
        DATE_ADD(CAST(u.register_date AS DATE), INTERVAL r.interval_end_date DAY) AS index_end_date
    FROM
        `4th_week.mst_users` AS u
    LEFT OUTER JOIN
        `4th_week.action_log` AS a
    ON
        u.user_id = a.user_id
    CROSS JOIN
        repeat_interval AS r
),
user_action_flag AS (
    SELECT
        user_id,
        register_date,
        index_name,
        SIGN(
            SUM(
                CASE WHEN index_end_date <= latest_date THEN
                    CASE WHEN action_date BETWEEN index_begin_date AND index_end_date THEN 1 ELSE 0 END
                END
            )
        ) AS index_date_action
    FROM
        action_log_with_index_date
    GROUP BY
        user_id, register_date, index_name, index_begin_date, index_end_date
),
mst_action_bucket AS (
    -- 액션 단계 마스터 (BigQuery의 경우 SELECT 구문과 UNION ALL로 대체)
    SELECT 'comment' AS action, 0 AS min_count, 0 AS max_count UNION ALL
    SELECT 'comment', 1, 5 UNION ALL
    SELECT 'comment', 6, 10 UNION ALL
    SELECT 'comment', 11, 9999 UNION ALL
    SELECT 'follow', 0, 0 UNION ALL
    SELECT 'follow', 1, 5 UNION ALL
    SELECT 'follow', 6, 10 UNION ALL
    SELECT 'follow', 11, 9999
),
mst_user_action_bucket AS (
    -- 사용자 마스터와 액션 단계 마스터 조합하기
    SELECT
        u.user_id,
        u.register_date,
        a.action,
        a.min_count,
        a.max_count
    FROM
        `4th_week.mst_users` AS u
    CROSS JOIN
        mst_action_bucket AS a
)
SELECT *
FROM
    mst_user_action_bucket
ORDER BY
    user_id, action, min_count;
```

![](images/스크린샷%202026-09-28%20112907.png)

### 2-5 사용 일수에 따른 정착률 집계하기 

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql

WITH repeat_interval AS (
    SELECT '28 day retention' AS index_name, 22 AS interval_begin_date, 28 AS interval_end_date
),
action_log_with_index_date AS (
    SELECT
        u.user_id,
        u.register_date,
        DATE(TIMESTAMP(a.stamp)) AS action_date,
        MAX(DATE(TIMESTAMP(a.stamp))) OVER() AS latest_date,
        r.index_name,
        DATE_ADD(CAST(u.register_date AS DATE), INTERVAL r.interval_begin_date DAY) AS index_begin_date,
        DATE_ADD(CAST(u.register_date AS DATE), INTERVAL r.interval_end_date DAY) AS index_end_date
    FROM
        `4th_week.mst_users` AS u
    LEFT OUTER JOIN
        `4th_week.action_log` AS a
    ON
        u.user_id = a.user_id
    CROSS JOIN
        repeat_interval AS r
),
user_action_flag AS (
    SELECT
        user_id,
        register_date,
        index_name,
        SIGN(
            SUM(
                CASE WHEN index_end_date <= latest_date THEN
                    CASE WHEN action_date BETWEEN index_begin_date AND index_end_date THEN 1 ELSE 0 END
                END
            )
        ) AS index_date_action
    FROM
        action_log_with_index_date
    GROUP BY
        user_id, register_date, index_name, index_begin_date, index_end_date
),
register_action_flag AS (
    -- 등록일 다음날부터 7일 동안의 사용 일수와 28일 정착 플래그를 생성
    SELECT
        m.user_id,
        -- BigQuery의 경우 다음과 같이 사용하기
        COUNT(DISTINCT DATE(TIMESTAMP(a.stamp))) AS dt_count,
        f.index_name,
        f.index_date_action
    FROM
        `4th_week.mst_users` AS m
    LEFT JOIN
        `4th_week.action_log` AS a
    ON
        m.user_id = a.user_id
        -- 등록 다음날부터 7일 이내의 액션 로그 결합하기
        -- BigQuery의 경우 다음과 같이 사용하기
        AND DATE(TIMESTAMP(a.stamp)) BETWEEN DATE_ADD(CAST(m.register_date AS DATE), INTERVAL 1 DAY) 
                                     AND DATE_ADD(CAST(m.register_date AS DATE), INTERVAL 8 DAY)
    LEFT JOIN
        user_action_flag AS f
    ON
        m.user_id = f.user_id
    WHERE
        f.index_date_action IS NOT NULL
    GROUP BY
        m.user_id,
        f.index_name,
        f.index_date_action
)
SELECT *
FROM
    register_action_flag;
```

![](images/스크린샷%202026-09-28%20112926.png)

### 2-6 사용자의 잔존율 집계하기 

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH mst_intervals AS (
    -- Hive, Redshift, BigQuery, SparkSQL의 경우 SELECT 구문과 UNION ALL로 대체 가능
    SELECT 1 AS interval_month UNION ALL
    SELECT 2 UNION ALL
    SELECT 3 UNION ALL
    SELECT 4 UNION ALL
    SELECT 5 UNION ALL
    SELECT 6 UNION ALL
    SELECT 7 UNION ALL
    SELECT 8 UNION ALL
    SELECT 9 UNION ALL
    SELECT 10 UNION ALL
    SELECT 11 UNION ALL
    SELECT 12
)
SELECT *
FROM mst_intervals;
```

![](images/스크린샷%202026-09-28%20112943.png)

### 2-7 방문 빈도를 기반으로 사용자 속성을 정의하고 집계하기

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH monthly_user_action AS (
    -- 월별 사용자 액션 집약하기
    SELECT DISTINCT
        u.user_id,
        SUBSTR(u.register_date, 1, 7) AS register_month,
        SUBSTR(l.stamp, 1, 7) AS action_month,
        SUBSTR(CAST(
            DATE_SUB(DATE(TIMESTAMP(l.stamp)), INTERVAL 1 MONTH)
        AS STRING), 1, 7) AS action_month_priv
    FROM
        `4th_week.mst_users` AS u
    JOIN
        `4th_week.action_log` AS l
    ON
        u.user_id = l.user_id
),
monthly_user_with_type AS (
    -- 월별 사용자 분류 테이블
    SELECT
        action_month,
        user_id,
        CASE
            -- 등록 월과 액션월이 일치하면 신규 사용자
            WHEN register_month = action_month THEN 'new_user'
            -- 이전 월에 액션이 있다면 리피트 사용자
            WHEN action_month_priv = LAG(action_month) OVER(PARTITION BY user_id ORDER BY action_month) THEN 'repeat_user'
            -- 이외의 경우는 컴백 사용자
            ELSE 'come_back_user'
        END AS c,
        action_month_priv
    FROM
        monthly_user_action
)
SELECT
    action_month,
    -- 특정 달의 MAU
    COUNT(user_id) AS mau,
    -- ===============================================
    -- new_users: 특정 달의 신규 사용자 수
    -- repeat_users: 특정 달의 리피트 사용자 수
    -- come_back_users: 특정 달의 컴백 사용자 수
    -- ===============================================
    COUNT(CASE WHEN c = 'new_user' THEN 1 END) AS new_users,
    COUNT(CASE WHEN c = 'repeat_user' THEN 1 END) AS repeat_users,
    COUNT(CASE WHEN c = 'come_back_user' THEN 1 END) AS come_back_users
FROM
    monthly_user_with_type
GROUP BY
    action_month
ORDER BY
    action_month;
```
![](images/스크린샷%202026-09-28%20113002.png)

### 2-8 방문 종류를 기반으로 성장지수 집계하기 

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql 
WITH unique_action_log AS (
    -- 같은 날짜 로그를 중복해 세지 않도록 중복 배제하기
    SELECT DISTINCT
        user_id,
        -- BigQuery의 경우 다음과 같이 사용하기
        SUBSTR(stamp, 1, 10) AS action_date
    FROM
        `4th_week.action_log`
),
mst_calendar AS (
    -- 집계하고 싶은 기간을 캘린더 테이블로 만들어두기
    -- generate_series 등으로 동적 생성도 가능 (BigQuery: UNNEST(GENERATE_DATE_ARRAY(...)) 활용 가능)
    SELECT '2016-10-01' AS dt UNION ALL
    SELECT '2016-10-02' UNION ALL
    SELECT '2016-10-03' UNION ALL
    -- ... (필요한 날짜만큼 추가) ...
    SELECT '2016-11-04'
),
target_date_with_user AS (
    -- 사용자 마스터에 캘린더 테이블의 날짜를 target_date로 추가하기
    SELECT
        c.dt AS target_date,
        u.user_id,
        u.register_date,
        u.withdraw_date
    FROM
        `4th_week.mst_users` AS u
    CROSS JOIN
        mst_calendar AS c
),
user_status_log AS (
    SELECT
        u.target_date,
        u.user_id,
        u.register_date,
        u.withdraw_date,
        a.action_date,
        CASE WHEN u.register_date = a.action_date THEN 1 ELSE 0 END AS is_new,
        CASE WHEN u.withdraw_date = a.action_date THEN 1 ELSE 0 END AS is_exit,
        CASE WHEN u.target_date   = a.action_date THEN 1 ELSE 0 END AS is_access,
        LAG(CASE WHEN u.target_date = a.action_date THEN 1 ELSE 0 END)
            OVER(PARTITION BY u.user_id ORDER BY u.target_date) AS was_access
    FROM
        target_date_with_user AS u
    LEFT JOIN
        unique_action_log AS a
    ON
        u.user_id = a.user_id
        AND u.target_date = a.action_date
    WHERE
        -- 집계 기간을 등록일 이후로만 필터링하기
        u.register_date <= u.target_date
        -- 탈퇴 날짜가 포함되어 있으면, 집계 기간을 탈퇴 날짜 이전만으로 필터링하기
        AND (
            u.withdraw_date IS NULL
            OR u.target_date <= u.withdraw_date
        )
)
SELECT
    target_date,
    user_id,
    is_new,
    is_exit,
    is_access,
    was_access
FROM
    user_status_log;
```
![](images/스크린샷%202026-09-28%20113020.png)

### 2-9 지표 개선 방법 익히기 

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH user_metric AS (
    SELECT
        u.user_id,
        MAX(
            CASE
                WHEN a.action = 'follow'
                 AND DATE(a.stamp) = u.register_date
                THEN 1
                ELSE 0
            END
        ) AS did_follow,
        MAX(
            CASE
                WHEN DATE(a.stamp) = DATE_ADD(
                    u.register_date,
                    INTERVAL 1 DAY
                )
                THEN 1
                ELSE 0
            END
        ) AS next_day_repeat
    FROM mst_users_retention AS u
    LEFT JOIN action_log_retention AS a
        ON u.user_id = a.user_id
    GROUP BY u.user_id
)
SELECT
    did_follow,
    COUNT(*) AS users,
    ROUND(
        100.0 * AVG(next_day_repeat),
        2
    ) AS next_day_repeat_rate
FROM user_metric
GROUP BY did_follow
ORDER BY did_follow DESC;
```


## 3. 시계열에 따른 사용자의 개별적인 행동 분석하기 

### 3-1 사용자의 액션 간격 집계하기

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH reservations AS (
    -- BigQuery의 경우 VALUES 구문 대신 UNION ALL을 사용하여 일시 테이블 생성
    SELECT 1 AS reservation_id, DATE '2016-09-01' AS register_date, DATE '2016-10-01' AS visit_date, 3 AS days UNION ALL
    SELECT 2, DATE '2016-09-20', DATE '2016-10-01', 2 UNION ALL
    SELECT 3, DATE '2016-09-30', DATE '2016-11-20', 2 UNION ALL
    SELECT 4, DATE '2016-10-01', DATE '2017-01-03', 2 UNION ALL
    SELECT 5, DATE '2016-11-01', DATE '2016-12-28', 3
)
SELECT
    reservation_id,
    register_date,
    visit_date,
    -- BigQuery의 경우 DATE_DIFF 함수 사용하기
    DATE_DIFF(DATE(TIMESTAMP(visit_date)), DATE(TIMESTAMP(register_date)), DAY) AS lead_time
FROM
    reservations;
```

![](images/스크린샷%202026-09-28%20113042.png)

### 3-2 카트 추가 후에 구매했는지 파악하기 

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH row_action_log AS (
    SELECT
        dt,
        user_id,
        action,
        product_id,
        stamp
    FROM
        `4th_week.action_log`
    -- 쉼표로 구분된 product_id 리스트 전개하기 (BigQuery의 경우 CROSS JOIN unnest 사용)
    CROSS JOIN UNNEST(SPLIT(products, ',')) AS product_id
),
action_time_stats AS (
    -- 사용자와 상품 조합의 카드 추가 시간과 구매 시간 추출하기
    SELECT
        user_id,
        product_id,
        MIN(CASE action WHEN 'add_cart' THEN dt END) AS dt,
        MIN(CASE action WHEN 'add_cart' THEN stamp END) AS add_cart_time,
        MIN(CASE action WHEN 'purchase' THEN stamp END) AS purchase_time,
        -- BigQuery의 경우 unix_seconds 함수로 초 단위 UNIX 시간 추출 후 차이 구하기
        MIN(CASE action WHEN 'purchase' THEN UNIX_SECONDS(TIMESTAMP(stamp)) END) 
        - MIN(CASE action WHEN 'add_cart' THEN UNIX_SECONDS(TIMESTAMP(stamp)) END) AS lead_time
    FROM
        row_action_log
    GROUP BY
        user_id, product_id
)
SELECT
    user_id,
    product_id,
    add_cart_time,
    purchase_time,
    lead_time
FROM
    action_time_stats
ORDER BY
    user_id, product_id;
```

![](images/스크린샷%202026-09-28%20113052.png)

### 3-3 등록으로부터의 매출을 날짜별로 집계하기 

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH index_intervals AS (
    -- Hive, Redshift, BigQuery, SparkSQL의 경우 UNION ALL 등으로 대체 가능
    SELECT '30 day sales amount' AS index_name, 0 AS interval_begin_date, 30 AS interval_end_date UNION ALL
    SELECT '45 day sales amount', 0, 45 UNION ALL
    SELECT '60 day sales amount', 0, 60
),
mst_users_with_base_date AS (
    SELECT
        user_id,
        -- 기준일로 등록일 사용하기
        register_date AS base_date
    FROM
        `4th_week.mst_users`
),
purchase_log_with_index_date AS (
    SELECT
        u.user_id,
        u.base_date,
        -- BigQuery의 경우 한 번 타임스탬프 자료형으로 변환하고 날짜 자료형으로 변환하기
        DATE(TIMESTAMP(p.stamp)) AS action_date,
        MAX(DATE(TIMESTAMP(p.stamp))) OVER() AS latest_date,
        SUBSTR(CAST(u.base_date AS STRING), 1, 7) AS month,
        i.index_name,
        
        -- BigQuery의 경우 다음과 같이 사용하기
        -- 지표 대상 기간의 시작일과 종료일 계산하기
        DATE_ADD(CAST(u.base_date AS DATE), INTERVAL i.interval_begin_date DAY) AS index_begin_date,
        DATE_ADD(CAST(u.base_date AS DATE), INTERVAL i.interval_end_date DAY) AS index_end_date,
        
        p.amount
    FROM
        mst_users_with_base_date AS u
    LEFT OUTER JOIN
        `4th_week.action_log` AS p
    ON
        u.user_id = p.user_id
        AND p.action = 'purchase'
    CROSS JOIN
        index_intervals AS i
)
SELECT *
FROM
    purchase_log_with_index_date;
```

![](images/스크린샷%202026-09-28%20113106.png)



### 🎉 수고하셨습니다.