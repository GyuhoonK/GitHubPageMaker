---
layout: post
current: post
cover: assets/built/images/hadoop/spark-overwrite-partition.jpg
navigation: True
title: Overwrite Partition in Spark
date: 2021-12-21 22:30:00 +0900
tags: [hadoop]
class: post-template
subclass: 'post tag-hadoop'
author: GyuhoonK
---

Spark를 이용하여 특정 파티션만 overwrite하기

# Partitioned Table in Hive

Hive에서  파티셔닝을 이용하는 가장 큰 이유 중 하나는 쿼리 성능 향상입니다. HDFS 상에서 partition column을 기준으로 각각 다른 디렉토리에 저장되므로 쿼리 조회 시 조회가 필요 없는 파일은 조회를 수행하지 않으므로, 쿼리 성능이 향상될 수 있습니다.

예를 들어 설명해보겠습니다. 아래와 같은 간단한 테이블을 hive에 저장해보겠습니다.

```python
df = pd.DataFrame(data={'col1' : ['a','b','c'], 'col2':['apple','banana','car']})
df
|    | col1   | col2   |
|---:|:-------|:-------|
|  0 | a      | apple  |
|  1 | b      | banana |
|  2 | c      | car    |

sdf = spark.createDataFrame(df)
sdf.show()
+----+------+
|col1|  col2|
+----+------+
|   a| apple|
|   b|banana|
|   c|   car|
+----+------+

schema, table = 'default.t_test'.split('.')
sdf.write.mode('append')\
    .partitionBy("col1")\
    .option("path", f'/user/hive/warehouse/{schema}.db/{table}')\
    .saveAsTable(f"{schema}.{table}")
    
spark.sql(f"SELECT * FROM {schema}.{table}").show()
+------+----+
|  col2|col1|
+------+----+
|banana|   b|
| apple|   a|
|   car|   c|
+------+----+
```
`default.t_test`  테이블은 아래와 같은 구조로 저장됩니다.

```shell
hdfs -dfs ls /user/hive/warehouse/default.db/t_test # hdfs 디렉토리 조회 
>> /user/hive/warehouse/default.db/t_test/col1=a
>> /user/hive/warehouse/default.db/t_test/col1=b
>> /user/hive/warehouse/default.db/t_test/col1=c
```

따라서, 만약 아래와 같은 쿼리를 실행한다면 `col1=a` 디렉토리 내부의 파일만 조회하여 결과를 반환하므로 파티션이 적용되지 않은 테이블보다 쿼리 성능이 향상됩니다.

```sql
SELECT *
FROM   default.t_test
WHERE  col1='a'; -- col1=a 디렉토리만 조회
```

이외에도 파티션 테이블이 갖는 장점이 하나 더 있습니다.

# Overwrite Only a Single(or Specific) Partition

각 파티션에 해당하는 디렉토리 내에 파일이 따로 저장되어 있으므로 테이블의 수정이 필요한 경우 수정이 필요한 파티션에 대해서만 overwrite할 수 있습니다.

예를 들어, 아래 쿼리를 이용하여 위 테이블에서 `col=1` 파티션을 `tmp` 테이블(혹은 뷰) 내용으로 수정할 수 있습니다.

```sql
INSERT OVERWRITE TABLE default.t_test PARTITION(col1='a')
SELECT col2
FROM tmp;
```

HiveQL은 `INSERT OVERWRITE`와 STATIC PARTITION 문법을 통해 이러한 Partition Overwrite가 비교적 간단한 편입니다.

Spark에서는 Partition Overwrite를 어떻게 구현할 수 있을까요?

## Test1 : saveAsTable

`col1=a` 파티션의 내용을 변경하기 위한 데이터를 만들어 실험해보겠습니다. 목표는 다른 파티션은 건드리지 않고, `{a:apple}`인 현재 상태를 `{a:art}`로 변경하는 것입니다.


```python
a_art_df = pd.DataFrame(data={'col1' : ['a'], 'col2':['art']})
a_art_df
|    | col1   | col2   |
|---:|:-------|:-------|
|  0 | a      | art    |

a_art_sdf = spark.createDataFrame(a_art_df)
a_art_sdf.show()
+----+----+
|col1|col2|
+----+----+
|   a| art|
+----+----+
```

이제 `a_art_sdf`를 `default.t_test`에 `saveAsTable`을 이용하여 `overwrite`해보겠습니다. 


```python
a_art_sdf.write.mode('overwrite')\
    .partitionBy("col1")\
    .option("path", f'/user/hive/warehouse/{schema}.db/{table}')\
    .saveAsTable(f"{schema}.{table}")
    
spark.sql(f"SELECT * FROM {schema}.{table}").show()
+----+----+
|col2|col1|
+----+----+
| art|   a|
+----+----+
```

실행 결과 다른 파티션(`col1=b`, `col1=c`)이 모두 삭제되고 `col1=a` 파티션만 남았음을 확인할 수 있습니다.

이는 saveAsTable이 파티션에 관련된 메소드가 아니라, 테이블 단위의 메소드이기 때문입니다.

