---
layout: post
current: post
cover: assets/built/images/scala-banner-post.png
navigation: True
title: Learning Apache Spark 3 with Scala (Section4)
date: 2022-02-21 22:30:00 +0900
tags: [scala, hadoop]
class: post-template
subclass: 'post tag-scala'
author: GyuhoonK
---

Learning Apache Spark 3 with Scala (Section4 - ``DataFrame`` & `Dataset`)



## SparkSQL

SparkSQL와 함께 `DataFrame`, `Dataset`이라는 현대적인 API가 등장했습니다. 이들 API는 대부분의 데이터 분석 문제가 SQL 명령어로 해결되는 것처럼, Spark내에서도 SQL 명령어를 사용할 수 있게끔 지원합니다. 

Spark 2가 도입되면서 SparkSQL 인터페이스를 더 강조하기 시작했습니다. 그리고 Spark2부터  `DataFrame`, `Dataset`이 적극적으로 개발되었습니다.

## `DataFrame`

RDD를 `DataFrame` 객체로 확장합니다. `DataFrame`은 RDB와 비슷한 점이 많습니다. `DataFrame`은 `Row` 객체와 Schema라는 두 가지 특징을 갖습니다.  먼저, `DataFrame`은 `Row` 객체로 나뉘어집니다. `Row` 객체는 구조화된 정보(structured information)를 담고 있습니다. 두번째로, schema를 가지고 있기에 효율적으로 정보를 저장할 수 있습니다. 그리고 스키마와 `Row`가 있기 때문에 우리는 `DataFrame`을 DB처럼 다룰 수 있고, SQL 커맨드를 실행할 수 있습니다. 

`DataFrame`은 RDB와 비슷해 보이기 때문에, RDB와의 상호운영성(inter-operability)을 확보할 수 있습니다. 이는 Json, Hive, Parquet과 같은 다양한 데이터 소스로부터 파일/데이터를 읽고 쓸 수 있음을 의미합니다. 이외에도 JDBC/ODBC 인터페이스나 Taleau에 직접 작업도 가능합니다.

SQL 쿼리로 접근할 수 있다는 사실은 쿼리 최적화(query optimization)을 `DataFrame`에 적용할 수 있다는 의미이기도 합니다.

그래서 보통 DAG(Directed Acyclic Graph) 최적화 이후, 실행되는 SQL 쿼리를 볼 수 있고 해당 쿼리에 RDB에 시도하는 것처럼 쿼리 최적화를 적용해볼 수도 있습니다.

##  `Dataset`

```scala
case class Person(id:Int, name:String, age:Int, friends:Int)

import spark.implicits._
val schemaPeople = spark.read
      .option("header", "true")
      .option("inferSchema", "true")
      .csv("data/fakefriends.csv")
      .as[Person]
```



`Dataset`은 `DataFrame`과 같은 종류입니다. 기술적으로, `DataFrame`은 `Row` 객체로 이루어진 `Dataset`의 하나 일뿐입니다. 차이점은 `Dataset`이 명확한 타입(Expicit Type)을 갖고 있으며 컴파일 타임에 이를 검사한다는 사실입니다. 다시 말해, `DataFrame`은 실행 단계에서 스키마를 적용하지만, `Dataset`이 컴파일 타임에 스키마를 검사합니다.

이러한 차이점 때문에 `DataFrame`은 `Row` 객체가 존재하고, `Row`를 정의할 때까지 무엇이든 담을 수 있는 것에 비해 `Dataset`은 스키마를 통해 명확한 타입을 지정했으므로 `Row` 객체에 담을 수 있는 자료형이 분명합니다(제한됩니다). 따라서, `Dataset`은 컴파일 타임에 이미 스키마를 알고 있으므로 스크립트 실행 시가 아니라 스크립트 **빌드** 시에 Type Error를 발견할 수 있습니다. 클러스터에서 스크립트 실행 시 많은 비용을 요구하기 때문에 실행 전에 미리 이러한 에러를 알아챌 수 있는 것은 큰 장점입니다. 또한 `Dataset`을 이용하면 더 나은 최적화를 가능하게 합니다. 런타임이 아니라, 컴파일하는 동안 최적화가 진행되기 때문입니다.

