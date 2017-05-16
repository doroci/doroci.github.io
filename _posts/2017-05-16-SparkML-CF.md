---
layout: post
title: Spark ML
categories: [general, demo]
tags: [demo, dbyll, dbtek]
description: simple SparkML
---

## Spark MLlib Guide
- [Spark MLlib Programming Guide](http://spark.apache.org/docs/latest/ml-guide.html)
<br><br>

## Spark MLlib
- ML Algorithms
  - classification
  - regression
  - clustering
  - collaborative filtering
  
  
```
  Supervised Learning (지도학습)  
    - regression(회귀): 연속성
    - classification(분류): 비연속성
      
  Unsupervised Learning (비지도 학습)
    -  collaborative filtering(협업 필터링): 연속성
    -  clustering(군집합): 비연속성
```
  
- Featurization
  - feature extraction
  - transformation
  - selection
  - dimensionality reduction

```
  Extraction: raw data 추출
  Transformation: feature를 다른 featur로 변환
  Selection: a larger feature set에서 subset 선택   
  Locality Sensitive Hashing (LSH): feature 변환의 측면을 다른 알고리즘과 결합
```

- Pipelines
  - ML Pipelines을 쉽게 만들기 위한 고수준의 API 제공 (DataFrame)  
  
- Persistence
  -  saving and load algorithms, models, and Pipelines 
  
- Utilities
  - linear algebra(선형 대수), statistics(통계), data handling(데이터 처리), etc.


<br><br>


## 라이브러리 추가

{% highlight scala %}

//sbt Dependency 추가
libraryDependencies ++= Seq(
"org.apache.spark" % "spark-mllib_2.11" % "2.1.0",
)
{% endhighlight %}


## Collaborate Filtering (ALS) - DataFrame
- numBlocks 
  > 평가에 대해 병렬화 하기 위해 user-item의 분할될 블록 수 (기본값 10).
- rank 
  > 모델의 잠재 요인 갯수. 즉 user-feature 행렬과 product-feature 행렬에서 열의 갯수 k (기본값 10)
- maxIter
  > 행렬 분해를 반복하는 횟수 (기본값 to 10)
- regParam
  > ALS의 정규화 매개 변수 (기본값 to 1.0)
- implicitPrefs
  > 명시 적 피드백 ALS 변형을 사용할지 또는 암시 적 피드백 데이터에 적용 할지를 지정
    (기본값은 명시 적 피드백 사용 (파라미터 값: false))
- alpha
  > 선호도 관측치에 대한 기본 신뢰도 (기본값 1.0)
    ALS의 암시 적 피드백 변형에 적용 가능한 매개변수 
    
- nonnegative
  > 최소 제곱에 대해 음수가 아닌 제약 조건을 사용할지 여부를 지정 (기본값 false).


## Collaborate Filtering (DataFrame) 예제
{% highlight scala %}

import org.apache.spark.ml.evaluation.RegressionEvaluator
import org.apache.spark.ml.recommendation.ALS
import org.apache.spark.sql.SparkSession
import spark.implicits._


// Using DataFrame
case class Rating(userId: Int, movieId: Int, rating: Float, timestamp: Long)

def parseRating(str: String): Rating = {
    /*
        데이터 포맷 확인
        head -n 2 ${SPARK_HOME}/data/mllib/als/sample_movielens_ratings.txt
        
        0::2::3::1424380312
        0::3::1::1424380312
    */
  val fields = str.split("::")
  Rating(fields(0).toInt, fields(1).toInt, fields(2).toFloat, fields(3).toLong)
}

val ratings = spark
    .read.textFile("${SPARK_HOME}/data/mllib/als/sample_movielens_ratings.txt")
    .map(parseRating)
    .toDF()
    
val Array(training, test) = ratings.randomSplit(Array(0.8, 0.2))

// ALS알고리즘 사용 
val als = new ALS()
    .setMaxIter(5)
    // 다른 source로 부터 rating matrix가 파생된 경우 
    // setImplicitPrefs(true)을 통해 더 나은 결과를 얻을 수 있다.
    // .setImplicitPrefs(true) 
    .setRegParam(0.01)
    .setUserCol("userId")
    .setItemCol("movieId")
    .setRatingCol("rating")

val model = als.fit(training)

// test data에서 RMSE(평균 제곱근 편차)을 계산하여 모델을 평가한다.
val predictions = model.transform(test)

val evaluator = new RegressionEvaluator()
    .setMetricName("rmse")
    .setLabelCol("rating")
    .setPredictionCol("prediction")
    
val rmse = evaluator.evaluate(predictions)
println(s"Root-mean-square error = $rmse")


{% endhighlight %}


## Collaborate Filtering (RDD) 예제
{% highlight scala %}

import org.apache.spark.mllib.recommendation.ALS
import org.apache.spark.mllib.recommendation.Rating


val data = sc.textFile("${SPARK_HOME}/data/mllib/als/test.data")
val ratings = data.map(_.split(',') match {case Array(user, item, rate) =>
    Rating(user.toInt, item.toInt, rate.toDouble)
})

val rank = 10
val numIterations = 10
val model = ALS.train(ratings, rank, numIterations, 0.01)

// rating data을 통한 모델 평가 
val usersProducts = ratings.map {
case Rating(user, product, _) =>
  (user, product)
}
val predictions = model.predict(usersProducts).map {
  case Rating(user, product, rate) =>
    ((user, product), rate)
}

val ratesAndPreds = ratings.map {
case Rating(user, product, rate) =>
  ((user, product), rate)
}.join(predictions)

val MSE = ratesAndPreds.map {
case ((user, product), (r1, r2)) =>
  val err = (r1 - r2)
  err * err
}.mean()

println("Mean Squared Error = " + MSE)

// Save and load model
// model.save(sc, "/tmp/myCollaborativeFilter")
// val sameModel = MatrixFactorizationModel.load(sc, "/tmp/myCollaborativeFilter")

{% endhighlight %}

## 실행 결과 - DataFrame
평균 제곱근 편차 오차율 `1.7277440550109346`  
<br><br>

## 실행 결과 - RDD
평균 제곱근 편차 오차 = `5.193340946374788E-6`
<br><br>

## 회고
CF를 RDD로 돌리는것 보다 DataFrame을 사용하는게 평균적으로 평균 제곱근 편차 오차율이 적었다.
DataFrame이 좀 더 나은 결과를 주는것 같다.
Spark2.x부터 RDD보단 DataFrame를 사용하는 쪽으로 가이드를 주고 있으며 참고로 MLlib 3.0부터는 RDD API는 제거 된다고 한다.
<br><br>

