---
title: Apache Druid 논문 정리
layout: post
---

### Abstract
드루이드는 대규모 데이터에 대한 실시간 탐색 및 분석을 위해 설계된 데이터 저장소이며 아래와 같은 특징이 있다. [논문 링크]({{ "/assets/docs/druid.pdf" }})
1. 컬럼 지향(column-oriented storage) 
    1. 각 column별로 압축 방법을 달리해 저장 용량을 줄일 수 있다.
    1. 필요한 column만 읽어오기 때문에 빠르게 결과를 리턴할수 있다.
1. 분산 및 비공유 모델(distributed, shared-nothing architecture)
	  1.   각 노드가 서로 독립적으로 운영되기 때문에 어느 한 노드가 고장나도 전체 서비스에는 문제가 없다.
 

### Introduction
전체 적인 Big Data의 역사 및 하둡 장 단점 설명  
장점 
  * 큰 데이터 저장에 및 접근에 탁월함

단점
  * 데이터에 빠른 접근 - x
  * 동시 처리 - x
  * 입력된 데이터 즉시 처리 - x

하둡으로는 실시간 데이터 분석을 하려는 metamarket의 니즈를 충족시킬수 없었기 때문에 직접
open source, distributed, column-oriented, real-time analytical data store를 만들었다.

### Problem Definition
드루이드는 대용량의 이벤트(로그)를 저장하고 빠르게 탐색 및 분석할수 있도록 개발되었다. (OLAP 처럼)

샘플 데이터 
<img src="/assets/images/druid/druid data table.png" alt="드루이드 저장 데이터">
드루이드의 데이터는 아래와 같이 3가지로 이루어져 있다. 
  1. timestamp : 이벤트가 일어난 시간. 
  1. dimension (page, username, gender, city) 
     * 각 이벤트의 속성 값
     * 역색인을 생성해 특정 행을 빠르게 로드 할 수 있다.
  1. metric (Characters Added, Characters Removed) 
     * aggregation 연산을 수행할 컬럼

드루이드의 위와 같은 데이터를 이용해서 drill-down, aggregate하여 아래와 같은 쿼리의 결과를 **빠른 시간에** 뽑아내기 위해 만들어졌다.
```
	샌프란시스코에 거주하는 여자들이 저스틴비버의 위키 페이지를 얼마나 많이 수정했나요?
	Calgary사는 사람들이 한달동안 평균적으로 몇개의 글자를 추가했나요?
```
이러한 요구사항을 빠른시간에 뽑기 위해 RDBMS, NoSQL k/v store 등을 사용해 보았지만 대화형 응용 프로그램에 적합한  low latency,  data ingestion을 제공하지 못하여서 드루이드를 만들게 되었다.

     
#### Architecture
드루이드는 아래와 같이 서로 다른 역할을 하는 노드들로 이루어져 있으며 각 노드들간의 상호작용은 거의 없기때문에 다른 노드가 죽어도 전체 클러스터에 끼치는 영향은 미미하다.   
드루이드 노드 이외 Kafka(실시간 데이터 입력), MySQL(메타 데이터 및 Rule 저장),  Zookeeper(상태 관찰 및 전달), Deep Storage(HDFS, S3등 외부 저장소, Segment 저장)의 외부 모듈이 필요하다.

<img src="/assets/images/druid/druid architecture.png" alt="드루이드 아키텍쳐">

##### Real-time nodes
스트림 데이터를 **즉시** 저장 및 색인하여 리얼타림 쿼리를 저원하도록 하며 다른 노드들과 coordination을 위해서 zookeeper에 현재 상태를 전달한다.
인메모리 인덱스 버퍼로 들어오는 이벤트를 즉시 색인하며 쿼리 가능한 상태유지하며 메모리 오버플로우를 방지하기 위해 주기적으로 메모리 데이터를 불변 데이터로 변환해 저장한다.
  * memeory data : row oriented
  * immutable data : column oriented        

<img src="/assets/images/druid/real time node 1.png" alt="리얼타임 노드 1" width="600">
flush된 불변 데이터는 os의 off heap 영역에 로드하여 쿼리 가능한 상태를 유지한다. real time node는 주기적으로 불변데이터를 머지하여 segment라는 data block를 만들어 deep storage에 전달한다.

<img src="/assets/images/druid/real time node 2.png" alt="리얼타임 노드 2" width="600">
  1. 노드 시작 및 주키퍼에 13:00 ~ 14:00 데이터를 담당한다고 알림
  2. 13:00 ~ 14:00사이의 데이터를 매 10분마다 불변데이터로 저장
  3. 14시 넘어서(약 10분 대기) 13:00 ~ 14:00 데이터 병합 및 deep storage로 전송
	  * 데이터 입력에 딜레이가 있을수도 있기 때문에 어느정도 대기한다.
  4. 13:00 ~ 14:00 데이터는 리얼타임 노드 담당이 아님을 주키퍼에 전송

