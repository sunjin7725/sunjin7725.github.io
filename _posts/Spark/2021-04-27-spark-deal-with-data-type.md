---
title: Chapter 6 - 다양한 데이터 타입 다루기
categories: [Spark]
tags: [Spark, bigdata]
comments : true
---

## Chapter 6 - 다양한 데이터 타입 다루기

### 불리언 타입 다루기
```python
from pyspark.sql.functions import col

## equal
df.where("InvoiceNo = 536365")\
  .select("InvoiceNo", "Description")\
  .show(5, False)

## not equal
df.where("InvoiceNo != 536365")\
  .show(5, False)
df.where("InvoiceNo <> 536365")\
  .show(5, False)

## and/or
from pyspark.sql.functions import instr

priceFilter = col("UnitPrice") > 600
describefilter = instr(df.Description, "POSTAGE") >= 1
df.where(df.StockCode.isin("DOT")).where(priceFilter | describefilter).show()
```

### 수치형 데이터 타입 다루기

|     통계 함수                   |     내용                                               |
|---------------------------------|--------------------------------------------------------|
|     pow                         |     거듭제곱                                           |
|     round                       |     반올림                                             |
|     bround                      |     내림                                               |
|     corr                        |     피어슨 상관계수                                    |
|     count                       |     count                                              |
|     mean                        |     평균                                               |
|     stddev_pop                  |     표준편차                                           |
|     min                         |     최솟값                                             |
|     max                         |     최대값                                             |
|     describe                    |     기초 통계 요약(count, mean,   stddev, min, max)    |
|     approxQuantile              |     백분위수                                           |
|     crosstab                    |     교차표                                             |
|     freqItems                   |     항목빈도수                                         |
|     monotonically_increasing_id |     고윳값 생성                                        |

### 문자열 다루기

|     함수              |     내용                          |
|-----------------------|-----------------------------------|
|     initcap           |     단어 맨 앞 대문자 치환        |
|     lower             |     소문자 치환                   |
|     upper             |     대문자 치환                   |
|     ltrim             |     왼쪽 공백 제거                |
|     rtrim             |     오른쪽 공백 제거              |
|     trim              |     양쪽 공백 제거                |
|     lpad              |     왼쪽 공백 추가                |
|     rpad              |     오른쪽 공백 추가              |
|     regexp_extract    |     정규 표현식에 따른 추출       |
|     regexp_replace    |     정규 표현식에 따른 치환       |
|     translate         |     색인 문자열에 따른 치환       |
|     instr             |     문자열 내 단어의 존재 위치    |
|     locate            |     문자열 내 단어의 존재 위치    |

### 날짜 다루기

|     함수                 |     내용                           |
|--------------------------|------------------------------------|
|     current_date         |     현재 날짜                      |
|     current_timestamp    |     현재 시간까지                  |
|     date_add             |     날짜 더하기                    |
|     date_sub             |     날짜 빼기                      |
|     datediff             |     날짜 차이                      |
|     months_between       |     월 차이                        |
|     to_date              |     문자열을 날짜로 변환           |
|     to_timestamp         |     문자열을 timestamp로   변환    |

### null 값 다루기

|     함수                      |     내용                                                                           |
|-------------------------------|------------------------------------------------------------------------------------|
|     coalesce                  |     Null 이면 두번째 인자 반환                                                     |
|     dataframe.na.drop()       |     Null 값이 있는 row 제거(“any”)     모든 record가 null일 때, row 제거(“all”)    |
|     dataframe.na.fill()       |     널 값 바꾸기                                                                   |
|     dataframe.na.replace()    |     특정 문자열의 null 값   치환                                                   |

### 복합 데이터 타입
#### 구조체
> 구조체는 DataFrame 내부의 DataFrame으로 생각할 수 있다. 다수의 칼럼을 구조체로 묶어서 활용할 수 있다.
>
> Python Code :
> ```python
> from pyspark.sql.functions import struct, col
>
> complexDf = df.select(struct("Description", "InvoiceNo").alias("complex"))
> complexDf.createOrReplaceTempView("complexDf")
>
> complexDf.select("complex.Description")
>
> complexDf.select(col("complex").getField("Description"))
>
> complexDf.select("complex.*")
> ```

#### 배열
> split 함수를 통해 문자열을 배열로 바꿀 수 있다.
>
> Python Code :
> ```python
> from pyspark.sql.functions import split, col
>
> df.select(split(col("Description"), " ")).show(2)
> ```
> SQL Code:
> ```sql
> SELECT split(Description, ' ') FROM dfTable
> ```

