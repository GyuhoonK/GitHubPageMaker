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

repartition

# Repartition ?

Spark에서 RDD, Dataset, DataFrame의 작업 최소단위는 partition입니다. 데이터에 작업(Job)을 적용할 때, Spark는 최소 단위인 partition 별로 쪼개서 수행(task)합니다.하나의 executor가 하나의 task, 즉 하나의 partition에 대해 작업을 수행합니다. 

이때, 해당 데이터(RDD, Dataset, DataFrame) 내부의 partition의 개수, 사이즈, 정렬상태는 task 수행에 영향을 줍니다. 예를 들어 아래와 같은 상황이라면 partition A를 전달받은 executor는 OOM을 피할 수 없을 것입니다. 

[image]

이러한 상황을 피하기 위해, partition 사이즈를 조절(rebalancing)할 수 있는데 이를 repartition이라고 부릅니다.

[image - repartition]

# Repartition by specific column

위에서 보았던 예시처럼 단순히 데이터 사이즈만 분배하는 것이 목적은 아닙니다. partition 전략을 적용할 수 있습니다. 이 때, 특정 column을 기준으로 전략을 수립합니다. 전략을 적용한다는 것은, 무작위로 partition에 데이터를 분배하는 것이 아니라 규칙에 따라 partition에 데이터를 분배하는 것입니다. 이러한 전략으로 hash partitoin, range partition은 Spark에서 기본으로 제공하고있고, 필요하다면 자신이 직접 전략을 만들어 적용할 수도 있습니다. 

이러한 전략을 적용하게 되면, 이후 partition 기준으로 사용된 column을 사용한 집계/조건 적용 시 쿼리 성능이 향상됩니다.

## hash partition

[hash image]

hash partition의 기본 전략은 특정 column에 hash function을 적용하여 나온 값인 hash value가 같은 값들은 같은 partition에 분배하는 것입니다.

이러한 전략에 의해 column 값이 같은 Row 객체들은 같은 partition에 위치하게 됩니다. 

[code]

## range partition

[range image]

range partition은 column 값에 대해서 지정된 개수만큼의 범위를 나누고 같은 범위에 속하는 값은 같은 partition에 분배합니다. 

[code]

## custom Partitioner

위 두가지 방법을 적용했을 때 rebalancing에 보장되는 것은 아닙니다. 

[code]

이처럼, partition 개수를 증가시키는 경우에는 데이터가 갖는 특징에 따라 partition에 데이터가 균등하게 분배되지 않을 수 있습니다.

이러한 경우에는 partition 전략을 만들어 적용해야합니다. 이러한  custom partition을 적용하기 위해서는 현재 다루고 있는 데이터의 도메인 지식이 필요할 것입니다.

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

# Sort rows in partition

`repartitionAndSortWithinPartitions`

partition에 분배된 데이터들을 정렬시킬 수도 있습니다.



## Data Size 

sort 시 parquet 파일 사이즈 줄어듬 

parquet 압축 방식의 특징

[참고]

https://m.blog.naver.com/syung1104/221103154997

https://techvidvan.com/tutorials/spark-partition/