###### Availability and Scalability
실시간 노드와 데이터 생성자 사이 data durability를 위해 카프카와 같은 메세지 버스를 둔다
  * 버퍼로 사용 리얼타임 노드가 도중에 실패하면 저장된 인덱스부터 다시 데이터를 읽어오면 된다. - fail over
  * 여러 노드가 동일한 카프카에서 이벤트를 읽어오면 한 노드가 완전 실패하였을때에도 다른 노드의 데이터를 사용하면 된다. - replication

##### Historical Nodes
real-time node에서 만든 세그먼트를 로드 및 서빙
<img src="/assets/images/druid/historical node.png" alt="historical node" width="500">
shared nothing architecture로 노드들 사이에 contention이 없으며,  단순 load, drop, serve만 담당하며 주키퍼를 이용해 상태를 관리한다.  불변 데이터를 다루기 때문에 read consistency를 지원함 
###### Tiers
우선순위
  * ex)자주 읽히는 데이터는 hot tier, 그 외 에는 cold tier
  * hadoop의 queue정도로 생각하면 될 듯

###### Availability
주키퍼를 통해 어떤 세그먼트를 서빙할지 정해진다. 주키퍼가 정상 작동을 하지 않으면 새로운 데이터를 서빙할수 없지만 기존 데이터는 계속 서빙 가능함

##### Broker Nodes
쿼리를 받아 real time, historical node에 전달하는 query router 역활을 하며,  zookeeper을 통해 각 노드에 할당된 세그먼트를 알고 있다. 입력 쿼리에 해당하는 세그먼트를 가진 노드에 쿼리를 요청한고  real time, historical node의 결과를 합쳐 결과를 생성함
###### Caching
JVM 메모리나 외부 저장소(memcached, redis)에 LRU 캐시를 가지고 결과를 캐싱한다.
캐시에 없는 쿼리 결과는 각 노드별로 쿼리를 요청하게 되고 **historical node의 결과만** 세그먼트 단위로 캐싱함
<img src="/assets/images/druid/caching.png" alt="캐싱.png" width="700">

###### Availability
주키퍼가 모두 죽은 상태에서도 마지막 노드의 상태 정보를 알고 있기때문에 쿼리는 가능하다.
딱히 별 의미 없음

##### Coordinator Nodes
historical node의 분산 코디네이터. 주키퍼를 통해 historical node에 load, drop 명령 및 복제 이동등의 명령을 전달함
주키퍼를 이용해 리더를 선출하고 나머지 노드들은 백업 노드로 동작함

###### Rules
historical node가 세그먼트를 로드, 드랍하는 방법 정의 - MySql에 저장되어 있다  
  * 어떤 tier에 할당되는가?
  * 몇개의 복제본을 가지고 있는가?
  * 언제 드랍되는가? 

###### Load Balancing  
Historical Node의 리소스는 무한하지 않기때문에 세그먼트들은 반드시 각 클러스터에 나누어서 로딩되어야 하며 
최적의 세그먼트 분포를 결정하기 위해선 쿼리 패턴을 분석해야 한다.
  * 주로 최근 세그먼트를 자주 참조한다 - 최근 데이터의 tier 및 replication factor을 높인다.    

이러한 일을 잘하기 위해서 cost-based optimization 라는 알고리즘을 개발함
###### Replication
서로다른 historical node에서 같은 세그먼트의 데이터를 로드해 서빙함 - 데이터는 불변, 다른노드에서 돌려도 결과는 같다.   롤링 업데이트에 사용했다고 함 

##### Storage Format
segment : 일정 기간에 걸쳐진 데이터의 row 모음으로 정의되며 주로 5~10 million rows로 이루어져 있다. - 저장 복제되는 기본단위     
주로 1y, 1d, 1h 등으로 파티셔닝 가능하며 어떤 값으로도 파티셔닝이 가능하다.     
컬럼 지향형으로 저장이 됨
  * 효율적인 cpu usage : 필요한 컬럼 만 읽으면 된다. 
  * 여러 타입의 컬럼을 지원하며 각 컬럼별로 다른 압축 방법을 선택 할수 있다.
    * 샘플 데이터에 나온 표 중 Gender같은 경우 Male -> 0, Female -> 1로 매핑을 하면  실제 데이터보다 더 적은 용량을 차지하며 저장 할수 있다.


###### Indices for Filtering Data
대부분의 OLAP 쿼리는 dimension 필터링한 metric의 aggregate다. 특정 쿼리 필터와 관련된 행만 검색이 가능하도록 추가 인덱스를 생성해 read data 양을 줄여 속도를 향상 시킨다.
  * dimension에 나온 값이 각각 어느 row에 존재하는지 인덱싱 해둔다음 쿼리의 조건에 맞는 row만 읽으면 된다.   
  * 인덱스는 bit set이며 Concise Algorithm를 사용해 압축함 [link](http://ricerca.mat.uniroma3.it/users/dipietro/publications/0020-0190.pdf)