# SQL_MASTER 6주차 정규과제

📌SQL MASTER 정규과제는 매주 정해진 분량의 『*데이터 분석을 위한 SQL 레시피*』 를 읽고 학습하는 것입니다. 이번 주는 아래의 **SQL_MASTER_6th_TIL**에 나열된 분량을 읽고 공부하시면 됩니다.

아래 실습을 수행하며 학습 내용을 직접 적용해보세요. 단순히 결과를 재현하는 것이 아니라, SQL을 직접 작성하는 과정에서 개념을 스스로 정리하는 것이 중요합니다.

필요한 경우 교재와 추가 자료를 참고하여 이해를 보완하시기 바랍니다.

## SQL_MASTER_6th_TIL

### 7장 데이터 활용의 정밀도를 높이는 분석 기술
#### 1. 데이터를 조합해서 새로운 데이터 만들기
#### 2. 이상값 검출하기 
#### 3. 데이터 중복 검출하기
#### 4. 여러 개의 데이터셋 비교하기 


## Study Schedule

| 주차  | 공부 범위     | 완료 여부 |
| ----- | ------------- | --------- |
| 1주차 | p.52~136   | ✅         |
| 2주차 | p.138~184  | ✅         |
| 3주차 | p.186~232 | ✅         |
| 4주차 | p.233~321 | ✅         |
| 5주차 | p.324~406 | ✅         |
| 6주차 | p.408~464 | ✅         |
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

## 1. 데이터를 조합해서 새로운 데이터 만들기

### 1-1 IP 주소를 기반으로 국가와 지역 보완하기 


```sql
SELECT
    a.ip,
    l.continent_name,
    l.country_name,
    l.city_name,
    l.time_zone
FROM public.action_log_with_ip AS a
LEFT JOIN public.mst_city_ip AS i
    ON a.ip::inet <<= i.network
LEFT JOIN public.mst_locations AS l
    ON i.geoname_id = l.geoname_id;
```


### 1-2 주말과 공휴일 판단하기 


```sql
SELECT
    a.action,
    a.stamp,
    c.dow,
    c.holiday_name,
    (
        c.dow_num IN (0, 6)
        OR c.holiday_name IS NOT NULL
    ) AS is_day_off
FROM public.access_log AS a
JOIN public.mst_calendar AS c
    ON CAST(SUBSTRING(a.stamp, 1, 4) AS INTEGER) = c.year
   AND CAST(SUBSTRING(a.stamp, 6, 2) AS INTEGER) = c.month
   AND CAST(SUBSTRING(a.stamp, 9, 2) AS INTEGER) = c.day
ORDER BY a.stamp;

```


### 1-3 하루 집계 범위 변경하기 


```sql
WITH action_log_with_mod_stamp AS (
    SELECT
        *,
        stamp::timestamp - INTERVAL '4 hours' AS mod_stamp
    FROM public.action_log
)
SELECT
    session,
    user_id,
    action,
    stamp,
    stamp::timestamp::date AS raw_date,
    mod_stamp::date AS mod_date
FROM action_log_with_mod_stamp
ORDER BY stamp::timestamp;
```



## 2. 이상값 검출하기 

### 2-1 데이터 분산 계산하기 



```sql
-- 세션별로 페이지 열람 수 랭킹 비율을 구하는 쿼리
WITH session_count AS (
    SELECT
        session,
        COUNT(*) AS access_count
    FROM public.action_log_with_noise
    GROUP BY session
)
SELECT
    session,
    access_count,
    RANK() OVER (ORDER BY access_count DESC) AS access_rank,
    PERCENT_RANK() OVER (ORDER BY access_count DESC) AS percent_rank,
    PERCENT_RANK() OVER (ORDER BY access_count DESC) <= 0.05
        AS is_high_access_candidate
FROM session_count
ORDER BY access_count DESC, session;
```



### 2-2 크롤러 제외하기 



```sql
SELECT *
FROM public.action_log_with_noise
WHERE NOT (
    LOWER(COALESCE(user_agent, '')) LIKE '%bot%'
    OR LOWER(COALESCE(user_agent, '')) LIKE '%crawler%'
    OR LOWER(COALESCE(user_agent, '')) LIKE '%spider%'
    OR LOWER(COALESCE(user_agent, '')) LIKE '%archiver%'
);
```


### 2-3 데이터 타당성 확인하기 



