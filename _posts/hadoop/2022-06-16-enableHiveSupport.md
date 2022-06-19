---
layout: post
current: post
cover: assets/built/images/hadoop/sparkhive.jpeg
navigation: True
title: enableHiveSupport
date: 2022-06-16 22:30:00 +0900
tags: [hadoop]
class: post-template
subclass: 'post tag-hadoop'
author: GyuhoonK
---

Spark enableHiveSupport(Hive metaStore)

# Hive Table in Spark

spark에서 Hive table에 접근할 수 있도록 설정하기 위해서 'enableHiveSupport()'를 주로 사용하는데요. 

```python
spark = SparkSession.builder.enableHiveSupport().config(conf=conf).getOrCreate()

df = spark.sql("""
SELECT *
FROM default.test1
""")
```

`enableHiveSupport()`의 의미가 무엇인지 생각해보지 않았던 것 같아서 한번 조사해보았습니다.

이에 대해 알기 위해서는 Hive Table Location에 대해서 먼저 알아야합니다.

## Hive - Table Location

예를 들어 defualt.test_1 이라는 테이블이 있다고 가정하면, 해당 테이블은 HDFS 상에서 '/usre/hive/warehouse/default.db/test_1' 경로에 저장되어 있습니다.

이를 `TABLE LOACTION`이라고 합니다.

나아가, PARTITIONED TABLE의 경우에는 해당 LOCATION 하위에 PARTITION DIR이 더 존재합니다.

| 구분              | TABLE                             | HDFS LOCATION                                 |
| ----------------- | --------------------------------- | --------------------------------------------- |
| Table             | default.table1                    | /user/hive/warehouse/default.db/table1        |
| Partitioned Table | default.table2 PARTITION (col1=a) | /user/hive/warehouse/default.db/table2/col1=a |

우리가 Hive에 쿼리를 작성하여 테이블을 조회하는 것은 사실 HDFS에서 LOCATION에 존재하는 파일을 읽는 것입니다.

따라서, 우리가 아래 쿼리를 실행하면

```sql
SELECT *
FROM default.table1
```


Hive는 `default.table1`에 해당하는 LOCATION에 저장된 파일로부터 데이터를 읽어 결과를 출력합니다. 즉, 아래 커맨드로 확인할 수 있는 파일들을 스캔합니다.

```bash
$ hdfs dfs -ls /user/hive/warehouse/default.db/table1
```

## Hive metaStore

그렇다면 TABLE의 `LOCATION`과 `PARTITION` 정보 즉, **hive metastore**는 어디에 저장하고 있을까요? `hive-stie.xml`에서 hive metastore의 저장 위치를 확인할 수 있습니다.

hive metastore는 1) 임베디드, 2) 로컬, 3) 원격 셋 중에 하나로 구성할 수 있습니다.

아래는 hive metastore를 3) 원격으로 저장했을 경우 `hive-site.xml` 예시입니다.

```bash
$ cat /etc/hive/hive-site.xml
...
<property>
      <name>hive.metastore.uris</name>
      <value>thrift://host.example:9083</value>
 </property> 
...
```

`host.example:9083`에 hive metastore가 저장되어있음을 확인할 수 있습니다. 즉, 해당 경로에 hive table들의 location, partition 같은 메타 데이터들이 저장되어 있습니다. 

## enableHiveSupport

spark의 `enableHiveSupport()`는 `hive.metastore.uris` 설정 값에 접근하여 hive metastore를 사용하겠다 라는 의미입니다.

default로 `hive-site.xml`에서 정의된 값을 사용하고, 정의되어있지 않다면  `spark.sql.warehouse.dir` 로 전달된 경로에서 metastore를 검색합니다. 

> When working with Hive, one must instantiate SparkSession with Hive support, including connectivity to a persistent Hive metastore, support for Hive serdes, and Hive user-defined functions. Users who do not have an existing Hive deployment can still enable Hive support. When not configured by the hive-site.xml, the context automatically creates metastore_db in the current directory and creates a directory configured by **spark.sql.warehouse.dir**, which defaults to the directory spark-warehouse in the current directory that the Spark application is started. Note that the hive.metastore.warehouse.dir property in hive-site.xml is deprecated since Spark 2.0.0. Instead, use spark.sql.warehouse.dir to specify the default location of database in warehouse. You may need to grant write privilege to the user who starts the Spark application.

위의 정보를 종합해보면 spark가 hive table을 읽어오는 과정은 다음과 같습니다.

1.  `enableHiveSupport()`를 선언한 `sparkSession`에서 `spark.sql`을 이용하여 테이블(`default.test1`)을 조회합니다.
2. spark는 `hive-site.xml`  혹은 `spark.sql.warehouse.dir` 에서 hive metastore가 저장된 경로를 참조합니다.
3. hive metastore로부터 해당 테이블의 경로(`/user/hive/warehouse/default.db/test1`)를 알아냅니다. 
4. 이후 해당 테이블 위치(location)에서 파일(데이터)를 읽어옵니다.

따라서 아래 두 방법은 기본적으로 같은 동작을 수행합니다.

```python
# query
df1= spark.sql("""
SELECT *
FROM default.test1
""")

# file scan
df2 = spark.read.parquet("/user/hive/warehouse/default.db/test1")
```

Partitioned Table도 위와 크게 다르지 않습니다. 단지 table location에 partition column이 추가될 뿐입니다.

```python
# query
df1= spark.sql("""
SELECT *
FROM default.test1
WHERE col1 = 'a'
""")

# file scan
df2 = spark.read.parquet("/user/hive/warehouse/default.db/test1/col1=a")
```



[참고]

[5-메타스토어](https://wikidocs.net/28353)

[sql-data-sources-hive-tables](https://spark.apache.org/docs/latest/sql-data-sources-hive-tables.html)
