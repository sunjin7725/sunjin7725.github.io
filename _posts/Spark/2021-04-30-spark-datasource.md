---
title: Chapter 9 - 데이터 소스
categories: [Spark]
tags: [Spark, bigdata]
comments : true
---

## Chapter 9 - 데이터 소스

### JSON 파일 읽고 쓰기

```python
spark.read.format('json').option('mode', 'FAILFAST')\
      .option('inferSchema', 'true')\
      .load('dir').show(5)

 csvFile.write.format('json').mode('overwrite').save('dir')
```

### Parquet 형식 읽고 쓰기
> parquet형식은 json이나 csv보다 읽기 연산시 훨씬 효율적으로 동작하므로 장기 저장용 데이터는
> parquet형식으로 저장하는 것이 좋음.
> json이나 csv에 비해 옵션이 거의 없음. orc형식도 마찬가지고 둘 사이에 차이는 거의 없지만
> spark에 최적화 된 것은 parquet. hive는 orc

```python
 spark.read.format('parquet')\
      .load('dir').show()

 csvFile.write.format('parquet').mode('overwrite')\
        .save('dir')
```

### Database 읽고 쓰기

```python
driver = 'org.sqlite.JDBC'
path = 'dir'
url = 'jdbc:sqlite:' + path
tableName = 'target'

# sql테이블을 읽어 dataframe화. sqlite
dbDataFrame = spark.read.format('jdbc').option('url', url)\
                        .option('dbTable', tableName).option('driver', driver).load()


# postgreSQL에서 테이블을 읽어 dataframe화
# 좀 더 많은 옵션 필요
pgDF = spark.read.format('jdbc')\
                 .option('driver', 'org.postgresql.driver')\
                 .option('url', 'jdbc:postgresql://database_server')\
                 .option('dbTable', 'schema.tableName')\
                 .option('user', 'username').option('password', 'my-secret-password').load()
```

#### query push down
> spark에서 dataframe을 만들기 전에 데이터베이스 자체에서 데이터를 필터링하도록 하는 방식
> 연결된 db에서 데이터가 필터링 된 후 필터링 된 데이터로 spark dataframe을 만듦

```python
pushDownQuery = """(select distinct(dest_country_name) from flight_info) as flight_info"""
dbDataFrame = spark.read.format('jdbc')\
                        .option('url', url)\
                        .option('dbTable', pushDownQuery)\
                        .option('driver', driver)\
                        .load()
```

#### 데이터베이스 병렬로 읽기
```python
# option에 numPartitions 추가

dbDataFrame = spark.read.format('jdbc')\
                   .option('url', url)\
                   .option('dbTable', tablename)\
                   .option('driver', driver)\
                   .option('numPartitions', 10).load()

# 필터를 db에 위임할 수 있지만(push down), spark 자체 partition에 결과 데이터를 저장함으로써 더 많은 처리 가능
# 데이터소스 생성 시 조건절 목록을 정의해서 spark 자체 partition에 결과 데이터 저장 가능
props = {'drivers':'org.sqlite.JDBC'}
predicates = [
    "dest_country_name = 'Sweden' or origin_country_name = 'Sweden'",
    "dest_country_name = 'Anguilla' or origin_country_name = 'Anguilla'"
]
spark.read.jdbc(url, tablename, predicates=predicates, properties=props).show()
spark.read.jdbc(url, tablename, predicates=predicates, properties=props)\
          .rdd.getNumPartitions() # 2
```

#### SQL쓰기
```python
newPath = 'jdbc:sqlite://tmp/my-sqlite.db'
props = {'drivers':'org.sqlite.JDBC'}

csvFiles.write.jdbc(newPath, tablename, mode='overwrite', properties=props)
csvFiles.write.jdbc(newPath, tablename, mode='append', properties=props)
```

### 텍스트파일 읽고 쓰기

```python
spark.read.textFile("/data/spark/data/flight-data/csv/2010-summary.csv")\
          .selectExpr("split(value, ',') as rows").show()

# 텍스트파일을 쓸(write) 때에는 문자열 컬럼이 하나만 존재해야 함.
csvFiles.select('dest_country_name').write.text('/tmp/simple-txt-file.txt')

# 텍스트 파일에 데이터를 저장할 때 파티셔닝 작업을 수행하면 더 많은 컬럼 저장 가능. 하지만 모든 파일에 컬럼을 추가하는 것이 아닌
# 텍스트 파일이 저장되는 디렉터리에 폴더별로 컬럼 저장
csvFiles.limit(10).select('dest_country_name', 'count')\
                  .write.partitionBy('count').text('/tmp/five-csv-files2py.csv')
```

#### 파티셔닝
```python
# 나라별로 파티셔닝된 데이터들 각 조건에 맞는 디렉터리에 저장됨
# 파티셔닝은 필터링을 자주 사용하는 테이블을 가진 경우에 사용 가능한 가장 손쉬운 최적화 방식
csvFiles.limit(10).write.mode('overwrite').partitionBy('dest_country_name')\
                        .save('/tmp/partitioned-files.parquet')
```

#### 버케팅(Bucketing)
```python
# 특정 컬럼을 파티셔닝하면 수억개의 디렉터리를 만들어 낼 수 있음.
# 이런 경우 데이터를 모아서 저장하는 방법이 필요한데 이를 버케팅이라 함
# 이후의 사용방식에 맞춰 사전에 파티셔닝되므로 조인이나 집계 시 발생하는 고비용의 셔플을 피할 수 있음

csvFiles.write.format('parquet').mode('overwrite')\
        .bucketBy(numberBuckets, columnToBucketBy).saveAsTable('bucketedFiles')
```

