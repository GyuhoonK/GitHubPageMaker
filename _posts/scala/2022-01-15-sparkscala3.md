---
layout: post
current: post
cover: assets/built/images/scala-banner-post.png
navigation: True
title: Learning Apache Spark 3 with Scala (Section3)
date: 2022-01-15 22:30:00 +0900
tags: [scala]
class: post-template
subclass: 'post tag-scala'
author: GyuhoonK
---

Learning Apache Spark 3 with Scala (Section3 - `RDD` & `RDD Action`)

# What is RDD?

 RDD는 복원력 있는(Resilient) 분산(Distributed) 데이터 세트(Dataset)입니다(Spark의 `Dataset`와는 다른 개념). RDD는 Spark의 오리지널 API이고, 이를 바탕으로 Spark `DataFrame`, `DataSet`과 같은 최신 API가 개발되었습니다. 물론, 최근에는 `DataFrame`, `DataSet`이 주로 이용되지만 그럼에도 불구하고 RDD를 이해하고 사용할 줄 아는 것은 필수적입니다. 이는 1. 여전히 RDD가 효율적인 솔루션이며 2. RDD를 사용하는 타사 라이브러리의 레거시 코드를 분석할 수 있으며, 3. RDD가 Spark의 핵심 API이기 때문입니다. 

RDD는 기본적으로 일련의 데이터(Dataset)입니다. 또한 행(row)로 나뉘어져 있기 때문에 각 행들을 서로 다른 컴퓨터에 분배(Distributed)하여 병렬 처리할 수 있습니다. 여기서 Resilient란 Spark가 RDD에서 실행되는 작업을 어떤 식으로든 이루어지도록 해준다는 의미입니다. 즉, 작동 도중에 한 노드(node)에서 문제가 발생하면 상황을 파악하고 새로운 노드로 전환시켜 해당 문제 상황을 해결하고자 합니다.

RDD의 이러한 분산, 복원력이라는 특성을 위해서는 SparkContext를 먼저 만들어야합니다. SparkContext 오브젝트는 RDD의 복원(Resilient)과 분산(Distributed)을 가능하게 합니다. 이 과정에서 우리는 노드/하드웨어 장애 발생 시 어떻게 처리할지, 전체 클러스터에 데이터를 어떻게 배포해야하는지에 대해는 작성하지 않습니다. 이러한 작업은 SparkContext 내의 RDD 객체가 자동으로 찾습니다! 오로지 우리가 신경써야할 부분은 데이터의 변환(transformation)입니다. 

# Creating RDD's

RDD 객체를 생성하는 대표적인 몇 가지 방법에 대해 알아보겠습니다.

가장 간단하게는 `List` 객체를 전달하는 방법이 있습니다.

```scala
val nums = sc.parallezize(List((1,2,3,4)))
```

저장되어있는 데이터를 RDD 객체로 불러오는 것도 가능합니다.

```scala
// Local에 저장된 txt 파일 불러오기
sc.textFile("file:///C:/Users/frank/gobs-o-text.txt")
// 이와 같은 저장소에서도 불러올 수 있습니다(s3n://, hdfs://)
```

Hive에서도 불러올 수 있습니다. 이 경우 HiveContext를 선언해야합니다.

```scala
hiveCtx=HiveContext(sc)
rows = hiveCtx.sql("SELECT name, age FROM users")
```

 이외에도  JDBC, cassandra, Hbase, ElasticSeartch, JSON, CSV  등 여러 포맷과 데이터소스로부터 RDD를 생성할 수 있습니다.

# Transforming RDD's

- map

  MapReduce처럼 작동하여 RDD의 모든 행에 어떤 기능을 적용할 수 있게 해줍니다. map은 원본 RDD 행과 결과  RDD 행이 1:1 관계를 가집니다. 

  ```scala
  val rdd = sc.parallezize(List(1,2,3,4))
  val squared = rdd.map(x => x*x)
  println(squared)
  // 1, 4, 9, 16
  ```

- flatmap

  map과 유사하지만 1:1 관계를 갖지 않습니다. 원본 RDD 행 하나로부터 여러 행을 만들 수 있습니다. 예를 들어, RDD 행이 List라면 flatmap을 적용하여 각각의 행으로 나눌 수 있습니다.

  ```scala
  // input = "Hello World!" 1 row RDD 
  val words = input.flatMap(x => x.split("\\W+"))
  // Hello, World : 2 rows RDD
  ```

- filter

  데이터를 삭제하고 정리할 수 있습니다. 적용하는 함수의 결과가 false인 행은 버려집니다.

  ```scala
  val minTemps = parsedLines.filter(x => x._2 == "TMIN")
  // minTepms는 각 row의 2번째 field가 "TMIN"인 경우로 필터링됨
  ```

- distinct

  중복되는 값을 제거합니다.

- sample

  표본을 추출합니다.

- union, intersection, subtract, cartesian, etc.

# RDD Actions

Transform을 적용한 RDD 결과는 Action을 통해 반환받을 수 있습니다. Spark의 기본적인 전략은 Lazy Evaluation이기 때문에 action이 호출되기 전까지 Spark는 어떠한 연산도 수행하지 않습니다. 따라서 action이 실행되기 전까지 최적화하지 않다가 action이 호출되는 순간, Spark는 Directed Acyclic Graph를 생성하여 최적화 작업을 시작합니다.

Action은 RDD를 collapse하여 Driver Script로 반환합니다. 다음과 같은 action들이 존재합니다.

- collect

  RDD에 있는 모든 데이터 행을 반환합니다. 

- count

   RDD에 있는 데이터 행의 개수를 반환합니다. 

- countByValue

  key-value 형식의 RDD에 대해서 key에 대해 몇 개의 행이 존재하는지 반환합니다.

- take

- top

- reduce

- and more ...

# Practice : TotalSpentByCustormer.scala

```scala
package com.sundogsoftware.spark
import org.apache.spark._
import org.apache.log4j._

object TotalSpentByCustormerGyuhoon {
  def parseLine(line:String): (Int, Float) = {
    val fields = line.split(",")
    val field1 = fields(0).toInt // first field
    val field2 = fields(2).toFloat // thrid field
    (field1, field2)
  }

  def main(args: Array[String]): Unit = {
     // Set log level
    Logger.getLogger("org").setLevel(Level.ERROR)
    // SparkContext
    val sc = new SparkContext(master="local[*]", // User All CPU
                              appName="TotalSpentByCustormerGyuhoon") 
    												
    val lines = sc.textFile("data/customer-orders.csv")
    // map -> 1:1 trnasformtaion
    val parsedLines = lines.map(parseLine)
    // reduceByKey(trnasformtaion)
    val totalAmountByID = parsedLines.reduceByKey( (x,y) => x+y)
    // map & sortByKey(trnasformtaion)
    val idByTotal = totalAmountByID.map(x => (x._2, x._1)).sortByKey()
    // collect(action)
    val results = idByTotal.collect()
    // Print the results.
    results.foreach(println)
  }
}
```



[참고]

[Learning Apache Spark 3 with Scala](https://www.udemy.com/course/best-scala-apache-spark/)
