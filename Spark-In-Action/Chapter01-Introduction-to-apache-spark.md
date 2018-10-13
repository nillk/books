# 1장 아파치 스파크 소개
사용자는 스파크가 제공하는 API를 통해 로컬 머신에서 스칼라나 자바, 파이썬의 컬렉션을 사용하는 것처럼 스파크의 데이터 컬렉션을 사용할 수 있지만 사실은 클러스터 밑에서 여러 노드에 분산된 데이터를 참조하여 병렬로 동작

이 장에서는
* 스파크의 주요 기능과 구성하고 있는 컴포넌트 소개
* 하둡 생태계를 간략하게 살펴보고 스파크와 비교
* 간단한 스파크 예제

## 1.1 스파크란
아파치 하둡은 *HDFS*와 *맵리듀스 처리 엔진*으로 구성된다. 스파크는 범용 분산 컴퓨팅 플랫폼이라는 점에서 하둡과 유사하지만, *데이터를 메모리에 유지*함으로써 성능을 대폭 향상시켰다.

스파크의 컬렉션은 대량의 노드에 분산된 데이터를 참조할 수 있고, 사용자는 이런 환경을 알 필요 없이 로컬 프로그램을 작성하는 방식으로 분산 프로그램을 작성할 수 있다.

스파크는 빅데이터 애플리케이션에 필요한 아래와 같은 기능들을 모두 통합했다.
* 맵리듀스와 유사한 일괄처리 기능
* 실시간 데이터 처리 기능
* SQL과 유사한 정형 데이터 처리 기능
* 그래프 알고리즘
* 머신 러닝 알고리즘

하지만 스파크는 분산 아키텍처 때문에 처리 시간에 약간의 오버헤드가 필연적으로 발생하므로, 단일 머신에서도 충분히 처리할 수 있는 작은 데이터셋을 다룰 때는 다른 프레임워크를 사용하는 것이 더 효율적

### 1.1.1 스파크가 가져온 혁명
하둡은 데이터 분산 처리에서 반드시 고민해야 하는 다음 세 가지 주요 문제를 해결하여 분산 컴퓨팅을 최초로 대중화하는 데 성공했다.
* 병렬 처리 (parallelization): 전체 연산을 잘게 나누어 동시에 처리
* 데이터 분산 (distribution): 데이터를 여러 노드로 분산
* 장애 내성(fault tolerance): 분산 컴포넌트의 장애에 대응하는 방법

### 1.1.2 맵리듀스의 한계
맵리듀스는 하나의 잡이 끝나면 *결과를 HDFS에 저장*해야 다음 잡에서 데이터를 사용할 수 있고 따라서 여러 잡이 연속적으로 이어지는 반복 알고리즘에는 본질적으로 맞지 않다. 그리고 모든 유형의 문제가 맵과 리듀스 연산으로 분해되지는 않는다.

### 1.1.3 스파크가 가져다준 선물
스파크의 핵심은 데이터를 디스크에서 매번 가져오는 대신, *데이터를 메모리에 캐시로 저장하는 인-메모리 실행 모델*에 있다. 여러 단계로 이루어진 알고리즘을 맵 리듀스로 풀려면 각 알고리즘의 단계가 넘어갈 때마다 디스크에 저장하고 읽는 작업을 반복해야 한다. 하지만 스파크에서는 각 단계가 끝나면 데이터를 메모리에 캐시해두었다가 다음 단계에서 바로 사용할 수 있다. 따라서 스파크가 맵리듀스보다 성능이 좋을 것은 자명하다.

#### 1.1.3.1 사용 편의성
대표적인 word-count 예제를 맵리듀스로 구현하려면 세 개의 클래스가 필요함
* 각각 작업을 셋업하는 메인 클래스
* Mapper 클래스
* Reducer 클래스

하지만 스파크는 아래와 같은 코드면 완성
```scala
val file = spark.sparkContext.textFile("hdfs://...")
val counts = file.flatMap(line => line.split(" "))
    .map(word => (word, 1)).countByKey()
```

또한 대화형 콘솔인 스파크 셸(또는 스파크 REPL(Read-Eval-Print Loop))을 이용해 간단하게 코드를 돌려볼 수 있다.

마지막으로 스파크는 스파크 자체 클러스터(Standalone), 하둡의 YARN(Yet Another Resource Negotiator), 아파치 메소스(Mesos) 등 다양한 클러스터 매니저를 사용할 수 있다.

#### 1.1.3.2 통합 플랫폼
하둡 생태계의 여러 도구가 제공하는 아래와 같은 다양한 기능을 하나의 플랫폼으로 통합. 프로그래머와 데이터 엔지니어, 데이터 과학자 등 여러 사람이 하나의 플랫폼에서 일할 수 있도록 도와줌
* 실시간 스트림 데이터 처리
* 머신 러닝
* SQL 연산
* 그래프 알고리즘
* 일괄 처리

#### 1.1.3.3 안티패턴
스파크는 일괄 분석을 염두에 두고 설계되었으므로 공유된 데이터를 비동기적으로 갱신하는 연산에는 비적합

잡과 태스크를 시작하는 데 상당한 시간이 소모되므로 대량의 데이터 처리 작업이 아니면 굳이 사용할 필요가 없음

