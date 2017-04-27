---
layout: post
title: 트위터 스트리밍 단어(감정) 분석
categories: [general, demo]
tags: [demo, dbyll, dbtek]
description: Twitter Streaming Sentiment Score
---

## 감정 분석
최근 트윗 정보를 가져와 3가지 파일(긍정적, 부정적, 쓸모없는 단어들)로 구성된 단어를 통해 정제를 하여
정제된(감정적인 단어가 사용된 문장) 정보를 추출 한다.
<br><br>


## AWS S3
S3를 통해 추츨한 정보를 적재한다.
- [AWS S3](http://docs.aws.amazon.com/ko_kr/AmazonS3/latest/dev/Welcome.html)
<br><br>


## 감정적 단어들
![pos-words](/image/spark/twitter/pos-words.png)
![neg-words](/image/spark/twitter/neg-words.png)
![useless-words](/image/spark/twitter/stop-words.png)


## 감정 분석 코드
{% highlight scala %}

import org.apache.spark.streaming.dstream.DStream
import org.apache.spark.streaming.twitter.TwitterUtils
import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.{SparkConf, SparkContext}

object SentimentScore extends App{

  import Utils._

  val conf = new SparkConf()
    .setAppName("spark-twitter-stream-example")
    .setMaster("local[*]")

  val sc = new SparkContext(conf)
  val ssc = new StreamingContext(sc, batchDuration = Seconds(10))

  // DStream[Status] 생성
  val tweets = TwitterUtils.createStream(ssc, None)

  // 각 파일을 읽어 broadcast 변수에 담는다.
  val uselessWords = sc.broadcast(load("/stop-words.dat"))
  val positiveWords = sc.broadcast(load("/pos-words.dat"))
  val negativeWords = sc.broadcast(load("/neg-words.dat"))

  // 튜플타입으로 text와 공백을 제거한 Sentence 생성
  val textAndSentences: DStream[(TweetText, Sentence)] =
    tweets
      .map(_.getText)
      .map(tweetText => (tweetText, wordsOf(tweetText)))

  // Sentence 데이터 정제 => .dat에 명시된 단어를 얻기 위해서
  val textAndMeaningfulSentences: DStream[(TweetText, Sentence)] =
    textAndSentences
      .mapValues(toLowercase)
      .mapValues(keepActualWords)
      .mapValues(words => keepMeaningfulWords(words, uselessWords.value))
      .filter { case (_, sentence) => sentence.length > 0 }

  // xx.dat에 명시된 positive, negative words에 대해 각 합을 구한다.
  val textAndNonNeutralScore: DStream[(TweetText, Int)] =
    textAndMeaningfulSentences
      .mapValues(sentence =>
           computeScore(sentence, positiveWords.value, negativeWords.value))
      .filter { case (_, score) => score != 0 }


  // textAndNonNeutralScore.map(makeReadable).print

  // negativeWords의 스코어(합)가 높은 순서대로 정렬을 한다.
  val tweetscores = textAndNonNeutralScore
    .map{case (tweetText, score) => (score, tweetText)}
    .transform(_.sortByKey(true))
    .map{case (score, tweetText) => (tweetText, score)}
    .map(makeReadable)


  // 출력
  tweetscores.print

  // s3에 저장
  tweetscores.saveAsTextFiles("s3n://path")

  ssc.start()

  ssc.awaitTermination()
}

{% endhighlight %}


## 유틸리티 코드
{% highlight scala %}
import twitter4j.Status

import scala.io.{AnsiColor, Source}

object Utils {

  // type alias을 통해 타입을 명시해준다. => type safe에 좋다.
  type Tweet = Status
  type TweetText = String
  type Sentence = Seq[String]

  private def format(n: Int): String = f"$n%2d"

  private def wrapScore(s: String): String = s"[ $s ] "

  private def makeReadable(n: Int): String =
    if (n > 0)      s"${AnsiColor.GREEN + format(n) + AnsiColor.RESET}"
    else if (n < 0) s"${AnsiColor.RED   + format(n) + AnsiColor.RESET}"
    else            s"${format(n)}"

  private def makeReadable(s: String): String =
    s.takeWhile(_ != '\n').take(80) + "..."

  def makeReadable(sn: (String, Int)): String =
    sn match {
      case (tweetText, score) =>
        s"${wrapScore(makeReadable(score))}${makeReadable(tweetText)}"
    }


  def load(resourcePath: String): Set[String] = {
    val source = Source.fromInputStream(getClass.getResourceAsStream(resourcePath))
    val words = source.getLines.toSet
    source.close()
    words
  }

  def wordsOf(tweet: TweetText): Sentence =
    tweet.split(" ")

  def toLowercase(sentence: Sentence): Sentence =
    sentence.map(_.toLowerCase)

  def keepActualWords(sentence: Sentence): Sentence =
    sentence.filter(_.matches("[a-z]+"))

  def extractWords(sentence: Sentence): Sentence =
    sentence.map(_.toLowerCase).filter(_.matches("[a-z]+"))

  def keepMeaningfulWords(sentence: Sentence,
    uselessWords: Set[String]): Sentence =
        sentence.filterNot(word => uselessWords.contains(word))

  def computeScore(words: Sentence, positiveWords: Set[String],
    negativeWords: Set[String]): Int =
        words.map(word =>
            computeWordScore(word, positiveWords, negativeWords)).sum

  def computeWordScore(word: String, positiveWords: Set[String],
    negativeWords: Set[String]): Int =
        if (positiveWords.contains(word)) 1
        else if (negativeWords.contains(word)) -1
        else 0

}


{% endhighlight %}
<br><br>

## 추출 결과
![print](/image/spark/twitter/print.png)
<br><br>

## S3 저장
![saveToS3](/image/spark/twitter/s3.png)
<br><br>

## 회고
`적재된 S3파일을 활용하여 dataFrame을 통해 분석을 하여 추가적인 분석을 해봐야겠다.`



