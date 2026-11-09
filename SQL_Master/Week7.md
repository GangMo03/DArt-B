# SQL_MASTER 7주차 정규과제

📌SQL MASTER 정규과제는 매주 정해진 분량의 『*데이터 분석을 위한 SQL 레시피*』 를 읽고 학습하는 것입니다. 이번 주는 아래의 **SQL_MASTER_7th_TIL**에 나열된 분량을 읽고 공부하시면 됩니다.

아래 실습을 수행하며 학습 내용을 직접 적용해보세요. 단순히 결과를 재현하는 것이 아니라, SQL을 직접 작성하는 과정에서 개념을 스스로 정리하는 것이 중요합니다.

필요한 경우 교재와 추가 자료를 참고하여 이해를 보완하시기 바랍니다.

## SQL_MASTER_7th_TIL

### 8장 데이터를 무기로 삼기 위한 분석 기술
#### 1. 검색 기능 평가하기
#### 2. 데이터마이닝
#### 3. 추천 (범위 제외)
#### 4. 점수 계산하기 (범위 제외)


## Study Schedule

| 주차  | 공부 범위     | 완료 여부 |
| ----- | ------------- | --------- |
| 1주차 | p.52~136   | ✅         |
| 2주차 | p.138~184  | ✅         |
| 3주차 | p.186~232 | ✅         |
| 4주차 | p.233~321 | ✅         |
| 5주차 | p.324~406 | ✅         |
| 6주차 | p.408~464 | ✅         |
| 7주차 | p.466~566 | ✅         |

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

## 1. 검색 기능 평가하기

### 1-1 No Match 비율과 키워드 집계하기 


```sql
SELECT
    stamp::timestamp::date AS dt,
    COUNT(*) AS search_count,
    SUM(CASE WHEN result_num = 0 THEN 1 ELSE 0 END) AS no_match_count,
    AVG(CASE WHEN result_num = 0 THEN 1.0 ELSE 0.0 END) AS no_match_rate
FROM public.access_log
WHERE action = 'search'
GROUP BY stamp::timestamp::date
ORDER BY dt;
```

![](images/스크린샷%202026-11-09%20115154.png)

### 1-2 재검색 비율과 키워드 집계하기 


```sql
WITH access_log_with_next_search AS (
    SELECT
        *,
        LEAD(action) OVER w AS next_action,
        LEAD(keyword) OVER w AS next_keyword,
        LEAD(result_num) OVER w AS next_result_num
    FROM public.access_log
    WINDOW w AS (
        PARTITION BY session
        ORDER BY stamp::timestamp
    )
)
SELECT
    keyword,
    result_num,
    next_keyword,
    next_result_num,
    COUNT(*) AS retry_count
FROM access_log_with_next_search
WHERE action = 'search'
  AND next_action = 'search'
GROUP BY keyword, result_num, next_keyword, next_result_num
ORDER BY retry_count DESC, keyword, next_keyword;
```

![](images/스크린샷%202026-11-09%20115239.png)

### 1-3 재검색 키워드를 분류해서 집계하기 


```sql
WITH access_log_with_next_search AS (
    SELECT
        *,
        LEAD(action) OVER w AS next_action,
        LEAD(keyword) OVER w AS next_keyword,
        LEAD(result_num) OVER w AS next_result_num
    FROM public.access_log
    WINDOW w AS (
        PARTITION BY session
        ORDER BY stamp::timestamp
    )
), retry_log AS (
    SELECT
        *,
        NULLIF(LOWER(BTRIM(keyword)), '') AS normalized_keyword,
        NULLIF(LOWER(BTRIM(next_keyword)), '') AS normalized_next_keyword
    FROM access_log_with_next_search
    WHERE action = 'search'
      AND next_action = 'search'
), classified_retry AS (
    SELECT
        *,
        CASE
            WHEN result_num = 0 THEN 'NoMatch 조건 변경'
            WHEN normalized_keyword IS NULL
              OR normalized_next_keyword IS NULL THEN '키워드 결손'
            WHEN normalized_next_keyword = normalized_keyword
                THEN '동일 키워드 반복'
            WHEN POSITION(normalized_keyword IN normalized_next_keyword) > 0
                THEN '검색어 구체화 후보'
            ELSE '검색 키워드 변경'
        END AS retry_type
    FROM retry_log
)
SELECT
    retry_type,
    keyword,
    result_num,
    next_keyword,
    next_result_num,
    COUNT(*) AS retry_count
FROM classified_retry
GROUP BY retry_type, keyword, result_num, next_keyword, next_result_num
ORDER BY retry_type, retry_count DESC, keyword, next_keyword;
```

![](images/스크린샷%202026-11-09%20115311.png)

### 1-4 검색 이탈 비율과 키워드 집계하기

