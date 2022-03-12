---
layout: post
current: post
cover: assets/built/images/scala-banner-post.png
navigation: True
title: Learning Apache Spark 3 with Scala (Section5)
date: 2022-02-21 22:30:00 +0900
tags: [scala]
class: post-template
subclass: 'post tag-scala'
author: GyuhoonK
---

Learning Apache Spark 3 with Scala (Section5 - Spark Progamming)

## broadcast

`broadcast`를 이용하면 모든 executor에 해당 객체(object)를 보낼 수 있습니다. `broadcast`를 이용하는 방법은 간단합니다. `sparkConetxt` 객체에 서 `broadcast` 함수를 사용하면 됩니다. 각 executor에서는 `.values`를 사용하여 해당 객체를 조회하고 사용할 수 있습니다. 

> Using the [broadcast functionality](https://spark.apache.org/docs/3.2.1/rdd-programming-guide.html#broadcast-variables) available in `SparkContext` can greatly reduce **the size of each serialized task**, and **the cost of launching a job** over a cluster. If your tasks use any large object from the driver program inside of them (e.g. a static lookup table), consider turning it into a broadcast variable. Spark prints the serialized size of each task on the master, so you can look at that to decide whether your tasks are too large; in general tasks larger than about 20 KiB are probably worth optimizing.
>

`broadcast` 함수를 이용하면, 하나의 노드에서만 해당 객체에 대한 연산을 수행하고 저장하고 있으므로, 메모리 사용을 줄이고 수행 시간을 줄일 수 있습니다. 

```scala
def loadMovieNames() : Map[Int, String] = {

  // Handle character encoding issues:
  implicit val codec: Codec = Codec("ISO-8859-1") // This is the current encoding of u.item, not UTF-8.

  // Create a Map of Ints to Strings, and populate it from u.item.
  var movieNames:Map[Int, String] = Map()

  val lines = Source.fromFile("data/ml-100k/u.item")
  for (line <- lines.getLines()) {
    val fields = line.split('|')
    if (fields.length > 1) {
      movieNames += (fields(0).toInt -> fields(1))
    }
  }
  lines.close()
  movieNames
}

val nameDict = spark.sparkContext.broadcast(loadMovieNames())
// nameDict is a broadcast variable.
...

val lookupName : Int => String = (movieID:Int)=>{
      nameDict.value(movieID)
    }
// broadcast variable을 호출할 때는 .value 사용
```



## sparkSQL is NOT always Faster than RDD

코드 작성의 편이성과 속도의 측면에서 sparkSQL을 사용하여 코드를 작성하는 것을 권장하지만, 이것이 항상 sparkSQL이 RDD보다 빠르다는 이야기는 아닙니다. 경우에 따라서는 RDD를 이용하는 것이 속도가 더 빠르기도 합니다. 

|         | Code                                                         | Execution Time |
| ------- | ------------------------------------------------------------ | -------------- |
| Dataset | [Dataset Code](https://github.com/GyuhoonK/sparkscala/blob/main/src/main/scala/com/sundogsoftware/spark/DegreesOfSeparationDataset.scala) | 7s             |
| RDD     | [RDD Code](https://github.com/GyuhoonK/sparkscala/blob/main/src/main/scala/com/sundogsoftware/spark/DegreesOfSeparation.scala) | 3s             |

위 처럼 Dataset API를 이용하는 경우보다 RDD API를 이용하여 작성한 경우가 2배 이상 빠른 경우도 있습니다. 상황에 맞는 API를 선택하여 활용해야합니다. 

Dataset, DataFrame API는 전통적인 데이터 분석 문제를 다루는데 효율적입니다. 즉 SQL 명령어를 활용하여 분석이 가능한 문제라면 sprakSQL API를 활용하는 것이 좋습니다.

반면, 새로운 프레임을 만들어 데이터 분석을 해야하는 경우에는 RDD와 같이 저수준 API에서 더 좋은 결과를 얻을 수 있습니다.

## cache, persist

모든 노드에서 자주 사용해야하는 데이터에 대해서는 `broadcast`가 아니라 `cache`, `persist`를 사용합니다.

|           | description                                                  |
| --------- | ------------------------------------------------------------ |
| `cache`   | lets you cache dataset to memory                             |
| `persist` | optionally lets you cache dataset to not jus memory but also disk |

아래와 같은 storage level에 대해서 선택이 가능합니다.

| Storage Level                          | Meaning                                                      |                                                              |
| :------------------------------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| MEMORY_ONLY                            | Store RDD as deserialized Java objects in the JVM. If the RDD does not fit in memory, some partitions will not be cached and will be recomputed on the fly each time they're needed. This is the default level. | The `cache()` method is a shorthand for using the default storage level, which is `StorageLevel.MEMORY_ONLY` (store deserialized objects in memory). |
| MEMORY_AND_DISK                        | Store RDD as deserialized Java objects in the JVM. If the RDD does not fit in memory, store the partitions that don't fit on disk, and read them from there when they're needed. |                                                              |
| MEMORY_ONLY_SER (Java and Scala)       | Store RDD as *serialized* Java objects (one byte array per partition). This is generally more space-efficient than deserialized objects, especially when using a [fast serializer](https://spark.apache.org/docs/latest/tuning.html), but more CPU-intensive to read. |                                                              |
| MEMORY_AND_DISK_SER (Java and Scala)   | Similar to MEMORY_ONLY_SER, but spill partitions that don't fit in memory to disk instead of recomputing them on the fly each time they're needed. |                                                              |
| DISK_ONLY                              | Store the RDD partitions only on disk.                       |                                                              |
| MEMORY_ONLY_2, MEMORY_AND_DISK_2, etc. | Same as the levels above, but replicate each partition on two cluster nodes. |                                                              |
| OFF_HEAP (experimental)                | Similar to MEMORY_ONLY_SER, but store the data in [off-heap memory](https://spark.apache.org/docs/latest/configuration.html#memory-management). This requires off-heap memory to be enabled. |                                                              |



[참고]

[Learning Apache Spark 3 with Scala](https://www.udemy.com/course/best-scala-apache-spark/)

[Broadcast Variables](https://spark.apache.org/docs/3.2.1/rdd-programming-guide.html#broadcast-variables)

[RDD Persistence](https://spark.apache.org/docs/3.2.1/rdd-programming-guide.html#rdd-persistence)
