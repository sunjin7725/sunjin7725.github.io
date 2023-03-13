---
title: Chapter 5 - 구조적 API 기본 연산
categories: [Spark]
tags: [Spark, bigdata]
comments : true
---

## Chapter 5 - 구조적 API 기본 연산

### Spark DataType - Python DataType

|     스파크 데이터 타입    |     파이썬 데이터 타입                |     데이터 타입 생성/접근용 API    |
|:---------------------------|:---------------------------------------|:------------------------------------|
|     ByteType              |     Int, long   / (-2^8 ~ 2^8-1)      |     ByteType()                     |
|     ShortType             |     Int, long   / (-2^16 ~ 2^16-1)    |     ShortType()                    |
|     IntegerType           |     Int, long                         |     IntegerType()                  |
|     LongType              |     Long /   (-2^63 ~ 2^64-1)         |     LongType()                     |
|     FloatType             |     float                             |     FloatType()                    |
|     DoubleType            |     float                             |     DoubleType()                   |
|     DecimalType           |     Decimal                           |     DecimalType()                  |
|     StringType            |     string                            |     StringType()                   |
|     BinaryType            |     bytearray                         |     BinaryType()                   |
|     BooleanType           |     bool                              |     BooleanType()                  |
|     TimestampType         |     datetime                          |     TimestampType()                |
|     DateType              |     Datetime,   date                  |     DateType()                     |
|     ArrayType             |     List,   tuple, array              |     ArrayType()                    |
|     MapType               |     dict                              |     MapType()                      |
|     StructType            |     List, tuple                       |     StructType()                   |
|     StructField           |                                       |     StructField()                  |

### 표현식
여러 칼럼을 입력 받아 식별하여 각 레코드에 적용하는 함수로,
expr("someCol"), col("somCol")로 표현한다.

```py
from pyspark.sql.functions import expr

expr("((((someCol) + 5) * 200) - 6) < otherCol")
```

### DataFrame 컬럼에 접근하기
```py
from pyspark.sql import SparkSession
spark = SparkSession.builder(master='local') ## python spark session 생성

spark.read.format("dataFormat").load("$dataPath").columns
```

### SQL 을 위한 임시뷰로 등록하기
```py
df = spark.read.format("dataFormat").load("$dataPath")
df.createOrReplaceTempView("dfTable") ## 임시 뷰 생성
```

### select와 selectExpr
> Python Code :
> ```py
> df.select("DEST_COUTRY_NAME", "ORIGIN_COUNTRY_NAME").show(2)
> ```
> SQL Code :
> ```sql
> SELECT  DEST_COUNTRY_NAME
>        ,ORIGIN_COUNTRY_NAME
>   FROM  dfTable
>  LIMIT  2
> ```

> Python Code :
> ```py
> df.select(expr("DEST_COUTRY_NAME AS destination")).show(2)
> df.select(("DEST_COUNTRY_NAME").alias(destination)).show(2)
> df.selectExpr("DEST_COUTRY_NAME", "destination").show(2)
> ```
> SQL Code :
> ```sql
> SELECT DEST_COUNTRY_NAME AS destination
>   FROM dfTable
>  LIMIT 2;
> ```

> Python Code :
> ```py
> df.selectExpr(
>    "*",
>    "(DEST_COUNTRY_NAME = ORIGIN_COUNTRY_NAME) as withinCountry"
> ).show(2)
> ```
> SQL Code :
> ```sql
> SELECT *
>       ,(DEST_COUNTRY_NAME = ORIGIN_COUNTRY_NAME) as withinCountry
>  FROM  dfTable
> LIMIT  2
> ```

### 컬럼 추가하기
> Python Code:
> ```py
> from pyspark.sql.functions import lit
> df.withColumn("numberOne", lit(1)).show(2)
> ```
> SQL Code:
> ```sql
> SELECT *, 1 as numberOne
>   FROM dfTable
>  LIMIT 2
> ```

> Python Code:
> ```py
> df.withColumn("withinCountry", expr("DEST_COUNTRY_NAME = ORIGIN_COUNTRY_NAME")).show(2)
> ```
> ```sql
> SELECT *
>       ,(DEST_COUNTRY_NAME = ORIGIN_COUNTRY_NAME) as withinCountry
>  FROM  dfTable
> LIMIT  2
> ```

### 컬럼명 변경하기
```py
df.withColumnRenamed("DEST_COUNTRY_NAME", "dest")
```

### 컬럼 제거하기
```py
df.drop("ORIGIN_COUNTRY_NAME")
df.drop("ORIGIN_COUNTRY_NAME", "DEST_COUNTRY_NAME")
```

### 컬럼 데이터 타입 변경하기
```py
from pyspark.sql.functions import col
df.withColumn("count2", col("count").cast("string"))
```

### 레코드 필터링
> Python Code :
> ```py
> from pyspark.sql.functions import col
> df.filter(col("count") < 2).show(2)
> df.where("count < 2").show(2)
> ```
> SQL Code :
> ```sql
> SELECT *
>   FROM dfTable
>  WHERE count < 2
>  LIMIT 2
> ```

> Python Code :
> ```py
> from pyspark.sql.functions import col
> df.where("count < 2").where(col("ORIGIN_COUNTRY_NAME =!= Croatia").show(2)
> ```
> SQL Code:
> ```sql
> SELECT *
>   FROM dfTable
>  WHERE count < 2
>    AND ORIGIN_COUNTRY_NAME <> Croatia
>  LIMIT 2
> ```

### Distinct
> Python Code :
> ```py
> df.select("ORIGIN_COUNTRY_NAME", "DEST_COUNTRY_NAME").distinct().count()
> ```
> SQL Code :
> ```sql
> SELECT COUNT(DISTINCT(ORIGIN_COUNTRY_NAME, DEST_COUNTRY_NAME))
>   FROM dfTable
> ```

### 무작위 샘플 만들기
```py
seed = 5
withReplacement = False
fraction = 0.5

df.sample(withReplacement, fraction, seed).count()
```

### 임의 분할하기
```py
seed = 5
train_set, test_set = df.randomSplit([0.75, 0.25], seed)
```

### 로우 추가하기
```py
from pyspark.sql import Row

schema = df.schma
newRows = [
    Row("New Country", "Other Country", 5L),
    Row("New Country2", "Other Country2", 1L),
]
parallelizedRows = spark.spakrContext.parallelize(newRows)
newDF = spark.createDataFrame(parallelizedRows, schema)

df.union(newDF)\
  .where("count = 1")\
  .where("ORIGIN_COUNTRY_NAME =!= United States")\
  .show()
```

### 레코드 정렬하기
```py
from pyspark.sql.functions import asc, desc, col
df.sort("count").show(5)
df.orderBy("count", "DEST_COUNTRY_NAME").shw(5)
df.orderBy(col("count").asc(), col("DEST_COUNTRY_NAME").desc()).show(5)
```

### 로우수 제한하기
```py
from pyspark.sql.functions import expr
df.orderBy(expr("count desc")).limit(6).show()
```

<br>
<br>

##### Reference
[스파크 완벽 가이드](https://www.hanbit.co.kr/store/books/look.php?p_code=B6709029941)
