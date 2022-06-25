---
layout: post
current: post
cover: assets/built/images/scala-banner-post.png
navigation: True
title: Learning Apache Spark 3 with Scala (Section7)
date: 2022-03-06 22:30:00 +0900
tags: [scala]
class: post-template
subclass: 'post tag-scala'
author: GyuhoonK
---

Learning Apache Spark 3 with Scala (Section6 - Machine Learning Library)

# machine learning in Spark

## why to use machine learning in Spark

 머신러닝 알고리즘을 적용해야할 데이터의 양이 1대의 PC가 처리할 수 있는 사이즈보다 큰 경우, Spark를 이용하여 머신러닝 알고리즘을 수행할 수 있습니다. spark에서 머신러닝 알고리즘을 실행하면, 각 클러스터의 데이터에 해당 머신러닝 알고리즘을 적용합니다. 이는 1대의 노트북이나 데스크탑에서 수행할 수 없는 대량의 데이터에 대한 머신러닝 작업을 가능하게 합니다. 

대부분의 머신러닝 알고리즘은 많은 GPU와 메모리를 지닌 단일 머신을 작동해야하는 방식(monolithic)이었습니다. Spark는 이와는 반대로 데이터를 여러대의 머신에 수평으로 전개하여 머신러닝 알고리즘을 적용합니다.

### GPU in Spark?

