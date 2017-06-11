---
layout: post
title: 스파크를 사용하여 간단한 음악 추천 하기- 1
categories: [general, demo]
tags: [demo, dbyll, dbtek]
description: simple music recommend 
---

## 예제 및 참고자료
- [MLlib Collaborative Filtering](https://doroci.github.io/general/demo/2017/05/16/SparkML-CF.html)
- [소스코드](https://github.com/sryza/aas)
- [9가지 사례로 익히는 고급 스파크 분석](http://www.kyobobook.co.kr/product/detailViewKor.laf?ejkGb=KOR&mallGb=KOR&barcode=9788968482892&orderClick=LEA&Kc=)
<br><br>

## RDD 기반의 API 사용

> import 및 dataSet 확인
![dataSet](/image/spark/advancedWithSpark/importAndDataSet.png)

> buildArtistByID 함수
![buildArtistByID](/image/spark/advancedWithSpark/buildArtistByID.png)

> buildArtistAlias 함수
![buildArtistAlias](/image/spark/advancedWithSpark/buildArtistAlias.png)

> preparation 함수
![preparation](/image/spark/advancedWithSpark/preparation.png)

> preparation 실행결과
![exec_preparation](/image/spark/advancedWithSpark/exec_preparation.png)

> buildRatings 함수
![buildRatings](/image/spark/advancedWithSpark/buildRatings.png)

> unpersist 함수
![unpersist](/image/spark/advancedWithSpark/unpersist.png)

> model1 함수
![model1](/image/spark/advancedWithSpark/model1.png)

> model2 함수
![model2](/image/spark/advancedWithSpark/model2.png)

> model 실행결과
![exec_model](/image/spark/advancedWithSpark/exec_model.png)

### ROC
`ROC에서 y=x그래프는 Random Guess를 의미하며 y값이 커질수록 better값을 y값이 작아질수록 worst값을 의미한다.`
`평면에서의 한 점의 좌표 x에 대해 (0,0) 일수록 보수적이며 (1,1)에 가까울수록 모험적인 의미를 나타낸다.`
  - [ROC - Receiver operating characteristics](https://en.wikipedia.org/wiki/Receiver_operating_characteristic)

### AUC
`AUC은 곡선 아래 영역라는 의미이다.`
 - [AUC - Area under the curve](https://en.wikipedia.org/wiki/Area_under_the_curve_(pharmacokinetics))

> AUC1 함수
![auc1](/image/spark/advancedWithSpark/auc1.png)

> AUC2 함수
![auc2](/image/spark/advancedWithSpark/auc2.png)

> AUC3 함수
![auc3](/image/spark/advancedWithSpark/auc3.png)

> predictMostListened 함수
![predictMostListened](/image/spark/advancedWithSpark/predictMostListened.png)

> evaluate 함수 
![evaluate](/image/spark/advancedWithSpark/evaluate.png)

> evaluate 실행결과
![exec_evaluate](/image/spark/advancedWithSpark/exec_evaluate.png)
```
위의 결과에서 rank는 50, lambda는 1.0, alpha를 40으로 하였을때가 가장 추천 결과가 좋았다. 
이와 같이 하이퍼 파라미터를 변경하여 추천 모델을 튜닝을 할 수 있다.
추가적으로 과도한 하이퍼 파라미터 설정은 오히려 성능 저하를 가져올 수 있으니
적절하게 사용하는게 좋다.
```


> recommend 함수
![recommend](/image/spark/advancedWithSpark/recommend.png)

> recommend 실행결과
![exec_recommend](/image/spark/advancedWithSpark/exec_recommend.png)
```
사용자ID(2093760)에게 5가지의 아티스트를 추천해주었는데 그중에 [unknown]의 데이터가 속해있다.
해당 데이터는 유용한 데이터가 아니므로 제거 후 다시 위의 과정을 실행한다.
데이터 분석시 이러한 경우는 비일비재 하며 이런 과정을 통해 가치 없는 데이터를 필터링을 하는게 중요하다.
```