```sql
SELECT
    action,
    AVG(CASE WHEN session IS NOT NULL
             THEN 1.0 ELSE 0.0 END) AS session_valid_ratio,
    AVG(CASE WHEN user_id IS NOT NULL
             THEN 1.0 ELSE 0.0 END) AS user_id_valid_ratio,
    AVG(CASE
        WHEN action = 'view' AND category IS NULL THEN 1.0
        WHEN action <> 'view' AND category IS NOT NULL THEN 1.0
        ELSE 0.0
    END) AS category_valid_ratio,
    AVG(CASE
        WHEN action = 'view' AND products IS NULL THEN 1.0
        WHEN action <> 'view' AND products IS NOT NULL THEN 1.0
        ELSE 0.0
    END) AS products_valid_ratio,
    AVG(CASE
        WHEN action = 'purchase' AND amount IS NOT NULL THEN 1.0
        WHEN action <> 'purchase' AND amount IS NULL THEN 1.0
        ELSE 0.0
    END) AS amount_valid_ratio,
    AVG(CASE WHEN stamp IS NOT NULL
             THEN 1.0 ELSE 0.0 END) AS stamp_valid_ratio
FROM public.invalid_action_log
GROUP BY action
ORDER BY action;
```

<!-- 이 부분을 지우고 실행 결과 화면을 제출해주세요. -->

### 2-4 특정 IP 주소에서의 접근 제외하기 

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH mst_reserved_ip (network, description) AS (
    VALUES
        ('127.0.0.0/8'::inet, 'Loopback'),
        ('10.0.0.0/8'::inet, 'Private network'),
        ('172.16.0.0/12'::inet, 'Private network'),
        ('192.0.0.0/24'::inet, '교재 예시 제외 대역'),
        ('192.168.0.0/16'::inet, 'Private network')
)
SELECT
    l.user_id,
    l.ip,
    l.stamp,
    m.network,
    m.description
FROM public.action_log_with_ip AS l
LEFT JOIN mst_reserved_ip AS m
    ON l.ip::inet <<= m.network
WHERE m.network IS NULL
ORDER BY l.stamp, l.user_id;
```



## 3. 데이터 중복 검출하기

### 3-1 마스터 데이터의 중복 검출하기 

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
-- 전체 건수와 키 개수 확인하기
SELECT
    COUNT(*) AS total_num,
    COUNT(id) AS non_null_key_num,
    COUNT(DISTINCT id) AS unique_key_num,
    COUNT(*) - COUNT(id) AS null_key_num
FROM public.mst_categories;
```



### 3-2 로그 중복 검출하기 



```sql
-- 중복 후보 확인하기
SELECT
    user_id,
    action,
    products,
    COUNT(*) AS record_num,
    STRING_AGG(session, ', ' ORDER BY stamp::timestamp) AS session_list,
    STRING_AGG(stamp::text, ', ' ORDER BY stamp::timestamp) AS stamp_list
FROM public.dup_action_log
GROUP BY user_id, action, products
HAVING COUNT(*) > 1
ORDER BY user_id, action, products;
```



## 4. 여러 개의 데이터셋 비교하기 

### 4-1 데이터의 차이 추출하기 

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
SELECT new_mst.*
FROM public.mst_products_20170101 AS new_mst
LEFT JOIN public.mst_products_20161201 AS old_mst
    ON new_mst.product_id = old_mst.product_id
WHERE old_mst.product_id IS NULL
ORDER BY new_mst.product_id;
```


### 4-2 두 순위의 유사도 계산하기 


```sql
WITH path_stat AS (
    SELECT
        path,
        COUNT(DISTINCT long_session) AS access_users,
        COUNT(DISTINCT short_session) AS access_count,
        COUNT(*) AS page_view
    FROM public.access_log
    GROUP BY path
), metric_values AS (
    SELECT path, 'access_user' AS type, access_users AS metric_value
    FROM path_stat
    UNION ALL
    SELECT path, 'access_count', access_count
    FROM path_stat
    UNION ALL
    SELECT path, 'page_view', page_view
    FROM path_stat
), path_ranking AS (
    SELECT
        path,
        type,
        RANK() OVER (
            PARTITION BY type
            ORDER BY metric_value DESC
        ) + (
            COUNT(*) OVER (PARTITION BY type, metric_value) - 1
        ) / 2.0 AS rank
    FROM metric_values
), pair_ranking AS (
    SELECT
        r1.path,
        r1.type AS type1,
        r1.rank AS rank1,
        r2.type AS type2,
        r2.rank AS rank2,
        POWER(r1.rank - r2.rank, 2) AS diff
    FROM path_ranking AS r1
    JOIN path_ranking AS r2
        ON r1.path = r2.path
)
SELECT
    type1,
    type2,
    COUNT(*) AS path_count,
    CORR(
        rank1::double precision,
        rank2::double precision
    ) AS spearman
FROM pair_ranking
GROUP BY type1, type2
ORDER BY type1, spearman DESC NULLS LAST, type2;
```


---

### 🎉 수고하셨습니다.