---
layout: post
title: Spark StandAlone cluster 성능테스트
categories: [general, demo]
tags: [demo, dbyll, dbtek]
description: Spark StandAlone cluster Performance
---

<h3>스탠드얼론 클러스터 미 구성시</h3>
[이전글 참조 - 스탠드얼론 클러스터 구성](https://doroci.github.io/general/demo/2017/01/31/Spark-StandAlone-Cluster.html)
<br><br>

### 데이테셋 다운로드
```
https://grouplens.org/datasets/movielens
데이터셋 다운로드(저 같은 데이터셋이 큰 ml-latest.zip을 선택함)
```
![ml-latest.zip](/image/spark/dataset-240M.png)
<br><br>

### 테스트 소스
{% highlight scala %}

import org.apache.spark.sql.{DataFrame, Dataset, Row, SparkSession}
import org.apache.spark.sql.functions._

//스파크 세션 설정
val spark = SparkSession.builder
  .appName("standAlone-cluster-test")
  .getOrCreate
import spark.implicits._

// csv파일 읽기
val preview = spark
  .read
  .csv("/Users/lee/Downloads/ml-latest/ratings.csv")
preview.show

// option값 설정
val parsed = spark.read
  .option("header", "true") // 파일의 첫 줄을 필드 명으로 사용
  .option("nullValue", "?") // 필드 데이터를 변경( "?" => null )
  .option("inferSchema", "true") // 데이터 타입을 추론한다.
  .csv("/Users/lee/Downloads/ml-latest/ratings.csv")
parsed.show

parsed.count
parsed.cache
parsed
  .groupBy("movieId")
  .count
  .orderBy($"count".desc)
  .show

// createOrReplaceTempView(): 사용중인 DataFrame의 하나의 뷰를 생성한다.
val createView = parsed.createOrReplaceTempView("parks")
spark.sql("""
  SELECT movieId, COUNT(*) cnt
  FROM parks
  GROUP BY movieId
  ORDER BY cnt DESC
""").show

// describe() : numeric columns, including count, mean, stddev, min, and max의 통계를 리턴해준다.
val summary = parsed.describe()
summary.show

{% endhighlight %}

<br><br>

### 스탠드얼론 (싱글 머신)
```
1. spark-shell 실행: $SPARK_HOME/bin/spark-shell --master local
```
![ui-Executor](/image/spark/standAlone-driver.png)
<br><br>

### 스탠드얼론 클러스터 (싱글 머신)
```
1. spark-shell 실행: $SPARK_HOME/bin/spark-shell --master spark://localhost:7077
2. spark-shell 웹 UI 접속: http://localhost:4040
3. executors 카테고리 선택
```
![ui-Executor](/image/spark/Spark-StandAlone-Cluster-Ui-Executors.png)
<br><br>

### 스탠드얼론 (싱글 머신) - 결과(1core)
![standAlone-local-jobs](/image/spark/standAlone-local-jobs.png)<br><br><br>
![standAlone-local-stages](/image/spark/standAlone-local-stages.png)<br><br><br>
![standAlone-local-executors](/image/spark/standAlone-local-executors.png)<br><br><br><br>

### 스탠드얼론 (싱글 머신) - 결과(6core)
![standAlone-local-6core-jobs](/image/spark/standAlone-local-6core-jobs.png)<br><br><br>
![standAlone-local-6core-stages](/image/spark/standAlone-local-6core-stages.png)<br><br><br>
![standAlone-local-6core-executors](/image/spark/standAlone-local-6core-executors.png)<br><br><br><br>

### 스탠드얼론 클러스터 (싱글 머신) - 결과
```
특정 Stage에서 non 클러스터에 비해 클러스터로 실행한 시간이 상당히 감소했다.(jobs에 빨간색으로 찍힌 점 )
참고로 스탠드얼론 - 결과(6core)는 spark-submit으로 실행한 결과이다.
최적화를 하기 위해선 다양한 요소들에 대해 고려를 해야하는데 추후에 글을 해봐야 겠다.
```
![standAlone-cluster-jobs](/image/spark/standAlone-cluster-jobs.png)<br><br><br>
![standAlone-cluster-stages](/image/spark/standAlone-cluster-stages.png)<br><br><br>
![standAlone-cluster-executors](/image/spark/standAlone-cluster-executors.png)<br><br><br><br>
