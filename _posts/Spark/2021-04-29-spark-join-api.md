---
title: Chapter 8 - 조인
categories: [Spark]
tags: [Spark, bigdata]
comments : true
---

## Chapter 8 - 조인

### inner_join
```python
joinExpression = person["graduate_program"] == graduateProgram['id']
joinDf = person.join(graduateProgram, joinExpression)

## or

joinType = 'inner'
joinExpression = person["graduate_program"] == graduateProgram['id']
joinDf = person.join(graduateProgram, joinExpression, joinType)
```

### full_outer_join
```python
joinType = 'outer'
joinExpression = person["graduate_program"] == graduateProgram['id']
joinDf = person.join(graduateProgram, joinExpression, joinType)
```

### left_outer_join
```python
joinType = 'left_outer'
joinExpression = person["graduate_program"] == graduateProgram['id']
joinDf = person.join(graduateProgram, joinExpression, joinType)
```

#### right_outer_join
```python
joinType = 'right_outer'
joinExpression = person["graduate_program"] == graduateProgram['id']
joinDf = person.join(graduateProgram, joinExpression, joinType)
```

#### left_semi_join
> 세미 조인은 오른쪽 DataFrame의 어떤 값도 포함하지 않기 때문에 다른 조인 타입과는 약간 다르다.
> 두번째 DataFrame은 값이 존재하는지 확인하기 위해 값만 비교하는 용도로 사용한다.
> 만약 값이 존재한다면 왼쪽 DataFrame에 중복 키가 존재하더라도 해당 로우는 결과에 포함된다.

```python
joinType = 'left_semi'

graduateProgram.join(person, joinExpression, joinType).show()
```

### left_anti_join
> 왼쪽 안티 조인은 왼쪽 세미 조인의 반대 개념이다.
> 왼쪽 세미 조인처럼 오른쪽 DataFrame의 어떤 값도 포함하지 않는다.
> 두 번째 DataFrame은 값이 존재하는지 확인하기 위해 값만 비교하는 용도로 사용하며,
> 두 번째 DataFame은 존재하는 값은 유지하는 대신
> 두번 째 DataFrame에서 관련된 키를 찾을 수 없는 로우만 결과에 포함한다.

```python
joinType = 'left_anti'
graduateProgram.join(person, joinExpression, joinType).show()
```

### cross_join(cartesian_join)
```python
joinType = 'cross'
graduateProgram.join(person, joinExpression, joinType).show()

### or

person.crossJoin(graduateProgram).show()
```

### 복합 데이터 타입의 조인
```python
from pyspark.sql.functions import expr

person.withColumnRenamed("id", "personId")\
  .join(sparkStatus, expr("array_contains(spark_status, id)")).show()
```

### 스파크의 조인 수행 방식
#### 네트워크 통신 전략
> 전체 노드 간 통신을 유발하는 셔플 조인(shuffle join)과 그렇지 않은 브로드캐스트 조인(broadcast join)이 있다.
> 큰 테이블과 큰 테이블의 조인시 셔플조인(거의 모든 워커노드에서 통신) 발생. 그러므로 큰 테이블과 작은 테이블 사이의
> 조인을 시도하는게 더 좋음(브로드캐스트 조인)

<br>
<br>

##### Reference
[스파크 완벽 가이드](https://www.hanbit.co.kr/store/books/look.php?p_code=B6709029941)