> `DataFrameWriter.saveAsTable`(*name*, *format=None*, *mode=None*, *partitionBy=None*, ***options*)[[source\]](https://spark.apache.org/docs/latest/api/python/_modules/pyspark/sql/readwriter.html#DataFrameWriter.saveAsTable)
>
> Saves the content of the [`DataFrame`](https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.sql.DataFrame.html#pyspark.sql.DataFrame) as the specified table.
>

따라서, `defaut.t_test` 내의 모든 내용에 대해서 overwrite가 발생했습니다.

## Test2 : insertInto

Spark의 `DataFrame`을 테이블로 저장하는 다른 메소드에 `insertInto`가 있습니다. 이를 이용하여 테스트해보았습니다.


```python
a_art_sdf.write.mode('overwrite')\
    .format('parquet')\
    .insertInto(f"{schema}.{table}")
    
spark.sql(f"SELECT * FROM {schema}.{table}").show()
+------+----+
|  col2|col1|
+------+----+
|banana|   b|
| apple|   a|
|   car|   c|
|     a| art|
+------+----+
```

`col1`, `col2`의 위치를 뒤바꿔 저장하는 모습을 보였습니다. 이는 `insertInto` 메소드가 칼럼 순서에 기반하여 테이블에 저장하기 때문입니다.

> Unlike [`DataFrameWriter.saveAsTable()`](https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.sql.DataFrameWriter.saveAsTable.html#pyspark.sql.DataFrameWriter.saveAsTable), [`DataFrameWriter.insertInto()`](https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.sql.DataFrameWriter.insertInto.html?highlight=insertinto#pyspark.sql.DataFrameWriter.insertInto) **ignores the column names** and **just uses position-based resolution**.

컬럼 순서를 바꿔서 다시 overwrite해보았습니다. 


```python
a_art_df = pd.DataFrame(data={'col2':['art'], 'col1' : ['a']})
a_art_df
|    | col2   | col1   |
|---:|:-------|:-------|
|  0 | art    | a      |

a_art_sdf = spark.createDataFrame(a_art_df)
a_art_sdf.show()
+----+----+
|col2|col1|
+----+----+
| art|   a|
+----+----+

a_art_sdf.write.mode('overwrite')\
    .format('parquet')\
    .insertInto(f"{schema}.{table}")
    
spark.sql(f"SELECT * FROM {schema}.{table}").show()
+------+----+
|  col2|col1|
+------+----+
|banana|   b|
| apple|   a|
|   art|   a| 
|   car|   c|
|     a| art|
+------+----+
```

분명 overwrite했음에도 불구하고, `col1=a`에 append된 결과를 보이고 있습니다. 

추측컨데, `insertInto` 메소드에 `overwrite=True` 옵션을 따로 줄 수 있는 것으로 보아, `write.mode`의 영향을 받지 않는 것으로 보입니다. 또한 이 메소드 역시 테이블 자체에 저장하는 방식이기 때문에 `overwrite=True` 옵션을 부여하게 되면 테이블 전체를 overwrite하여 `col1=b`, `col1=c` 파티션이 사라지게 됩니다.

```python
a_art_sdf.write.mode('overwrite')\
    .format('parquet')\
    .insertInto(f"{schema}.{table}", overwrite=True)
spark.sql(f"SELECT * FROM {schema}.{table}").show()

+------+----+
|  col2|col1|
+------+----+
|   art|   a|
+------+----+
```

## Test3 : save (directly)

마지막으로 실험해본 방법은 파티션 경로에 직접 해당 파일을 저장하는 것입니다. 

이 경우 주의해야할 점은 partition column(이 경우에는 `col1`)은 제외하고 저장해야한다는 점입니다. partition column에 해당하는 value는 디렉토리 이름(`col1=a`)를 통해 추론되는 값이기 때문입니다.


```python
a_art_sdf\
    .drop('col1')\
    .write.mode('overwrite')\
    .format('parquet')\
    .save(f'/user/hive/warehouse/{schema}.db/{table}/col1=a')
    
spark.sql(f"REFRESH TABLE {schema}.{table}") # 파일을 직접 수정했으므로, 캐싱된 메타데이터 update 실행
spark.sql(f"SELECT * FROM {schema}.{table}").show()
+------+----+
|  col2|col1|
+------+----+
|banana|   b|
|   art|   a|
|   car|   c|
+------+----+
```

성공했습니다! `col1=a`에 해당하는 값이 `apple`에서 `art`로 변경되었습니다.


```shell
hive -e "SELECT * FROM default.t_test"
+---------+-------+
|  col2   | col1  |
+---------+-------+
| art     | a     |
| banana  | b     |
| car     | c     |
+---------+-------+
```

HiveQL로 조회하는 경우에도 정상적으로 값이 조회됩니다!



# 끝내며

간단한 글이지만, 정리하지 않으면 항상 헷갈리고 혼란스러운 내용이 될 것 같아 정리해보았습니다.

테스트에 사용한 버전은 Spark 2.3.0이며 Spark 3.0 이상의 버전에서는 좀 더 멋지고 깔끔한 메소드가 있을지도 모릅니다.



감사합니다.



[참고]

[Overwrite specific partitions in spark dataframe write method](https://stackoverflow.com/questions/38487667/overwrite-specific-partitions-in-spark-dataframe-write-method)