그러나, 이러한 작업들은 컴파일 타임에 이루어지기 때문에 컴파일된 언어(Java, Scala)로만 사용이 가능합니다. python은 컴파일 타임 최적화가 불가능합니다.

[현재]() Spark 개발의 전반적인 트렌드는 RDD를 적게 사용하고 `Dataset`을 더 많이 사용하는 것입니다. RDD에 비해 `Dataset`이 갖는 장점은 먼저, `Dataset`은 매우 효율적으로 직렬화(serialized)됩니다. 컴파일 타임에 실행되는 실행 계획(execution plan)때문 입니다. 그리고 상호운영성(interoperability)가 뛰어납니다. RDB, 외부 파일 형식(parquet, json)과 호환될 수 있습니다. JDBC, ODBC에도 접근하여 데이터를 로드할 수 있습니다. 이는 spark를 수평적으로 확장된 데이터베이스(horizontally-scalable database)처럼 작동하게 만듭니다. 마지막으로 `Dataset`은 Spark MLLib과 Streaming에서 주로 사용됩니다. RDD로는 이들 라이브러리에 데이터를 전달할 수 없습니다.

## SparkSession

RDD로 작업할 때는 `SparkContext를` 이용했습니다. 그러나 sparkSQL을 사용하려면 `SparkSession`을 사용해야합니다. 이는 DataBase Session과 유사한 개념입니다.

```scala
val spark = SparkSession
            .builder
            .appName("SparkSession_test")
            .master("local[*]")
            .getOrCreate()
```

이제 `SparkSession` 객체로부터 `SparkContext를` 얻을 수 있고 `Dataset`에 쿼리를 날릴 수 있습니다.

## sparkSQL API

`show`, `select`, `filter`, `groupBy` 같은 쿼리를 실행할 수 있습니다.

```scala
	myResultDataSet.show()
	myResultDataSet.select("FieldName")
	myResultDataSet.filter(myResultDataset("FieldName")>200)
	myResultDataset.groupBy(myResultDataset("FieldName")).mean()
	myRestulDataset.rdd().map(mapperFunction)
```

마지막 라인처럼,  RDD로 다시 변환해서 자신이 만든 map Function을 적용해볼 수도 있습니다.

## UDF(User Defined Function)

사용자 정의 함수(UDF)가 있습니다. 데이터베이스에서 정의하여 사용하는 UDF와 비슷한 개념입니다.

`org.apache.spark.sql.functions.udf`를 임포트해서 간단한 사용자 정의 함수를 만들 수 있습니다.

```scala
import org.apache.spark.sql.functions.udf
val square = udf{(x => x*x)}
squaredDF = df.withColumn("square", square("value"))
```

## $ before string in Scala(Spark)

scala를 이용하여 spark 코드를 작성하다 보면 아래와 같이 $이 string 앞에 쓰이는 것을 자주 마주칩니다.

```scala
// example1
val words = input
      .select(explode(split($"value", "\\W+")).alias("word"))
// example2
df.select($"name", $"age" + 1).show()
```

이는 scala에서 사용되는 문법이라기 보다, spark에서 사용되는 `column` 객체 표현 방법입니다. 또한 `$"age"+1`처럼 `column`에 연산을 실행하는 경우에 이와 같은 표현이 사용됩니다. 

scala는 아래와 같이 작성하는 경우에 에러가 발생합니다.

```scala
dataframe.select("age"+1).show()
>>> org.apache.spark.sql.AnalysisException: cannot resolve '`age1`' given input columns: [age, name];;
'Project ['age1]
```

[참고]

[Learning Apache Spark 3 with Scala](https://www.udemy.com/course/best-scala-apache-spark/)

[what is the output of a $ some string in scala](https://stackoverflow.com/questions/42427388/what-is-the-output-of-a-some-string-in-scala)
