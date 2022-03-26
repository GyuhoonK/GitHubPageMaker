---
layout: post
current: post
cover: assets/built/images/hadoop/repartition-cover.jpg
navigation: True
title: repartition in Spark
date: 2022-03-12 22:30:00 +0900
tags: [hadoop]
class: post-template
subclass: 'post tag-hadoop'
author: GyuhoonK
---

repartition에 대해 알아보기

# Repartition ?



# Repartition by specific column

## hash partition 

## range partition

# Sort rows in partition

`repartitionAndSortWithinPartitions`

# Custom Partitioner

```scala
class CustomPartitioner(numParts: Int) extends Partitioner {
// 생성할 파티션의 개수를 리턴.
        override def numPartitions: Int = numParts

// 주어진 key에 대한 파티션 ID 리턴
// [주의] 음수를 리턴하지 않도록 조심해야함!
        override def getPartition(key: Any): Int = {
          val code = key.hashCode() % numPartitions
          if (code < 0) {
            code + numPartitions
          } else {
            code
          }
        }
        
// 이걸로 파티셔너가 동일한지 검사하므로 제대로 구현해야 함!
        override def equals(other: scala.Any): Boolean = other match {
          case custom: CustomPartitioner => custom.numPartitions == numPartitions
          case _ => false
        }
      }
```



## Data Size 

[참고]

https://m.blog.naver.com/syung1104/221103154997

https://techvidvan.com/tutorials/spark-partition/
