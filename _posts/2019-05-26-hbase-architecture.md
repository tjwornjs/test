---
title: HBase architecture
layout: post
---

MapR의 포스팅 정리  [링크](https://mapr.com/blog/in-depth-look-hbase-architecture/)

#### HBase Architectural Components
HBase는 이미지 처럼 Zookeeper, Region server(DataNode위에서 동작),  HMaster 등으로 이루어져 있다.
<img src="/assets/images/hbase_architecture/hbase component.png" alt="rdd api" width="600">{: .center-image}

#### Regions
HBase의 테이블은 row의 범위에 따라 분할되어 각 범위를 담당하는 region server에 할당되며, 이 region server는 r/w를 담당하며 1개의 region server에는 약 1000개의 region까지 할당 할수 있다.
<img src="/assets/images/hbase_architecture/region.png" alt="region" width="700">{: .center-image}

#### HBase HMaster
* region server 조정
	* region server 재시작시 로드 밸런싱을 위해 region 재 할당.
	* 주키퍼를 이용한 모든 region server 인스턴스 모니터링
* 테이블 관리
	* 테이블 생성, 삭제, 갱신을 위한 인터페이스 제공

#### ZooKeeper: The Coordinator
주키퍼를 이용해 클러스터의 상태를 유지 관리 한다. 각 리전 서버는 주키퍼로 보낸 heart beat를 이용해 각 노드의 정상 작동 여부를 한다. 
<img src="/assets/images/hbase_architecture/zookeeper.png" alt="zookeeper" width="700">{: .center-image}

#### How the Components Work Together
<img src="/assets/images/hbase_architecture/zookeeper 2.png" alt="rdd api" width="700">{: .center-image}
1. HMaster가 모니터링을 할수 있도록 각 region server은 주키퍼에 임시 노드를 생성
1. n개의 HMaster 또한 임시노드를 만들며, 주키퍼의 합의 알고리즘을 통해 선택된 1개만 활성화 된다.
1. 활성화된 HMaster는 주키퍼에 heart beat를 보내 정상 동작을 알리며, 비 활성화 된 HMaster은 주키퍼의 실패 알림을 대기 한다. 
1. Region server 또는 HMaster에서 heart beat가 오지 않으면 주키퍼의 임시노드는 삭제되고 리스너들은 해당 노드가 삭제됨을 알게 된다.
	* region server이 죽으면 HMaster에서 복구하고, HMaster이 죽게되면  비활성화된 다른 HMaster이 활성화 된다.


#### HBase First Read or Write
R/W를 하기 전 zookeeper에서 meta table을 담당하는 region server을 찾아 meta를 읽어오고(해당 데이터는 캐싱), 실제 r/w를 수행할 region server에 접속해 r/w를 수행한다.
<img src="/assets/images/hbase_architecture/meta table.png" alt="meta table" width="700">{: .center-image}

#### HBase Meta Table
<img src="/assets/images/hbase_architecture/meta table 2.png" alt="meta table2" width="700">{: .center-image}
* key : region start key, region id
* value : region server

#### Region Server Components
* Wal(Write Ahead Log) : HBase는 write 요청이 들어오면 memstore에 기록을 하고 주기적으로 hfile로 영구 저장을 한다. 영구적으로 저장을 하기 전 리전 서버가 죽어 버리면 데이터가 유실 되기 때문에 해당 리전 서버로 들어온 r/w 요청을 순차적으로 기록하여 데이터 유실에 대비 한다. 데이터를 살리는 방법은 간단하게 들어온 r/w를 순차적으로 리플레이 하는 것이다.
* BlockCache : read cache로써 자주 읽어 온 데이터를 캐싱한다. LRU 정책으로 운영됨
* MemStore : 아직 디스크에 저장되지 않은 새 데이터를 저장 임시로 저장해 쓰기 속도를 높인다. 컬럼패밀리당 하나의 memstore이 존재한다. 
* Hfiles : memstore의 데이터를 영구적으로 저장한 데이터
<img src="/assets/images/hbase_architecture/region component.png" alt="region component" width="600">{: .center-image}


#### HBase Write Steps
put 요청 -> wal 기록 -> memstore 저장
<img src="/assets/images/hbase_architecture/put.png" alt="put" width="600">{: .center-image}


#### HBase MemStore
업데이트된 key, value를 정렬된 순서로 저장하며 이 값은 hFile에 저장된 값과 같다. 각 컬럼 패밀리당 1개의 memstore이 존재 함.
<img src="/assets/images/hbase_architecture/memstore.png" alt="memstore" width="600">{: .center-image}


#### HBase Region Flush
Memstore에 충분한 데이터가 차면 HFile로 변환후 HDFS에 쓰여 진다. 컬럼패밀리당 memstore, HFile이 생기기 때문에 컬럼패밀이의 숫자에 제한이 있다.
<img src="/assets/images/hbase_architecture/region flush.png" alt="region flush" width="600">{: .center-image}

#### HBase HFile
데이터는 HFile에 정렬이 되어 저장이 되며 HDFS에 순차적으로 쓰여지기 때문에 디스크 드라이브 헤드의 이용이 없어 매우 빠르게 저장된다.
<img src="/assets/images/hbase_architecture/hfile.png" alt="hfile" width="600">{: .center-image}

#### HBase HFile Structure
HFile에는 전체 파일을 읽지 않고 데이터를 검색 할수 있는 멀티 레이어 인덱스(b+ tree)를 포함 하고 있다.
* 오름차순으로 정렬된 k/v pair
* 인덱스는 row key에 의해 64KB block안에 있는 k/v 데이터를 가르킨다.
* 각각의 block는 leaf-index를 포함한다.
* 각 block의 마지막 key는 intermediate index에 저장되어 있다.
* root index 는 intermediate index를 가르킨다.

<img src="/assets/images/hbase_architecture/hfile 2.png" alt="hfile " width="600">{: .center-image}

#### HFile Index
HFile을 로드할때 인덱스도 함께 읽어 온다. 읽혀진 인덱스는 메모리에 캐시 되며 다음 read 요청시 단일 디스크 검색으로 요청을 완료 할수 있게 한다.
<img src="/assets/images/hbase_architecture/hfile index.png" alt="hfile index" width="600">{: .center-image}

#### HBase Read Merge
1. block cache, memstore 순서로 메모리 데이터 확인.
1. 1번이 없으면 HFile에 있는 데이터 확인 
	* bloom filter, Cached index를 이용해 해당 row가 있는 데이터만 읽어 온다.
1. 데이터가 여러 HFile에 나누어져 저장되어 있으면 여러개를 다 읽어야 한다.
	* read amplification : 성능 하락
<img src="/assets/images/hbase_architecture/read.png" alt="read" width="600">{: .center-image}

#### HBase Minor Compaction
작은 HFile 여러개를 병합해 조금더 큰 HFile로 변환하는 과정. 
<img src="/assets/images/hbase_architecture/minor compaction.png" alt="minor compaction" width="600">{: .center-image}

#### HBase Major Compaction
컬럼 패밀리당 1개의 HFile로 머지하는 과정. 여러개의 HFile에 있던 삭제, 업데이트, 만료된 셀은 제거 된다. 읽기 성능 향상을 위해 수행되나 컴팩션 도중 i/o가 늘어나 성능에 영향을 끼칠수도 있다. 이러한 성능 이슈 때문에 자동수행보다는 요청이 적은 주말이나 새벽에 수행되도록 하는게 좋다.
<img src="/assets/images/hbase_architecture/major compaction.png" alt="major compaction" width="600">{: .center-image}

#### Region Split
테이블당 1개인 리전에 데이터가 쌓이게 되면 하나의 리전에서 처리하기 힘들어 지기 때문에 row를 기준으로 절반으로 나누어 진다. HMaster은 성능을 위해 해당 리전을 다른 리전 서버로 옮길수 있다.
<img src="/assets/images/hbase_architecture/region split.png" alt="region split" width="600">{: .center-image}

#### Read Load Balancing
리전 split는 최초 같은 리전 서버에서 일어나지만 로드 밸린싱 이유로 HMaster 리전을 다른 리전 서버로 옯길수 있다. 데이터는 현재 리전에 있고 그 데이터를 담당하는 리전은 다른 리전 서버에 있기 때문에 메이지 컴팩션 전까지는 원격에 있는 데이터를 서비스 해야 한다.
<img src="/assets/images/hbase_architecture/read load balancing.png" alt="read load balancing" width="600">{: .center-image}

#### HBase Crash Recovery
리전 서버에 장애가 발생하면 주키퍼에 하트 비트를 전달하지 못해 HMaster에서 장애를 감지하게 되고, 장애가 난 리전 서버의 리전을 다른 리전 서버로 할당하게 되며 유실된 memstore의 데이터를 복구 하기 위해 wal을 할당된 리전으로 분할 전송함. 새 리전을 할당받게 된 리전 서버는 wal를 리플레이 해 memstore을 재 구성한다.
<img src="/assets/images/hbase_architecture/recovery.png" alt="recovery" width="600">{: .center-image}