```sql
WITH access_log_with_next_search AS (
    SELECT
        *,
        LEAD(action) OVER w AS next_action,
        LEAD(keyword) OVER w AS next_keyword,
        LEAD(result_num) OVER w AS next_result_num
    FROM public.access_log
    WINDOW w AS (
        PARTITION BY session
        ORDER BY stamp::timestamp
    )
)
SELECT
    keyword,
    result_num,
    COUNT(*) AS search_count,
    SUM(CASE WHEN next_action IS NULL THEN 1 ELSE 0 END) AS exit_count,
    AVG(CASE WHEN next_action IS NULL THEN 1.0 ELSE 0.0 END) AS exit_rate
FROM access_log_with_next_search
WHERE action = 'search'
GROUP BY keyword, result_num
HAVING SUM(CASE WHEN next_action IS NULL THEN 1 ELSE 0 END) > 0
ORDER BY exit_count DESC, keyword, result_num;
```

![](images/스크린샷%202026-11-09%20115336.png)

### 1-5 검색 키워드 관련 지표의 집계 효율화 하기

```sql
WITH access_log_with_next_search AS (
    SELECT
        *,
        LEAD(action) OVER w AS next_action,
        LEAD(keyword) OVER w AS next_keyword,
        LEAD(result_num) OVER w AS next_result_num
    FROM public.access_log
    WINDOW w AS (
        PARTITION BY session
        ORDER BY stamp::timestamp
    )
), search_log_with_flags AS (
    SELECT
        stamp::timestamp::date AS dt,
        session,
        keyword,
        result_num,
        next_action,
        next_keyword,
        next_result_num,
        CASE WHEN result_num = 0 THEN 1.0 ELSE 0.0 END AS is_no_match,
        CASE WHEN next_action = 'search' THEN 1.0 ELSE 0.0 END AS is_retry,
        CASE WHEN next_action IS NULL THEN 1.0 ELSE 0.0 END AS is_exit
    FROM access_log_with_next_search
    WHERE action = 'search'
)
SELECT
    dt,
    COUNT(*) AS search_count,
    SUM(is_no_match) AS no_match_count,
    AVG(is_no_match) AS no_match_rate,
    SUM(is_retry) AS retry_count,
    AVG(is_retry) AS retry_rate,
    SUM(is_exit) AS exit_count,
    AVG(is_exit) AS exit_rate
FROM search_log_with_flags
GROUP BY dt
ORDER BY dt;
```

![](images/스크린샷%202026-11-09%20115448.png)

### 1-6 검색 결과의 포괄성을 지표화 하기

```sql
WITH search_result_with_correct_items AS (
    SELECT
        COALESCE(r.keyword, c.keyword) AS keyword,
        r.rank,
        COALESCE(r.item, c.item) AS item,
        CASE WHEN c.item IS NOT NULL THEN 1 ELSE 0 END AS correct
    FROM public.search_result AS r
    FULL OUTER JOIN public.correct_result AS c
        ON r.keyword = c.keyword
       AND r.item = c.item
), cumulative_result AS (
    SELECT
        *,
        SUM(correct) OVER (
            PARTITION BY keyword
            ORDER BY rank ASC NULLS LAST, item
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS cum_correct,
        SUM(correct) OVER (PARTITION BY keyword) AS total_correct,
        COUNT(rank) OVER (
            PARTITION BY keyword
            ORDER BY rank ASC NULLS LAST, item
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS retrieved_count
    FROM search_result_with_correct_items
), recall_at_5_by_keyword AS (
    SELECT
        keyword,
        SUM(CASE WHEN rank <= 5 THEN correct ELSE 0 END) AS hit_count_at_5,
        MAX(total_correct) AS total_correct,
        100.0 * SUM(CASE WHEN rank <= 5 THEN correct ELSE 0 END)
            / NULLIF(MAX(total_correct), 0) AS recall_at_5
    FROM cumulative_result
    GROUP BY keyword
)
SELECT
    *,
    AVG(recall_at_5) OVER () AS average_recall_at_5
FROM recall_at_5_by_keyword
ORDER BY keyword;
```

![](images/스크린샷%202026-11-09%20115516.png)

### 1-7 검색 결과의 타당성을 지표화 하기

```sql
WITH search_result_with_correct_items AS (
    SELECT
        COALESCE(r.keyword, c.keyword) AS keyword,
        r.rank,
        COALESCE(r.item, c.item) AS item,
        CASE WHEN c.item IS NOT NULL THEN 1 ELSE 0 END AS correct
    FROM public.search_result AS r
    FULL OUTER JOIN public.correct_result AS c
        ON r.keyword = c.keyword
       AND r.item = c.item
), cumulative_result AS (
    SELECT
        *,
        SUM(correct) OVER (
            PARTITION BY keyword
            ORDER BY rank ASC NULLS LAST, item
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS cum_correct,
        SUM(correct) OVER (PARTITION BY keyword) AS total_correct,
        COUNT(rank) OVER (
            PARTITION BY keyword
            ORDER BY rank ASC NULLS LAST, item
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS retrieved_count
    FROM search_result_with_correct_items
), precision_at_5_by_keyword AS (
    SELECT
        keyword,
        SUM(CASE WHEN rank <= 5 THEN correct ELSE 0 END) AS hit_count_at_5,
        COUNT(CASE WHEN rank <= 5 THEN 1 END) AS result_count_at_5,
        COALESCE(
            100.0 * SUM(CASE WHEN rank <= 5 THEN correct ELSE 0 END)
                / NULLIF(COUNT(CASE WHEN rank <= 5 THEN 1 END), 0),
            0.0
        ) AS precision_at_5
    FROM cumulative_result
    GROUP BY keyword
)
SELECT
    *,
    AVG(precision_at_5) OVER () AS average_precision_at_5
FROM precision_at_5_by_keyword
ORDER BY keyword;
```