## 1.2 스파크를 구성하는 컴포넌트
스파크는 여러 목적에 맞는 다양한 컴포넌트로 구성된 통합 플랫폼

### 1.2.1 스파크 코어
스파크 잡과 다른 스파크 컴포넌트에 필요한 여러 기본 기능 제공
* HDFS, GlusterFS, 아마존 S3 등 다양한 파일 시스템에 접근 가능
* 공유 변수(broadcast variable)와 누적 변수(accumulator)를 사용해 컴퓨팅 노드 간의 정보 공유 가능
* 네트워킹
* 보안
* 스케줄링 및 데이터 셔플링

***RDD (Resilient Distributed Dataset)***

스파크 API의 핵심 요소로서 분산 데이터 컬렉션(데이터 셋)을 추상화한 객체. 노드에 장애가 발생해도 데이터셋을 재구성할 수 있는 복원성을 갖춤

### 1.2.2 스파크 SQL
스파크와 하이브 SQL(HiveQL)이 지원하는 SQL을 사용해 대규모 분산 정형 데이터를 다룰 수 있음. 대략적으로 아래와 같은 정형 데이터를 읽고 쓰는 데 사용할 수 있다.
* DataFrame (Spark 1.3)
* Dataset (Spark 1.6)
* Json
* Parquet
* RDB 테이블
* 하이브 테이블

그 외에도 Catalyst라는 쿼리 최적화 프레임워크를 제공하고, 외부 시스템과 스파크를 연동할 수 있는 아파치 쓰리프트 서버도 제공한다.

### 1.2.3 스파크 스트리밍
HDFS, Apache Kafka, Apache Flume, Twitter, ZeroMQ, 사용자 정의 커스텀 데이터 소스 등 여러 데이터 소스에서 유입되는 실시간 스트리밍 데이터 처리. 장애가 발생하면 연산 결과를 자동으로 복구. 가장 마지막 타임 윈도 안에 유입된 데이터를 RDD로 구성해 주기적으로 생성하는 방식

스파크 2.0 에서는 정형 스트리밍 API를 도입해 스트리밍 프로그램을 일괄 처리 프로그램처럼 구현할 수 있다.

### 1.2.4 스파크 MLlib
머신 러닝 알고리즘 라이브러리

### 1.2.5 스파크 GraphX
그래프 RDD형태의 그래프 구조를 만들 수 있는 기능 제공

## 1.3 스파크 프로그램의 실행 과정
노드 세 개로 구성된 HDFS 클러스터에 분산 저장되어 있는 300MB 크기의 로그 파일을 스파크를 이용해 분석하고 싶다고 가정

1. Spark 셸에서 로그 파일 로드  
YARN에서 스파크를 실행하며, YARN 또한 스파크와 동일한 하둡 클러스터에서 실행한다고 가정
```scala
val lines = sc.textFile("hdfs://path/to/the/file")
```
   * 로그 파일의 각 블록이 저장된 위치를 하둡에게 요청
   * 모든 블록을 **클러스터 노드의 RAM 메모리**로 전송  
작업이 완료되면 스파크 셸에서 RAM에 저장된 각 블록(partition) 참조 가능  
**Partition의 집합이 바로 RDD가 참조하는 분산 컬렉션**

2. RDD 컬렉션에 함수형 프로그래밍을 사용해 원하는 작업 실행 (Spark가 API를 제공함)  
```scala
val oomLines = lines.filter(l => l.contains("OutOfMemoryError)).cache()
```
3. cache()를 호출해 이 RDD가 메모리에 계속 유지되도록 지정
4. 최종 RDD에서 원하는 데이터 추출
```scala
val result = oomLines.count()
```

위의 모든 작업은 노드 세 개에서 병렬로 실행됨

## 1.4 스파크 생태계
하둡 생태계는 크게 인터페이스 도구, 분석 도구, 클러스터 관리 도구, 인프라 도구로 구성

스파크가 대체 가능한 도구들
* [아파치 지라프](http://giraph.apache.org/) - 스파크 GraphX
* [아파치 머하웃](https://mahout.apache.org/) - 스파크 MLlib
* [아파치 스톰](http://storm.apache.org/) - 스파크 스트리밍
* [아파치 피그](https://pig.apache.org/), [아파치 스쿱](http://sqoop.apache.org/) - 스파크 코어, 스파크 SQL  
혹은 스파크에서 피그를 실행할 수 있음

스파크가 대체하기 힘든 도구들
* [아파치 우지](http://oozie.apache.org/): 하둡 잡을 스케쥴링하는 데 사용하며 스파크 잡도 스케쥴링 가능
* [HBase](https://hbase.apache.org/): 스파크와는 완전히 다른 성격의 대규모 분산 데이터베이스
* [주키퍼](https://zookeeper.apache.org/): 분산 동기화, 네이밍, 그룹 서비스 프로비저닝 등 분산 애플리케이션에 필요한 공통 기능을 견고하게 구현한 고성능 코디네이션 서비스이며 하둡 외 다른 분산 시스템에서도 널리 활용

스파크와 공존할 수 있는 도구들
* [아파치 임팔라](https://impala.apache.org/), [아파치 드릴](https://drill.apache.org/)

스파크가 사용할 수 있는 클러스터 매니저
* [YARN](https://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/YARN.html)
* [Apache Mesos](http://mesos.apache.org/)
* 스파크 자체 클러스터