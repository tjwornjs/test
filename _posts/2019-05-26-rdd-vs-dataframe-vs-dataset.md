---
title: RDD vs Dataframe vs Dataset
layout: post
description: spark 분석
date: '2019-05-25 18:00:00'
tags:
- spark
comments: true
---

데이터 브릭스의 포스팅 정리  [databricks](https://databricks.com/blog/2016/07/14/a-tale-of-three-apache-spark-apis-rdds-dataframes-and-datasets.html)

#### RDD
여러 노드에 분산 저장되는 변경 불가능한 데이터의 집합으로 각각의 RDD는 여러개의 파티션으로 분리 되어 있다.
Action, transformation 할수 있는 2개의 low-level api 를 제공 한다.
1. Action : RDD 값을 기반으로 결과(저장, count 등등 )를 생성해 내는 오퍼레이션
1. Transformation : 기존의 RDD를 새로운 RDD 로 변환 시키는 것 – 액션이 있기전까지는 동작 안한다.
<img src="/assets/images/rdd_dataframe_dataset/rdd.png" alt="rdd api" width="400">{: .center-image}

RDD를 사용하는 경우
* low-level의 데이터 변환 작업이 필요 할때
* unstructured data를 다룰 때 (media, text stream)

결론은 RDD가 없어 지진 않을 거지만 dataframe, dataset 또한 RDD 기반으로 만들어져 있고 간단한 api를 통해 변환이 가능하며 성능또한 좋다. 가능하면 dataset을 사용하자.


#### Dataframe
RDD와 마찬가지로 분산 불변 컬렉션이며 RDB의 테이블 처럼 row, column으로 이루어져 있다. 대용량의 데이터를 좀더 쉽게 사용할수 있도록 디자인 되어 있고, 개발자가 분산된 데이터에 구조(스키마)를 적용 할수 있어서 더 높은 수준의 추상화가 가능함. spark 2.0부턴 dataframe, dataset api가 통합 되었고 이로 인해 개발자들은 새로운 api를 학습 하지 않아도 dataset api를 사용할수 있게 되었다.
<img src="/assets/images/rdd_dataframe_dataset/dataset_dataframe.png" alt="dataframe dataset" width="400">{: .center-image}

#### Dataset
dataframe은 dataset의 한 부분으로 타입이 정해지지 않은 row로 이루어진 데이터로 생각하면 될거 같다. dataframe에 case class로 타입을 정의하게 되면 dataset.  type이 정해져 있기 때문에 type-safe api로 부르는거 같다.
이점
1. Run time type safe
  * Spark sql은 런타임까지 문법 에러를 확인하기 힘들지만 dataset, frame에서는 컴파일 타임에 에러를 확인할수 있고
  * dataset api는 모두 람다 및 jvm 객체로 표현 되기 때문에 모든 type parameter의 미스 매치는 컴파일 타임에 확인이 가능하다.
<img src="/assets/images/rdd_dataframe_dataset/error check.png" alt="error check" width="500">
1. 높은 수준의 추상화 및 custom view 제공
1. 사용하기 쉽다.
1. 성능 및 최적화 
  * dataframe, dataset는 모두 spark sql 엔진 위에서 돌아가며 catalyst를 사용하여 최적화된 query plan를 생성해 효율적으로 동작한다. 
  * spark은 노드간 데이터를 주고 받는 게 필수 인데 dataset는 데이터의 형식이 정해져 있기 때문에 tungsten 을 사용해 효율적으로 serialize/deserialize할수 있다.
<img src="/assets/images/rdd_dataframe_dataset/Performance and Optimization.png" alt="Performance and Optimization" width="500">



결론은 정말 RDD말로 방법이 없을때를 제외 하곤 dataset를 사용하자!