![](images/스크린샷%202026-11-09%20115551.png)

### 1-8 검색 결과 순위와 관련된 지표 계산하기


```sql
WITH search_result_with_correct_items AS (
    SELECT
        COALESCE(r.keyword, c.keyword) AS keyword,
        r.rank,
        COALESCE(r.item, c.item) AS item,
        CASE WHEN c.item IS NOT NULL THEN 1 ELSE 0 END AS correct
    FROM public.search_result AS r
    FULL OUTER JOIN public.correct_result AS c
        ON r.keyword = c.keyword
       AND r.item = c.item
), cumulative_result AS (
    SELECT
        *,
        SUM(correct) OVER (
            PARTITION BY keyword
            ORDER BY rank ASC NULLS LAST, item
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS cum_correct,
        SUM(correct) OVER (PARTITION BY keyword) AS total_correct,
        COUNT(rank) OVER (
            PARTITION BY keyword
            ORDER BY rank ASC NULLS LAST, item
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS retrieved_count
    FROM search_result_with_correct_items
), precision_by_result AS (
    SELECT
        *,
        CASE
            WHEN rank IS NULL THEN 0.0
            ELSE 100.0 * cum_correct / NULLIF(retrieved_count, 0)
        END AS precision_at_rank
    FROM cumulative_result
), average_precision_by_keyword AS (
    SELECT
        keyword,
        SUM(CASE WHEN correct = 1 THEN precision_at_rank ELSE 0.0 END)
            / NULLIF(MAX(total_correct), 0) AS average_precision
    FROM precision_by_result
    GROUP BY keyword
)
SELECT
    keyword,
    average_precision,
    AVG(average_precision) OVER () AS mean_average_precision
FROM average_precision_by_keyword
ORDER BY keyword;
```

![](images/스크린샷%202026-11-09%20115617.png)


## 2. 데이터마이닝 

### 2-1 어소시에이션 분석 

```sql
WITH purchase_products AS (
    SELECT DISTINCT purchase_id, product_id
    FROM public.purchase_detail_log
    WHERE purchase_id IS NOT NULL
      AND product_id IS NOT NULL
), purchase_id_count AS (
    SELECT COUNT(DISTINCT purchase_id) AS purchase_count
    FROM purchase_products
), purchase_detail_log_with_counts AS (
    SELECT
        d.purchase_id,
        d.product_id,
        p.purchase_count,
        COUNT(*) OVER (PARTITION BY d.product_id) AS product_count
    FROM purchase_products AS d
    CROSS JOIN purchase_id_count AS p
), product_pair_with_stat AS (
    SELECT
        l1.product_id AS p1,
        l2.product_id AS p2,
        l1.product_count AS p1_count,
        l2.product_count AS p2_count,
        COUNT(*) AS p1_p2_count,
        l1.purchase_count
    FROM purchase_detail_log_with_counts AS l1
    JOIN purchase_detail_log_with_counts AS l2
        ON l1.purchase_id = l2.purchase_id
       AND l1.product_id <> l2.product_id
    GROUP BY
        l1.product_id, l2.product_id,
        l1.product_count, l2.product_count, l1.purchase_count
)
SELECT
    p1,
    p2,
    purchase_count,
    p1_count,
    p2_count,
    p1_p2_count,
    100.0 * p1_p2_count / NULLIF(purchase_count, 0) AS support,
    100.0 * p1_p2_count / NULLIF(p1_count, 0) AS confidence,
    (1.0 * p1_p2_count / NULLIF(p1_count, 0))
        / NULLIF(1.0 * p2_count / NULLIF(purchase_count, 0), 0) AS lift
FROM product_pair_with_stat
ORDER BY p1, p2;
```

![](images/스크린샷%202026-11-09%20115650.png)


한 학기 동안 다소 부족한 커리큘럼이었음에도 불구하고 성실하게, 그리고 열심히 과제를 수행해주셔서 진심으로 감사드립니다.

이번 과정을 통해 배운 내용이 앞으로 여러분이 SQL을 다루는 데에 조금이나마 도움이 되기를 바랍니다.

여러분의 앞날을 진심으로 응원합니다. 😊


### 🎉 수고하셨습니다.