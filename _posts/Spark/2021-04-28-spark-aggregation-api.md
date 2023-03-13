---
title: Chapter 7 - 집계 연산
categories: [Spark]
tags: [Spark, bigdata]
comments : true
---

## Chapter 7 - 집계 연산

### 집계 함수

|     함수                     |     내용                                    |
|------------------------------|---------------------------------------------|
|     count                    |     로우 수 카운트                          |
|     countDistinct            |     고유 레코드 수                          |
|     approx_count_distinct    |     근사치 고유 레코드 수 반환(오차있음)    |
|     first                    |     첫번째 값 반환                          |
|     last                     |     마지막 값 반환                          |
|     min                      |     최소값                                  |
|     max                      |     최대값                                  |
|     sum                      |     칼럼의 레코드 합산                      |
|     sumDistinct              |     고윳값 합산                             |
|     avg, mean                |     평균                                    |
|     var_pop, stddev_pop      |     분산, 표준편차                          |
|     skewness                 |     비대칭도                                |
|     kurtosis                 |     첨도                                    |
|     corr                     |     피어슨 상관계수                         |
|     covar_pop                |     표본 공분산                             |
|     covar_samp               |     모공분산                                |
|     collect_set              |     칼럼 내 고유 데이터 모음                |
|     collect_list             |     칼럼 내 데이터 리스트 모음              |
|     agg                      |     복합 데이터 집계                        |

### 그룹화
```python
df.groupBy("InvoiceNo", "CustomerId").count().show()
```

> agg를 통해 여러 집계 처리를 한 번에 지정할 수 있다.
> ```python
> from pyspark.sql.functions import count, expr
>
> df.groupBy("InvoiceNo").agg(
>    count("Quantity").alias("quan"),
>    expr("count(Quantity)")
> ).show()
> ```

> 칼럼을 키로, 수행할 집계 함수의 문자열을 값으로 하는 맵 타입을 사용해 정의할 수 있다.
> ```python
> from pyspark.sql.functions import expr
> df.groupBy("InvoiceNo").agg(expr("avg(Quantity)"),expr("stddev_pop(Quantity)"))\
>   .show()
> ```

### 윈도우 함수
> 윈도우 함수를 집계에 사용할 수 있다.
> ```python
> from pyspark.sql.window import Window
> from pyspark.sql.functions import desc
>
> windowSpec = Window\
>     .partitionBy("CustomerId", "date")\
>     .orderBy(desc("Quantity"))\
>     .rowsBetween(Window.unboundedPreceding, Window.currentRow)
>
> ## 윈도우 별 최대 구매 개수
> from pyspark.sql.functions import max, col
>
> maxPurchaseQuantity = max(col("Quantity")).over(windowSpec)
>
> ## 모든 고객에 대한 최대 구매 수량을 가진 날짜 산출
> from pyspark.sql.functions import dense_rank, rank
>
> purchaseDenseRank = dense_rank().over(windowSpec)
> purchaseRank = rank().over(windowSpec)
>
> ## 윈도우 값 확인
> dfWithDate.where("CustomerId IS NOT NULL").orderBy("CustomerId")\
>   .select(
>     col("CustomerId"),
>     col("date"),
>     col("Quantity"),
>     purchaseRank.alias("quantityRank"),
>     purchaseDenseRank.alias("quantityDenseRank"),
>     maxPurchaseQuantity.alias("maxPurchaseQuantity")
>   ).show()
> ```

### 롤업
> 롤업은 group by 스타일의 다양한 연산을 수행할 수 있는 다차원 집계 기능이다.
>
> ```python
> rolledUpDf = dfNoNull.rollup("Date", "Country").agg(sum("Quantiyu"))\
>     .selectExpr("Date", "Country", "`sum(Quantity)` as total_quantity")\
>     .orderBy("Date")
> rolledUpDf.show()
>
> rolledUpDf.where("Country IS NULL").show()
> rolledUpDf.where("Date IS NULL").show()
> ```

> 큐브는 롤업을 고차원적으로 사용할 수 있게 한다.<br>
> 큐브는 요소들을 계층적으로 다루는 대신 모든 차원에 대해 동일한 작업을 수행한다.<br>
> 즉, 전체 기간에 대해 날짜와 국가별 결과를 얻을 수 있다.<br>
> ```python
> from pyspark.sql.functions import sum
>
> dfNoNull.cube("Date", "Country").agg(sum(col("Quantity")))\
>   .select("Date", "Country", "sum(Quantity)").orderBy("Date").show()
> ```

> 큐브와 롤업을 사용하다 보면 집계 수준에 따라 쉽게 필터링하기 위해 집계수준을 조회하는 경우가 필요하다.
> 이때, grouping_id를 사용한다.

> 피벗을 사용해 로우를 칼럼으로 변환할 수 있다.
> ```python
> pivoted = dfWithDate.groupBy("date").pivot("Country").sum()
>
> pivoted.where("date > '2011-12-05'").select("date", "`USA_sum(Quantity)`").show()
> ```

<br>
<br>

##### Reference
[스파크 완벽 가이드](https://www.hanbit.co.kr/store/books/look.php?p_code=B6709029941)
