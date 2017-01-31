---
layout: post
title: Intellij Spark 설치 이슈 in sbt
categories: [general, demo]
tags: [demo, dbyll, dbtek]
description: Intellij Spark 설치 이슈 in sbt
---

Intellij에서 sbt로 spark 2.0.2버전을 설치 하려고 했는데 디펜던시 에러가 나면서 봤더니 스칼라 관련하여 문제가 생긴 것이였다. (기존 1.6.0 버전을 설치했을땐 이런 이슈가 없었는데...)

```
SBT project import
[warn] Multiple dependencies with the same organization/name but different versions.
To avoid conflict, pick one version:
[warn]  * org.scala-lang:scala-compiler:(2.11.0, 2.11.7)
[warn]  * org.scala-lang:scala-library:(2.11.8, 2.11.7)
[warn]  * org.scala-lang.modules:scala-parser-combinators_2.11:(1.0.1, 1.0.4)
[warn]  * org.scala-lang.modules:scala-xml_2.11:(1.0.2, 1.0.4)
```
그래서 구글 검색을 해보니 원하지 않는 디펜던시를 제거를 제거 하면 된다고 한다. 그래서 아래와 같은 디펜던시를 추가를 하고 Rebuild를 하였더니 에러가 사라졌다. (2.0 설치를 할때 이러한 이슈가 생기면 참고 하시면 좋을 듯하다.)

```
  "org.scala-lang" % "scala-compiler" % "2.11.7",
  "org.scala-lang.modules" % "scala-parser-combinators_2.11" % "1.0.4",
  "org.scala-lang" % "scala-library" % "2.11.7",
  "org.scala-lang.modules" % "scala-xml_2.11" % "1.0.4"
```

