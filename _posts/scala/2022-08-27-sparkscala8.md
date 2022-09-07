---
layout: post
current: post
cover: assets/built/images/scala-banner-post.png
navigation: True
title: Learning Apache Spark 3 with Scala (Section8)
date: 2022-08-27 22:30:00 +0900
tags: [scala]
class: post-template
subclass: 'post tag-scala'
author: GyuhoonK
---

Learning Apache Spark 3 with Scala (Section8 -Spark Streaming)

# Spark Streaming

한 번에 많은 데이터를 처리하는 Batch Process와 달리 Streaming은 실시간으로 유입되는 데이터를 모니터링하고 처리합니다. 대표적으로 웹사이트, 서버에서 발생하는 로그 데이터 처리가 Streaming에 해당합니다. 실시간으로 데이터를 처리하는 것 뿐 아니라, 유입되는 데이터를 특정 간격 단위(interval, window)로 계산(aggregate)하고 분석(analyze)할 것입니다. 

Streaming에서 데이터는 끊임없이 유입됩니다. 따라서  streaming data는 로우 파일(raw file)로 유입되지 않을 수 있습니다. TCP 데이터를 받는 포트, Amazon의 Kinesis, HDFS와 같은 분산 처리 시스템, Kafka, Flume과 같이 다양한 데이터 소스들로부터 발생하는 데이터를 처리할 수 있어야합니다. 

Spark Streaming은 이러한 모든 데이터 소스를 통합(integrate)하고, 보내지는 데이터를 분석(analyze)할 수 있습니다. 뿐만 아니라 데이터를 변형(transform)하여 다른 데이터 소스로 전송할 수도 있습니다. 대부분의 경우 로그 데이터를 가져와서, 특정 정보를 추출하고, 원하는 곳에 저장하는 케이스일 것입니다.

## Checkpoint

Spark Streaming의 가장 강력한 기능은 `checkpoint`입니다. `checkpoint` 는 fault tolerance를 위해 제공되는 기능입니다. 만약 streaming을 실행중인 클러스터가 다운되거나, streaming이 다운된다고 가정해봅시다. 이러한 경우에 `checkpoint` 는 자동으로 다시 시작해야할 지점을 선택해줍니다. 



# DStream API

Spark Steaming은 DStream에서 시작되었습니다. DStream은 RDD에 기반한 API입니다. RDD에 기반하고 있다는 특징 때문에, DStream은 microbatch와 같은 작업에 특화되어있습니다. 즉, RDD로 표현되는 데이터 덩어리(chunks of data)를 다루고 처리할 수 있습니다. 이러한 작업은 data field나 row-by-row같은 방법으로 작동하지 않습니다. microbatch를 통해 데이터를 한 덩어리(chunk)로 만듭니다.

그러나 현재 DStream은 잘 사용되지 않습니다. 최근 Application들이 즉각적인 데이터 처리를 지원하는 데에 비해 몇 초 정도의 지연이 있지만 이러한 부분이 문제가 되는 것은 아닙니다. DStream을 사용하지 않게 된 가장 큰 이유는 시간이 지나며 Spark Stream의 인터페이스로부터 배척되었기 때문입니다. DStream보다 최신 API가 등장했고 이에 따라 Dstream은 사용하지 않는 추세입니다.

```scala
val stream = new StreamingContext(conf, Seconds(1))
val lines = stream.socketTextStream("localhost", 8888)
val errors = lines.filter(_.contains("error"))
error.print()

stream.start() // start streaming
streama.awaitTermination() // wait for termination
```

DStream에서는 `SparkContext`가 아니라, `StreamingContext`를 선언합니다. 위 예시의 경우 1초에 한번 씩, job을 처리하도록 셋팅되었습니다. 즉 1초마다 microbatch를 실행합니다. 

1초마다 microbatch는 localhost의 8888 포트로 텍스트 데이터를 전송합니다. 이때, 에러가 발생하는 경우 필터링하여 에러가 있는 텍스트 라인을 출력하고 있습니다. 

위 코드는 하나의 데이터 청크(a single chunk of data)를 처리하지 않습니다. 반복적으로 1초마다 유입되는 데이터들을 전송하고, 필터링하는 과정을 거칩니다. DStream에서 microbatch마다 처리하는 RDD는 유입되는 데이터의 하나의 작은 청크(one little chunk)입니다. 

## Window

Windowed operations을 사용하면, 다수의 배치로부터 발생한 결과를 결합시킬 수 있습니다. `window(), reduceByWindow(), reduceByKeyAndWindow()`와 같은 함수가 존재하고, 이들을 이용하여 일정 시간 동안의 데이터를 reduce할 수 있습니다. 몇 분, 몇 시간, 며칠과 같은 기간 동안의 데이터를 모아 reduce시킵니다. 기본적으로 window size는 batch와 일치할 필요가 없습니다. batch보다 클 수도 있습니다.

## updateStateByKey

`updateStateByKey()`를 이용하면 시간이 흘러도, 많은 배치 작업들 간의 스테이트(state)를 유지할 수 있습니다. 따라서, window, batch에 따른  running state를 추적해야한다면 `updateStateByKey()`를 사용해야합니다. count가 필요한 경우가 대표적인 `updateStateByKey()`가 사용되는 예입니다.



# Structrued Streaming

DStream이 Spark의 오리지날 스트리밍 API이긴 하지만, 최근에는 `Structured Streaming` 이 주로 사용되고 있습니다. `Structured Streaming`은 `DataSets`를 사용합니다. `DataSets`를 기반으로 스트리밍 작업을 구현하기 때문에 쿼리를 작성하여 실행할 수도 있습니다. 또한 새로운 정보가 들어올 때마다 `DataSets`에 실시간으로  row를 추가하는 방식으로 작업을 단순화시킬 수도 있습니다. 

`Structured Streaming`에서 다루는 `DataSets`는 지금까지 다루었던 `DataSet`과 큰 차이가 없습니다. 따라서 지금까지 배웠던 API를 배치 작업에 거의 그대로 사용할 수 있습니다. 

코드로 이를 구현하는 것 또한 매우 간단합니다. `spark.read`를 `spark.readStream`으로만 교체하면 됩니다. 

```scala
val inputDF = spark.readStream.json("s3://logs")
inputDF.groupBy($"action", window($"time", "1 hour")).count()
	.writeStream.format("jdbc").start("jdcb:mysql//...")
```

위 코드는 **실시간으로** s3 storage로부터 log 파일을 읽어 들이고,  `groupBy`, `window`를 적용하여 그 결과를 mysql DB에 저장합니다. 이러한 작업을 작성하는데 `DataSet`에서 사용했던 API가 그대로 사용되었습니다. 

## Window

```scala
dataset.groupBy(window(col("timestampColumnName"),
windowDuration = "10 minutes", slideDuration = "5 minutes"), col("columnWeAreGroupngBy"))
```

window 설정을 위와 같은 포맷으로 지정할 수 있습니다.

[참고]

[Learning Apache Spark 3 with Scala](https://www.udemy.com/course/best-scala-apache-spark/)