클러스터의 CPU만으로 처리할 수 없는 대용량의 데이터가 존재할 수 있습니다. 이런 경우에는 각 클러스터의 GPU까지 동원하며 클러스터 내부에서도 병렬 작업을 실행할 수 있도록 해야합니다. spark 3.x부터 GPU 가속을 사용할 수 있습니다(https://www.nvidia.com/ko-kr/ai-data-science/spark-ebook/gpu-accelerated-spark-3/).

## machine learning packages in Spark



- Feature Extraction
  - TF/IDF 
- Basic statistics
  - Chi-Squared Test, Peason or Speaman corr, min, max, mean, var
- Linear Regression, Logistic Regression
- Support Vector Machine
- Naive Bayses Classifier
- Decision Trees
- K-means Clustering
- Principal Component Analysis, Singular Value Decomposition
- Recommendations using Alternating Least Squares(ALS)



# Spark1,2 vs Spark3

 spark 버전에 따라 머신러닝 라이브러리에 변화가 있습니다. Spark1, 2 버전은 	`spark.mllib`을 기본으로 사용하고 있고, Spark3 버전은  `spark.ml`을 기본으로 사용합니다. `spark.mllib`은 spark3에서도 지원하기는 하지만, 개발이 중단되었고, 일부 기능도 작동하지 않습니다(**The MLlib RDD-based API is now in maintenance mode.**).

## `spark.mllib` in Spark1,2

현재 관점에서 `spark.mllib`은 구형 API입니다. 해당 API는 RDD를 사용합니다. 또한 특정 데이터 구조를 이용해 머신러닝 알고리즘을 수행했습니다.

### 특정 데이터 구조?

특정 데이터 구조는 `Vector`를 의미합니다. 아래와 같이 분류됩니다(https://spark.apache.org/docs/latest/mllib-data-types.html).

- [Local vector](https://spark.apache.org/docs/latest/mllib-data-types.html#local-vector)
- [Labeled point](https://spark.apache.org/docs/latest/mllib-data-types.html#labeled-point)
- [Local matrix](https://spark.apache.org/docs/latest/mllib-data-types.html#local-matrix)
- Distributed matrix
  - [RowMatrix](https://spark.apache.org/docs/latest/mllib-data-types.html#rowmatrix)
  - [IndexedRowMatrix](https://spark.apache.org/docs/latest/mllib-data-types.html#indexedrowmatrix)
  - [CoordinateMatrix](https://spark.apache.org/docs/latest/mllib-data-types.html#coordinatematrix)
  - [BlockMatrix](https://spark.apache.org/docs/latest/mllib-data-types.html#blockmatrix)

```scala
import org.apache.spark.mllib.clustering.{KMeans, KMeansModel}
import org.apache.spark.mllib.linalg.Vectors

// Load and parse the data
val data = sc.textFile("data/mllib/kmeans_data.txt")
val parsedData = data.map(s => Vectors.dense(s.split(' ').map(_.toDouble))).cache()

// Cluster the data into two classes using KMeans
val numClusters = 2
val numIterations = 20
val clusters = KMeans.train(parsedData, numClusters, numIterations)
```

RDD를 데이터를 특정 Vector로 변경해주어야 머신러닝 모델에 사용할 수 있습니다.

## newer Lib(`spark.ml`) in Spark3

`spark.ml`의 가장 큰 특징은 RDD가 아니라 `DataFrame`, `Dataset`을 사용한다는 점입니다. 이를 통해 spark의 다른 컴포넌트(ex. `sparkSQL`)와 데이터를 주고 받을 수 있습니다.   즉 **상호운영성**을 확보할 수 있습니다. 예를 들어, SparkSQL로부터 학습 데이터를 추출하여 SparkML의 머신러닝 알고리즘을 실행하고, 실행 결과를 다시 SparkSQL로 처리할 수 있습니다. 이러한 동작이 가능한 이유는 모든 컴포넌트가 `DataFrame`, `Dataset`을 기본으로 채택하고 있기 때문입니다.



# Examples

아래 예시들은 [Learning Apache Spark 3 with Scala](https://www.udemy.com/course/best-scala-apache-spark/) 강좌에서 제공되는 예제 코드들을 옮겨놓았습니다.

## Recommendation

```scala
import org.apache.spark._
import org.apache.spark.ml.recommendation._

val als = new ALS()
      .setMaxIter(5)
      .setRegParam(0.01)
      .setUserCol("userID")
      .setItemCol("movieID")
      .setRatingCol("rating")
val model = als.fit(ratings)

val userID:Int = args(0).toInt
val users = Seq(userID).toDF("userID")
val recommendations = model.recommendForUserSubset(users, 10)
```

## Linear regression 

`vectorAssembler`를 이용하여 데이터를 가공한 뒤 알고리즘을 적용합니다.

```scala
import org.apache.spark._
import 
org.apache.spark.ml.feature.VectorAssembler
import org.apache.spark.ml.regression.LinearRegression

// Load Data
val regressionSchema = new StructType()
      .add("label", DoubleType, nullable = true)
      .add("features_raw", DoubleType, nullable = true)

val dsRaw = spark.read
      .option("sep", ",")
      .schema(regressionSchema)
      .csv("data/regression.txt")
      .as[RegressionSchema]

// Transform
val assembler = new VectorAssembler().
      setInputCols(Array("features_raw")).
      setOutputCol("features")
val df = assembler.transform(dsRaw)
        .select("label","features")

// Split
val trainTest = df.randomSplit(Array(0.5, 0.5))
    val trainingDF = trainTest(0)
    val testDF = trainTest(1)
    
// Now create our linear regression model
val lir = new LinearRegression()
      .setRegParam(0.3) // regularization 
      .setElasticNetParam(0.8) // elastic net mixing
      .setMaxIter(100) // max iterations
      .setTol(1E-6) // convergence tolerance
    
// Train the model using our training data
val model = lir.fit(trainingDF)
    
// Now see if we can predict values in our test data.
// Generate predictions using our linear regression model for all features in our 
// test dataframe:
val fullPredictions = model.transform(testDF).cache()
    
// This basically adds a "prediction" column to our testDF dataframe.
    
// Extract the predictions and the "known" correct labels.
val predictionAndLabel = fullPredictions.select("prediction", "label").collect()
```

## Decision Tree

```scala
import org.apache.spark.ml.feature.VectorAssembler

import org.apache.spark.ml.regression.DecisionTreeRegressor

// skip load data //

val assembler = new VectorAssembler()
   .setInputCols(Array("HouseAge", "DistanceToMRT", "NumberConvenienceStroes"))
   .setOutputCol("features")
val df = assembler.transform(dsRaw)
   .select("PriceOfUnitArea", "features")

val trainTest = df.randomSplit(Array(0.5, 0.5))
val traningDF = trainTest(0)
val testDF = trainTest(1)

val lir = new DecisionTreeRegressor()
.setFeaturesCol("features")
.setLabelCol("PriceOfUnitArea")

val model = lir.fit(traingDF)
val fullPredictions = model.transform(testDF).cache()

val predictionAndLabel = fullPredictions.select("prediction", "PriceOfUnitArea")
```

예시로 제공된 3개 모두 모델에 대한 평가를 하지 않고 있는데, `org.apache.spark.ml.evaluation`  내에 클래스를 통해 학습에 대한 평가를 확인할 수 있습니다.

```scala
import org.apache.spark.ml.evaluation.RegressionEvaluator

val evaluator = new RegressionEvaluator()
      .setMetricName("rmse")
      .setLabelCol("PriceOfUnitArea")
      .setPredictionCol("prediction")
val rmse = evaluator.evaluate(fullPredictions)
println(s"RMSE error = $rmse")
```



[참고]

[Learning Apache Spark 3 with Scala](https://www.udemy.com/course/best-scala-apache-spark/)

[Spark ML lib Gruide](https://spark.apache.org/docs/latest/ml-guide.html)

[데이터 사이언스 가이드](https://www.nvidia.com/ko-kr/ai-data-science/spark-ebook/gpu-accelerated-spark-3/)
