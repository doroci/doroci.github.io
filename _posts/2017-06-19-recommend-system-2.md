---
layout: post
title: 스파크를 사용하여 영화 추천 서비스 만들기 - 1
categories: [general, demo]
tags: [demo, dbyll, dbtek]
description: simple music recommend 
---

## 예제 및 참고자료
- [An on-line movie recommending service using Spark & Flask - Building the web service](
https://github.com/jadianes/spark-movie-lens/blob/master/notebooks/online-recommendations.ipynb)
<br><br>

## 환경구축
- aws ec2 4core/16G
  > 워커 3 구성, 워커 당 익스큐터 1개, executor(2core, 5G memory)

- pyspark
  > [pyspark Document](http://spark.apache.org/docs/2.1.0/api/python/pyspark.html)  

- flask
  > [flask Document](http://flask.pocoo.org/) 
  
  > [flask-한글 Document](http://flask-docs-kr.readthedocs.io/ko/latest/)

- cherrypy
  > [http://cherrypy.org/](http://cherrypy.org/)



### app.py
flask 모듈을 사용하여 서버로 restful API 요청하는 역할을 함

{% highlight python %}
# -*- coding: utf-8 -*-
from flask import Blueprint
from engine import RecommendationEngine
from flask import Flask, request
import logging
import json

main = Blueprint('main', __name__)

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


@main.route("/<int:user_id>/ratings/top/<int:count>", methods=["GET"])
def top_ratings(user_id, count):

    logger.debug("User %s TOP ratings requested", user_id)

    # 특정 유저에게 평점이 높은 상위 n개의 영화를 추천해준다.
    top_ratings = recommendation_engine.get_top_ratings(user_id, count)
    return json.dumps(top_ratings)


@main.route("/<int:user_id>/ratings/<int:movie_id>", methods=["GET"])
def movie_ratings(user_id, movie_id):

    logger.debug("User %s rating requested for movie %s", user_id, movie_id)

    # 특정 유저에게 특정 영화에 대한 영화 평점을 알려준다.
    ratings = recommendation_engine.get_ratings_for_movie_ids(user_id, [movie_id])

    return json.dumps(ratings)


@main.route("/<int:user_id>/ratings", methods=["POST"])
def add_ratings(user_id):

    """
    curl -d @/path/user_ratings.json -X POST http://[hostname:port]/0/ratings
    """

    # POST request 요청하여 추가적인 ratings의 정보를 가져온다.
    ratings_list = request.get_json(force='false')

    # Tuple(user_id, movie_id, rating)
    # 속성값의 타입을 변경해준다
    ratings = map(lambda x: (user_id, int(x['movie_id']), float(x['rating'])), ratings_list)

    # add ratings 호출
    recommendation_engine.add_ratings(ratings)

    return json.dumps(ratings)


def create_app(spark_context, dataset_path):

    global recommendation_engine

    recommendation_engine = RecommendationEngine(spark_context, dataset_path)

    app = Flask(__name__)
    app.register_blueprint(main)
    return app
    

    
{% endhighlight %}


### server.py 
cherrypy 모듈을 사용하여 WSGI구성 및 engine(SparkContext) 및 app(flask) 초기화를 해준다.

{% highlight python %}
# -*- coding: utf-8 -*-
import cherrypy
import os
from paste.translogger import TransLogger
from app import create_app
from pyspark import SparkContext, SparkConf


def init_spark_context():

    # SparkConf 설정
    conf = SparkConf().setAppName("movie_recommendation-server")

    # SparkContext 파이썬 모듈(engin.py, app.py]) 적용
    sc = SparkContext(conf=conf, pyFiles=['engine.py', 'app.py'])

    return sc


def run_server(app):

    # WSGI accss 로깅 설정
    app_logged = TransLogger(app)

    # 루트 디렉토리에 마운트
    cherrypy.tree.graft(app_logged, '/')

    # web server 설정
    cherrypy.config.update({
        'engine.autoreload.on': True,
        'log.screen': True,
        'server.socket_port': 5000,
        'server.socket_host': '0.0.0.0'
    })

    # web server(CherryPy WSGI) 시작
    cherrypy.engine.start()
    cherrypy.engine.block()


if __name__ == "__main__":

    sc = init_spark_context()

    # dataSet 디렉토리 로드
    dataset_path = os.path.join('/Users/lee/Desktop/datasets', 'ml-latest')

    # app 초기화
    app = create_app(sc, dataset_path)

    # app 실행
    run_server(app)

{% endhighlight %}



### engine.py 
pyspark(mllib) 모듈을 사용하여 영화 추천에 관한 로직을 수행한다. 

{% highlight python %}
# -*- coding: utf-8 -*-

import os

from pyspark import StorageLevel
from pyspark.mllib.recommendation import ALS

import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


def get_counts_and_averages(ID_and_ratings_tuple):
    """Given a tuple (movieID, ratings_iterable)
    returns (movieID, (ratings_count, ratings_avg))
    """
    nratings = len(ID_and_ratings_tuple[1])

    # Tuple(movieID, (ratings_count, ratings_avg)) 반환
    return ID_and_ratings_tuple[0], (nratings, float(sum(x for x in ID_and_ratings_tuple[1])) / nratings)


class RecommendationEngine:
    """A movie recommendation engine
    """

    def __count_and_average_ratings(self):
        """Updates the movies ratings counts from
        the current data self.ratings_RDD
        """
        logger.info("Counting movie ratings...")

        """
            union 연산된 RDD (user_id, movie_id, rating)에 대해 
            (movie_id, rating)속성을 groupByKey count한 결과와 movie_id를 update한다.
            -> Tuple(movie_id, ratings count)
        """
        movie_ID_with_ratings_RDD = self.ratings_RDD.map(lambda x: (x[1], x[2])).groupByKey()
        movie_ID_with_avg_ratings_RDD = movie_ID_with_ratings_RDD.map(get_counts_and_averages)
        self.movies_rating_counts_RDD = movie_ID_with_avg_ratings_RDD.map(lambda x: (x[0], x[1][0]))

    def __train_model(self):
        """Train the ALS model with the current dataset
        """
        logger.info("Training the ALS model...")

        self.model = ALS.train(self.ratings_RDD, self.rank, seed=self.seed,
                               iterations=self.iterations, lambda_=self.regularization_parameter)
        logger.info("ALS model built!")

    def __predict_ratings(self, user_and_movie_RDD):
        """Gets predictions for a given (userID, movieID) formatted RDD
        Returns: an RDD with format (movieTitle, movieRating, numRatings)
        """
        predicted_RDD = self.model.predictAll(user_and_movie_RDD)

        predicted_rating_RDD = predicted_RDD.map(lambda x: (x.product, x.rating))

        # join 연산을 한다.
        predicted_rating_title_and_count_RDD = \
            predicted_rating_RDD.join(self.movies_titles_RDD).join(self.movies_rating_counts_RDD)

        # Tuple(movieTitle, movieRating, numRatings)을 반환한다.
        predicted_rating_title_and_count_RDD = \
            predicted_rating_title_and_count_RDD.map(lambda r: (r[1][0][1], r[1][0][0], r[1][1]))

        return predicted_rating_title_and_count_RDD

    def add_ratings(self, ratings):
        """Add additional movie ratings in the format (user_id, movie_id, rating)
        """

        # RDD 생성
        new_ratings_RDD = self.sc.parallelize(ratings)

        # union 연산을 한다.
        self.ratings_RDD = self.ratings_RDD.union(new_ratings_RDD)

        # __count_and_average_ratings 호출
        self.__count_and_average_ratings()

        # __train_model 호출
        self.__train_model()

        return ratings

    def get_ratings_for_movie_ids(self, user_id, movie_ids):
        """Given a user_id and a list of movie_ids, predict ratings for them
        """
        requested_movies_RDD = self.sc.parallelize(movie_ids).map(lambda x: (user_id, x))

        # predicted ratings에 대해 collect
        ratings = self.__predict_ratings(requested_movies_RDD).collect()

        return ratings

    def get_top_ratings(self, user_id, movies_count):
        """Recommends up to movies_count top unrated movies to user_id
        """
        # 평가가 되지 않은 movies에 대한 user_id의 Tuple(userID, movieID)값을 가져온다.
        user_unrated_movies_RDD = self.ratings_RDD.filter(lambda rating: not rating[0] == user_id) \
            .map(lambda x: (user_id, x[0])).distinct()

        # predicted ratings 호출
        ratings = self.__predict_ratings(user_unrated_movies_RDD) \
            .filter(lambda r: r[2] >= 25).takeOrdered(movies_count, key=lambda x: -x[1])

        return ratings

    def __init__(self, sc, dataset_path):
        """Init the recommendation engine given a Spark context and a dataset path
        """

        logger.info("Starting up the Recommendation Engine: ")

        self.sc = sc

        logger.info("Loading Ratings data...")

        # ratings.csv 로드
        ratings_file_path = os.path.join(dataset_path, 'ratings.csv')
        ratings_raw_RDD = self.sc.textFile(ratings_file_path)
        ratings_raw_data_header = ratings_raw_RDD.take(1)[0]

        """
        head -n 5 ratings.csv
            userId,movieId,rating,timestamp
            1,122,2.0,945544824
            1,172,1.0,945544871
            1,1221,5.0,945544788
            1,1441,4.0,945544871
        """

        # ratings.csv을 정제 및 persist 처리
        self.ratings_RDD = ratings_raw_RDD \
            .filter(lambda line: line != ratings_raw_data_header) \
            .map(lambda line: line.split(",")) \
            .map(lambda tokens: (int(tokens[0]), int(tokens[1]), float(tokens[2])))\
            .persist(storageLevel=StorageLevel.MEMORY_ONLY)

        logger.info("Loading Movies data...")

        # movies.csv 로드
        movies_file_path = os.path.join(dataset_path, 'movies.csv')
        movies_raw_RDD = self.sc.textFile(movies_file_path)
        movies_raw_data_header = movies_raw_RDD.take(1)[0]

        """
        head -n 5 movies.csv
            movieId,title,genres
            1,Toy Story (1995),Adventure|Animation|Children|Comedy|Fantasy
            2,Jumanji (1995),Adventure|Children|Fantasy
            3,Grumpier Old Men (1995),Comedy|Romance
            4,Waiting to Exhale (1995),Comedy|Drama|Romance
        """

        # movie_raw_RDD 파싱 및 persist 처리
        self.movies_RDD = movies_raw_RDD \
            .filter(lambda line: line != movies_raw_data_header) \
            .map(lambda line: line.split(",")) \
            .map(lambda tokens: (int(tokens[0]), tokens[1], tokens[2])) \
            .persist(storageLevel=StorageLevel.MEMORY_ONLY)

        # movies_titles_RDD 파싱 및 persist 처리
        self.movies_titles_RDD = self.movies_RDD.map(lambda x: (int(x[0]), x[1])) \
            .persist(storageLevel=StorageLevel.MEMORY_ONLY)

        # count_and_average_ratings 호출
        self.__count_and_average_ratings()

        # 하이퍼 파라미터 설정 및 train_model 호출
        self.rank = 8
        self.seed = 5L
        self.iterations = 10
        self.regularization_parameter = 0.1
        self.__train_model()



{% endhighlight %}


## deploy

- 파일 정보
![deploy](/image/pyspark/deploy.png)


- virtualEnv 실행 
 > . ~/path/bin/activate

- spark clutesr 실행
![sparkEnv](/image/pyspark/sparkEnv.png)


- server.py에게 submit 
 > $SPARK_HOME/bin/spark-submit --master spark://127.0.0.1:7077 server.py

 

## Restful request

- 유저(user_id=3)에게 평점이 높은 순으로 5개의 영화 추천
 > http://host:port/3/ratings/top/5
   
   - 실행시간은 7.7초 소요
![movieRecommend](/image/pyspark/movieRecommend.png)

<br><br>

- 유저(user_id=3)에게 해당 영화의 평점(movie_id=100)을 알려준다.
 > http://host:port/3/ratings/100
 
   - 실행시간은 2.5초정 소요
![ratingsRecommend](/image/pyspark/ratingsRecommend.png)
 

- 영화(movie_id)에 대한 ratings을 추가한다.
 > curl -d @/path/user_ratings.json -X POST host:port/3/ratings
![addRatings](/image/pyspark/addRatings.png)