> 배열 내의 원소는 따로 조회가 가능하다.
> Python Code :
> ```python
> from pyspark.sql.functions import split, col
>
> df.select(split(col("description"), " ").alias("array_col"))\
>           .selectExpr("array_col[0]").show(2)
> ```
> SQL Code :
> ```sql
> SELECT split(Description, ' ')[0] FROM dfTable
> ```

> 배열의 길이 조회
> ```python
> from pyspark.sql.functions import split, size, col
> df.select(size(split(col("Description"), " "))).show(2)
> ```

> array_contains
> array_contains를 사용해 배열에 특정 값의 존재 여부를 확인할 수 있다.
> Python Code :
> ```python
> from pyspark.sql.functions import array_contains, split, col
>
> df.select(array_contains(split(col("Description"), " "), "WHITE")).show(2)
> ```
> SQL Code :
> ```sql
> SELECT array_contains(split(Description, ' '), 'WHITE') FROM dfTable
> ```

> explode 함수를 통해 모든 배열 값을 로우로 변환한다. 나머지 칼럼은 중복!
> Python Code :
> ```python
> from pyspark.sql.functions import split, explode, col
>
> df.withColumn("splitted", split(col("Description")," "))\
>   .withColumn("exploded", explode(col("splitted")))\
>   .select("Description", "InvoiceNo", "exploded").show(2)
> ```
> SQL Code :
> ```sql
> SELECT Description, InvoiceNo, exploded
>   FROM (SELECT split(Description, " ") as splitted FROM dfTable)
> LATERAL VIEW explode(splitted) as exploded
> ```

#### 맵
> map 함수를 통해 컬럼을 키-값의 형태로 생성할 수 있다.
>
> Python Code:
> ```python
> from pyspark.sql.functions import create_map, col
> df.select(create_map(col("Description"), col(InvoiceId)).alias("complex_map")).show(2)
> ```
> SQL Code :
> ```sql
> SELECT map(Description, InvoiceNo) as complex_map FROM dfTable
> WHERE Description IS NOT NULL
> ```

> 이렇게 만들어진 map을 적합한 키를 통해 조회 할 수 있다.
> ```python
> from pyspark.sql.functions import create_map, col
> df.select(create_map(col("Description"), col(InvoiceId)).alias("complex_map"))\
>   .selectExpr("complex_map['WHITE METAL LANTERN']").show(2)
> ```

> map 타입은 분해하여 컬럼으로 다시 변환 할 수 있다.
> ```python
> from pyspark.sql.functions import create_map, col
> df.select(create_map(col("Description"), col(InvoiceId)).alias("complex_map"))\
>   .selectExpr("explode(complex_map)").show(2)
> ```

#### JSON
> 스파크는 json 데이터를 다루는 기능을 제공한다.
> ```python
> jsonDf = spark.range(1).selectExpr("""
>   '{
>       "myJSONKey": {
>           "myJSONValue": [1, 2, 3]
>       }
>     }' as jsonString
> """)
> ```

> json 객체를 인라인 쿼리로 조회 할 수도 있다. 중첩이 없는 단일 수준이라면 json_tuple 만으로도 가능하다.
> ```python
> from pyspark.sql.functions import get_json_object, json_tuple, col
>
> jsonDf.select(
>   get_json_object(col("jsonString"), "$.myJSONKey.myJSONValue[1]").alias("column"),
>   json_tuple(col("jsonString"), 'myJSONKey')
> ).show(2)
> ```

> to_json 함수를 사용하여 StructType을 JSON 문자열로 변경할 수 있다.
> ```python
> from pyspark.sql.functions import to_json, col
>
> df.selectExpr("(InvoiceNo, Description) as myStruct")\
>   .select(to_json(col("myStruct")))
> ```

#### 사용자 정의 함수
> 스파크는 각 프로그래밍 언어에 따라 사용자 정의 함수를 만들 수 있다.
> 하지만 각 프로그래밍 언어에 따라 성능의 차이는 있을 수 있다.
>
> 예제)
> ```python
> udfExmapleDf = spark.range(5).toDF("num")
>
> def power3(double_value):
>     return double_value ** 3
>
> from pyspark.sql.functions import udf, col
>
> power3udf = udf(power3)
> udfExmapleDf.select(power3udf(col("num"))).show(2)
> ```

> 하지만, 해당 udf는 문자열 표현식에서는 사용할 수 없다.
> 사용자 정의 함수를 스파크 SQL함수로 등록하면 모든 함수에서 사용할 수 있다.
> ```python
> from pyspark.sql.types import IntegerType, DoubleType
> spark.udf.register("power3py", power3, DoubleType())
> udfExampleDf.selectExpr("power3py(num)").show(2)
> ```

<br>
<br>

##### Reference
[스파크 완벽 가이드](https://www.hanbit.co.kr/store/books/look.php?p_code=B6709029941)
