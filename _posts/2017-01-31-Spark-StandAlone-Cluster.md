---
layout: post
title: Spark StandAlone cluster 구축해보기
categories: [general, demo]
tags: [demo, dbyll, dbtek]
description: Spark StandAlone cluster
---

### 스파크 스탠드얼론 클러스터 요소
![clusterManager](/image/spark/clusterManager.png)
스파크 홈페이지에서 상세 정보들을 제공한다. [Spark Cluster 참조링크](http://spark.apache.org/docs/latest/cluster-overview.html)
### 스탠드얼론 클러스터 실행
```
1. $SPARK_HOME/bin/spark-submit를 사용하여 배포파일(.jar)을 빌드하는 방법
2. master/slave shell을 사용하여 빌드를 하고나서 spark-shell로 접속하는 방법

첫번째 방법은 작성한 코드를 한번에 돌릴수 있는 장점이 있다.
두번째 방법은 작성하고자 하는 코드를 바로 입력하여 확인할 수 있다.

이번 글에서는 두번째에 해당하는 방법으로 스탠드얼론 클러스터를 사용할 것입니다.
```
<br>

### 스파크 스탠드얼론 클러스터 설정 정보
```
http://spark.apache.org/docs/latest/spark-standalone.html
```
<br>

### 스파크 설정 파일 복사
```
1. 디렉토리: $SPARK_HOME/conf
2. 파일복사: cp spark-env.sh.template spark-env.sh
```
<br>

### 스파크 설정 파일 변경
```
1. 설정 파일열기: vim spark-env.sh
2. export SPARK_MASTER_HOST=호스트네임 (ex. localhost)
3. export SPARK_WORKER_CORES=워커당 코어수 (ex. 2)
4. export SPARK_WORKER_MEMORY=워커당 메모리용량 (ex. 1G)
5. export SPARK_WORKER_INSTANCES=워커 인스턴스 갯수 (ex. 3)

* 2~4번에 해당하는 설정을 실행 시 shell에서 입력할 수도 있다.
   - $SPARK_HOME/sbin/start-master.sh -h 호스트네임
   - $SPARK_HOME/sbin/start-slave.sh -c 코어수 -m 메모리
```
<br>

### start-master.sh 실행
```
$SPARK_HOME/sbin/start-master.sh
```
<br>

<h3>브라우저에서 웹 접속: http://hostname:port (기본 포트는 80)</h3>
![스탠드얼론_클러스터_마스터_ui](/image/spark/standAlone-master-ui.png)

### start-slave.sh 실행
```
$SPARK_HOME/sbin/start-master.sh spark://localhost:7077

* spark://localhost:7077은 실행중인 master의 URL의 정보이다.
```
<br>

<h3>웹UI에서 Workers 확인</h3>
![스탠드얼론_클러스터_마스터_ui](/image/spark/standAlone-slave-ui.png)
<br>