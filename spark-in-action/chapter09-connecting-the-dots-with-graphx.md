# 9장 점을 연결하는 GraphX

스파크의 그래프 처리 API

이번 장에선 최단 경로, 페이지 랭크, 연결 요소 및 강연결 요소 등의 알고리즘과 스파크에서 그래프 알고리즘을 사용하는 예제를 소개한다. 스칼라에서만 제공하며, 파이썬이나 자바에서는 GraphX API를 사용할 수 없다.

## 9.1 스파크의 그래프 연산

그래프는 서로 연결된 객체들을 수학적으로 표현한 개념으로 객체를 정점이라 하고 각 객체를 잇는 연결선을 간선이라 한다. 스파크의 그래프는 directed이며, 각 간선과 정점에 _속성 객체\(property object\)를 부여_할 수 있다.

이 절에서는 심슨 가족의 주인공들을 이용해 정점 네 개와 간선 네 개로 구성된 그래프로 실습한다. 정점은 각 인물이며, 간선은 각 인물들 간의 관계이다. 엄밀히 말하자면 그래프가 양방향이어야 하지만 책에서는 단순하게 하나의 방향만 표현하기로 했다. \(스파크에서 같은 정점 사이에 여러 개의 간선을 추가하는 기능 자체는 가능\)

* 정점 속성: ID, 인물 이름 및 연령 정보
* 간선 속성: 관계 유형 정보

### 9.1.1 GraphX API를 사용해 그래프 만들기

[`Graph`](http://mng.bz/078M)는 GraphX의 핵심 클래스로 정점 및 간선 객체와 다양한 그래프 변환 연산을 제공

GraphX는 정점과 간선을 별도의 RDD클래스로 구현함

* `VertexRDD`: 2-요소 tuple\(정점 ID: Long, 속성: Custom Object\)로 구성된 RDD
* `EdgeRDD`: `Edge` 객체\(출발 정점 ID: Long, 도착 정점 ID: Long, 속성: Custom Object\)로 구성된 RDD

### 9.1.1.1 그래프 구축

스파크에서는 여러 방법으로 그래프를 만들 수 있다. 여기선 2-요소 tuple RDD와 `Edge`객체로 구성된 RDD를 이용해 그래프를 초기화하는 방법을 보자.

```scala
import org.apache.spark.graphx._

// 정점 속성
case class Person(name: String, age: Int)

// 정점
val vertices = sc.parallelize(Array((1L, Person("Homer", 39)), (2L, Person("Marge", 39)), (3L, Person("Bart", 12)), (4L, Person("Milhouse", 12))))

// 간선
val edges = sc.parallelize(Array(Edge(4L, 3L, "friend"), Edge(3L, 1L, "father"), Edge(3L, 2L, "mother"), Edge(1L, 2L, "marriedTo")))

// 그래프 생성
val graph = Graph(vertices, edges)

graph.vertices.count() // 4
graph.edges.count() // 4
```

### 9.1.2 그래프 변환

* 간선과 정점을 매핑해 속성 개체에 데이터를 추가하거나 새로운 속성 값을 계산해 그래프를 반환하는 방법 설명
* 그래프 전체에 메시지를 보내고 결과를 집계하는 `aggregateMessages` 메서드 소개
* 스파크 Pregel 클래스\(대규모 그래프를 처리하는 구글 시스템\) 소개
* 그래프의 조인과 필터링 연산

GraphX는 `Graph` 클래스를 사용하지만, 이를 확장한 [`GraphOps`](http://mng.bz/J4jv) 클래스도 사용함

#### 9.1.2.1 간선 및 정점 매핑

예제의 간선 속성 객체를 문자열 대신 새로운 `Relationship` 클래스 인스턴스로 변경하려면 어떻게 해야 할까?

그래프의 간선 속성 객체와 정점 속성 객체는 각각 `mapEdges`와 `mapVertices` 메서드로 변환할 수 있다.

`mapEdges`는 파티션 ID와 파티션 내 간선의 `Iterator`를 받아 각 간선의 새 간선 속성 객체를 담은 `Iterator`를 반환하는 _간선의 속성 객체를 변환하는 매핑 함수를 인수로 전달._ \(간선 추가 X, 그래프 구조 변경 X\)

```scala
// 새 Relationship 객체
case class Relationship(relation: String)

// 간선 정보 String을 Relationship 객체로 변경
val newgraph = graph.mapEdges((partId, iter) => iter.map(edge => Relationship(edge.attr)))

newgraph.edges.collect()
// Array[org.apache.spark.graphx.Edge[Ralationship]] = Array(Edge(3,1,Relationship(father)), ...)
```

혹은 `mapEdges` 외에도 `mapTriplets`를 사용해 간선을 매핑할 수 있다. 다른 점은 매핑 함수에 `EdgeTriplet` 객체를 전달한다는 것인데, 이 객체에는 `srcId`, `dstId`, `attr` 외에도 출발 정점의 속성 객체와 도착 정점의 속성 객체를 제공한다.

이제 정점을 매핑해 기존 정점 정보에 새로운 정보를 추가해보자. `mapVertices`는 `mapEdges`와 마찬가지로 정점 ID와 정점 속성 객체를 받아서 새로운 정점 속성 객체를 반환해야 한다.

```scala
case class PersonExt(name: String, age: Int, children: Int=0, friends: Int=0, married: Boolean=false)

val newGraphExt = newgraph.mapVertices((vid, person) => PersonExt(person.name, person.age))
```

각 정점의 속성 값은 `PersonExt`로 변환되었지만, 자녀, 친구, 결혼 여부 등이 모두 기본값으로 입력되었다. 실제 속성 값을 계산하기 위해서는 `aggregateMessages` 메서드를 사용해야 한다.

#### 9.1.2.2 메시지 집계

`aggregateMessages` 메서드는 _그래프의 각 정점에서 임의의 함수를 실행하고 필요에 따라 인접 정점으로 메시지를 전송_할 수 있다. 그리고 각 정점에 도착한 메시지를 모두 집계하고 그 결과를 새로운 `VertexRDD`에 저장한다.

_**`aggregateMessages` 메서드 시그니처**_

```scala
def aggregateMessages[A: ClassTag](
    sendMsg: EdgeContext[VD, ED, A] => Unit,
    mergeMsg: (A, A) => A,
    tripletFields: TripletFields = TripletFields.All): VertexRDD[A]
```

* `sendMsg`: 그래프의 각 간선별로 전달된 `EdgeContext` 객체를 사용해 각 간선의 메시지 전송 여부 검토. 필요시 해당 간선에 연결된 인접 정점으로 메시지를 전송하는 로직 구현
  * `EdgeContext`: 출발 정점 및 도착 정점의 ID, 정점 속성 객체, 간선 속성 객체, 인접 정점에 메시지를 보낼 수 있는 두 가지 메서드\(`sendToSrc`, `sendToDst`\) 제공
* `mergeMsg`: 각 정점에 도착한 메시지를 집계하는 로직 구현
* `tripletFields`: `EdgeContext`에 어떤 데이터 필드를 전달할지 결정 \(`None`, `EdgeOnly`, `Src`, `Dst`, `All`\)

```scala
val aggVertices = newGraphExt.aggregateMessages(
    // Tuple3 = 자녀수, 친구 수, 결혼 여부
    (ctx: EdgeContext[PersonExt, Relationship, Tuple3[Int, Int, Boolean]]) => {
        if (ctx.attr.relation == "marriedTo")
            { ctx.sendToSrc((0, 0, true)); ctx.sendToDst((0, 0, true)); }
        else if (ctx.attr.relation == "mother" || ctx.attr.relation == "father")
            { ctx.sendToDst((1, 0, false)); }
        else if (ctx.attr.relation.contains("friends"))
            { ctx.sendToSrc((0, 1, false)); ctx.sendToDst((0, 1, false)); }
    },
    (msg1: Tuple3[Int, Int, Boolean], msg2: Tuple3[Int, Int, Boolean]) =>
        (msg1._1 + msg2._1, msg1._2 + msg2._2, msg1._3 || msg2._3)
)

aggVertices.collect.foreach(println)
/*
(4,(0,1,false))
(2,(1,0,true))
(1,(1,0,true))
(3,(0,1,false))
*/
```

하지만 `aggregateMessages`에서 반환된 `aggVertices`는 `Graph`가 아니라 `VertexRDD`이므로 이 속성값을 원래 그래프에 반영하려면 기존 그래프와 조인해야 한다.

#### 9.1.2.3 그래프 데이터 조인

`outerJoinVertices` 메서드를 사용해 _기존 그래프와 새로운 정점을 조인_할 수 있다.

_**`outerJoinVertices` 메서드 시그니처**_

```scala
def outerJoinVertices[U: ClassTag, VD2: ClassTag](other: RDD[(VertexId, U)])(mapFunc: (VertexId, VD, Option[U]) => VD2): Graph[VD2, ED]
```

* `other`: \(정점 ID, 정점 속성 객체\) tuple로 구성된 새로운 정점 RDD 전달
* `mapFunc`: 기존 그래프의 정점 속성 객체 `VD`, 새로 입력한 RDD의 정점 속성 객체 `U`를 결합하는 매핑 함수 전달. 만약 새 그래프에 포함되지 않으면 `Option[U]`가 `None` 반환

```scala
val graphAggr = newGraphExt.outerJoinVertices(aggVertices)(
    (vid, origPerson, optMsg) => { optMsg match {
        case Some(msg) => PersonExt(origPerson.name, origPerson.age, msg._1, msg._2, msg._3)
        case None => origPerson
    }}
)

graphAggr.vertices.collect().foreach(println)
/*
(4,PersonExt(Milhouse,12,0,1,false))
(2,PersonExt(Marge,39,1,0,true))
(1,PersonExt(Homer,39,1,0,true))
(3,PersonExt(Bart,12,0,1,false))
*/
```

**GraphX에 구현된 프리겔**

구글이 개발한 대규모 그래프 처리 시스템으로 `superstep`이라는 일종의 _반복 시퀀스_를 실행해 메시지를 전달. GraphX의 프리겔 API는 `Pregel` 클래스로 사용

_**`Pregel.apply` 메서드 시그니처**_

```scala
def apply[VD: ClassTag, ED: ClassTag, A: ClassTag]
    (graph: Graph[VD, ED],
     initialMsg: A,
     maxIterations: Int = Int.MaxValue,
     activeDirection: EdgeDirection = EdgeDirection.Either)
    (vprog: (VertexId, VD, A) => VD,
     sendMsg: EdgeTriplet[VD, ED] => Iterator[(VertexId, A)],
     mergeMsg: (A, A) => A): Graph[VD, ED]
```

* `graph`: 연산을 실행할 입력 그래프
* `initialMsg`: 첫 번째 superstep에서 모든 정점에 보낼 초기 메시지
* `maxIterations`: superstep의 최대 실행 횟수
* `activeDirection`: `sendMsg` 함수를 호출할 간선의 조건을 지정
  * `EdgeDirection.Out`: 간선의 출발 정점이 메시지를 받았을 때
  * `EdgeDirection.In`: 간선의 도착 정점이 메시지를 받았을 때
  * `EdgeDirection.Either`: 출발 정점이나 도착 정점 중 최소 한 정점이 메시지를 받았을 때
  * `EdgeDirection.Both`: 출발 정점과 도착 정점 모두 메시지를 받았을 때
* `vprog`: 각 정점별로 호출되는 정점 프로그램 함수 전달
* `sendMsg`: `EdgeTriplet`을 받고 \(정점 ID, 메시지\) 튜플의 `Iterator`를 반환하는 함수 전달. 메시지는 tuple에 지정된 정점으로 전송
* `mergeMsg`: 동일한 정점에 도착한 메시지를 병합하는 함수 전달

#### 9.1.2.4 그래프 부분 집합 선택

그래프의 일부분을 선택하는 연산은 다음 세 가지 방법으로 수행할 수 있다.

* `subgraph`: 주어진 조건\(predicate\)을 만족하는 정점과 간선 선택. 정점 및 간선 조건 함수에서 `true`를 반환하는 정점과 간선만 포함
* `mask`: 주어진 그래프에 포함된 정점만 선택. 그래프를 또 다른 그래프에 _투영해 두 그래프에 모두 존재_하는 정점과 간선만 유지 \(속성 객체는 고려하지 않음\)
* `filter`: `subgraph`와 `mask` 메서드를 조합한 메서드. _전처리 함수_를 사용해 그래프를 변환한 후, 정점 및 간선 조건 함수를 기준으로 그래프 일부 선택. 선택한 그래프를 원본 그래프의 _마스크_로 사용. 다시 말해 전처리와 `mask`를 한 번에 수행하는 메서드이다.

_**`subgraph` 메서드 시그니처**_

```scala
def subgraph(
    epred: EdgeTriplet[VD, ED] => Boolean = (x => true), // 간선 조건 함수
    vpred: (VertexId, VD) => Boolean = ((v, d) => true)) // 정점 조건 함수
    : Graph[VD, ED]
```

결과 그래프에서 _제외된 정점에 연결된 간선은 자동으로 제거_됨

```scala
// 자녀가 있는 사람만 선택
val parents = graphAggr.subgraph(
    _ => true,
    (vertexId, person) => person.children > 0)

parents.vertices.collect.foreach(println)
/*
(1,PersonExt(Homer,39,1,0,true))
(2,PersonExt(Marge,39,1,0,true))
*/
parents.edges.collect.foreach(println)
// Edge(1,2,Relationship(marriedTo))
```

## 9.2 그래프 알고리즘

사실 GraphX는 그래프 데이터 처리에 특화된 알고리즘을 사용하기 위해 개발되었다. 여기서 소개할 스파크의 그래프 알고리즘은 다음과 같다.

* 최단 거리: 한 정점에서 다른 정점들로 향하는 최단 경로 탐색
* 페이지 랭크: 정점으로 들어오고 나가는 간선 개수를 바탕으로 정점의 상대적 중요도 계산
* 연결요소: 그래프에서 서로 완전히 분리된 서브 그래프 추출
* 강연결요소: 그래프에서 서로 연결된 정점들의 군집 추출

### 9.2.1 예제 데이터셋

스탠퍼드 대학교가 공개한 위키스피디아 데이터셋 중 일부 사용

* articles.tsv: 문서의 고유한 이름이 한 줄에 하나씩 존재
* links.tsv: 각 웹 링크별 출발 문서와 도착 문서의 이름이 탭\(\t\) 문자로 구분되어 있음

```scala
val articles = sc.textFile("first-edition/ch09/articles.tsv").
    filter(line => line.trim() != "" && !line.startsWith("#")).
    zipWithIndex().cache()

val links = sc.textFile("first-edition/ch09/links.tsv").
    filter(line => line.trim() != "" && !line.startsWith("#"))

val linkIndexes = links.map(x => {
        val spl = x.split("\t")
        (spl(0), spl(1)) // source, destination
    }).join(articles).map(x => x._2).join(articles).map(x => x._2)
// (출발 문서 ID, 도착 문서 ID)

import org.apache.spark.graphx._
val wikigraph = Graph.fromEdgeTuples(linkIndexes, 0) // Graph 준비 완료
```

하지만 여기서 `wikigraph.vertices.count()`와 `articles.count()`를 해보면 두 값이 다르다. 이는 링크 파일에 일부 분서가 누락되었기 때문인데, `linkIndexes`에서 유니크한 문서 ID개수를 세어 보면 `wikigraph.vertices.count()`와 동일한 것을 확인할 수 있다.

```scala
linkIndexes.map(x => x._1).union(linkIndexes.map(x => x._2)).distinct().count()
```

### 9.2.2 최단 경로 알고리즘 \(Shortest path\)

특정 정점에서 그래프의 각 정점으로 도달하기 위해 따라가야 할 _간선의 최소 개수_를 찾는 알고리즘

스파크에서는 `ShortestPaths` 객체를 통해 최단경로 알고리즘을 사용할 수 있다. `ShortestPaths`는 유일한 `run` 메서드를 가지는데, 이 메서드는 _그래프 객체와 정점 ID의 `Seq` 객체를 인수_로 받는다. 그리고 메서드는 각각의 정점에서 인수로 받은 정점 ID로 가는 최단 경로를 `Map` 형태로 정점 속성에 저장한 그래프를 반환한다.

Rainbow\(id: 3425\)문서에서 14th\_century\(id: 10\)문서로 가는 최단 경로

```scala
import org.apache.spark.graphx.lib._
val shortest = ShortestPaths.run(wikigraph, Seq(10)) // 14th_century ID 전달

shortest.vertices.filter(x => x._1 == 3425).collect.foreach(println)
// (3425,Map(10 -> 2)) -> 10번 정점으로 가는 데 최단 거리가 2
```

### 9.2.3 페이지랭크 \(Page Rank, PR\)

그래프 내 각 정점의 중요도를 해당 정점에 도착하는 간선 개수를 기반으로 계산하는 알고리즘

1. 각 정점의 페이지랭크 값을 1로 초기화
2. 각 정점의 페이지랭크 값을 이 정점에서 나가는 간선 개수로 나눔
3. 나눈 결과를 인접 정점의 페이지랭크 값에 더함
4. 모든 페이지랭크 값의 변동폭이 수렴 허용치 값보다 작을 때까지 반복

`Graph`의 `pageRank` 메서드를 수렴 허용치 매개변수와 함께 호출해 페이지랭크 알고리즘을 실행할 수 있다.

```scala
val ranked = wikigraph.pageRank(0.001)

// 가장 중요한 페이지 열 개
val ordering = new Ordering[Tuple2[VertexId, Double]] {
    def compare(x: Tuple2[VertexId, Double], y: Tuple2[VertexId, Double]): Int = x._2.compareTo(y._2)}

val top10 = ranked.vertices.top(10)(ordering) // 정점ID와 PR값 리턴

sc.parallelize(top10).join(articles.map(_.swap)).collect.sortWith((x, y) => x._2._1 > y._2._1).foreach(println)
```

### 9.2.4 연결요소 \(Connected components\)

모든 정점에서 다른 모든 정점으로 도달할 수 있도록 연결된 서브그래프, 그래프의 모든 정점이 다른 모든 정점으로 도달할 수 있을 때는 연결 그래프\(connected graph\)라고 함

`Graph`의 `connectedComponents` 메서드\(`GraphOps`의 암시적 제공\)를 호출해 연결 요소를 찾을 수 있다. 각 정점에는 연결 요소 내 가장 작은 정점 ID가 속성으로 저장된다.

```scala
val wikiCC = wikigraph.connectedComponents()

wikiCC.vertices.map(x => (x._2, x._2)).distinct().join(articles.map(_.swap)).collect.foreach(println)
// (0,(0,%C3%81ed%C3%A1n_mac_Gabr%C3%A1in))
// (1210,(1210,Directdebit))
// wiki graph는 두 군집으로 분리되어 있음을 확인할 수 있음

wikiCC.vertices.map(x => (x._2, x._2)).countByKey().foreach(println)
// (0,4589) 0번과 연결요소 페이지 4589개
// (1210,3) 1210번과 연결요소 페이지 3개
```

### 9.2.5 강연결요소 \(Strongly Connected Componsts, SCC\)

모든 정점이 다른 정점과 연결된 서브그래프로 이 그래프 내에 속한 모든 정점은 간선 방향을 따라 _서로 도달_할 수 있어야 함. 연결요소는 완전히 분리된 서브 그래프라면 SCC는 일부 정점을 통해 서로 연결될 수 있음

`Graph`의 `stronglyConnectedComponents` 메서드\(`GraphOps`의 암시적 제공\)를 최대 반복 횟수와 함께 호출해 강연결요소를 찾을 수 있다.

```scala
val wikiSCC = wikigraph.stronglyConnectedComponents(100)

wikiSCC.vertices.map(x => x._2).distinct.count // 519 개의 SCC

wikiSCC.vertices.map(x => (x._2, x._1)).countByKey().filter(_._2 > 1).toList.sortWith((x, y) => x._2 > y._2).foreach(println)
// (6,4051) -> 6번과 강연결요소가 4051개
// (2488,6)
// (1831,3)
// (892,2)
```

## 9.3 A\* 검색 알고리즘 구현

두 정점 사이의 최단 경로를 효율적으로 찾을 수 있는 알고리즘

### 9.3.1 A\* 알고리즘 이해

시작 정점과 종료 정점을 받고, 각 정점별로 시작 정점과 종료 정점으로 이동하는 데 필요한 상대적인 비용을 계산한 후, _가장 낮은 비용을 기록한 정점들로 구성된 경로_를 선택.

지나온 거리와 목적지까지 남은 거리를 _가늠_해 합한 값으로 비용을 계산한다. 직선 거리로 계산할 수도 있지만, 다른 함수를 사용해 추정할 수도 있다. 다만 A _알고리즘의 핵심은_ 종료 정점까지 남은 거리를 추정_하는 함수이므로, 정점 간의 거리를 예상할 수 없는 그래프에서는 A_ 알고리즘을 사용할 수 없다.

> F\(비용\) = G\(지나온 경로 길이\) + H\(종료 정점과의 추정 거리\)

1. 현재 정점의 인접 미방문 정점 비용 계산 \(이미 계산되어 있는 경우 값을 비교해 더 작은 값 채택\)
2. 비용이 계산된 정점 중 가장 작은 비용의 정점을 기방문 그룹에 넣고 다음 반복 차수의 정점\(다음 차수의 현재 정점\)으로 선택
3. 종료 정점에 도달할 때까지 반복
4. 종료 정점에 도달하면 가장 작은 비용값을 _역으로_ 따라가면서 시작 정점에 이르는 경로 구성

### 9.3.2 A\* 알고리즘 구현

[A\* 알고리즘 구현 코드](https://github.com/spark-in-action/first-edition/blob/master/ch09/scala/ch09-listings.scala)

#### 9.3.2.1 알고리즘 초기화

예제 코드는 `AStart` 객체의 `run` 메서드를 호출해 실행

`run` 메서드가 받는 인수

* `graph`: A\* 알고리즘을 실행할 그래프
* `origin`: 시작 정점 ID
* `dest`: 종료 정점 ID
* `maxIterations`: 알고리즘 최대 반복 횟수
* `estimateDistance`: 두 정점의 속성 객체를 받고 둘 사이의 거리를 추정하는 함수
* `edgeWeight`: 속성 객체를 받고 간선의 가중치를 반환하는 함수
* `shouldVisitSource`: 간선의 속성 객체를 인수로 받고 간선의 출발 정점을 방문할지 여부를 지정하는 함수
* `shouldVisitDestination`: 간선의 속성 객체를 인수로 받아 간선의 도착 정점을 방문할지 여부를 지정하는 함수

```scala
val arr = graph.vertices.flatMap(n =>
    if (n._1 == origin || n._1 == dest)
        List[Tuple2[VertexId, VD]](n)
    else
        List()).collect()

if (arr.length != 2)
    throw new IllegalArgumentException("Origin or destination not found")

val origNode = if (arr(0)._1 == origin) arr(0)._2 else arr(1)._2
val destNode = if (arr(0)._1 == origin) arr(1)._2 else arr(2)._2

val dist = estimateDistance(origNode, destNode)

// 작업 그래프의 정점 속성 객체로 사용할 WorkNode 클래스
case class WorkNode(origNode: VD,
                    g: Double=Double.MaxValue,
                    h: Double=Double.MaxValue,
                    f: Double=Double.MaxValue,
                    visited: Boolean=false,
                    predec: Option[VertexId]=None)

// 원본 그래프의 정점을 WorkNode로 매핑해 gwork(작업그래프)를 생성
var gwork = graph.mapVertices{ case(ind, node) => {
    if (ind == origin)
        WorkNode(node, 0, dist, dist) // 출발지 정점의 F, G, H값 설정
    else
        WorkNode(node)
}}.cache()

// 시작 정점을 현재 정점으로 설정
var currVertexId: Option[VertexId] = Some(origin)
```

#### 9.3.2.2 메인 루프 구현

```scala
for (iter <- 0 to maxIterations
    if currVertexId.isDefined; // 종료 정점에 도달하지 못했는데 미방문 그룹에 정점이 없을 때 currVertexId에 None 할당
    if currVertexId.getOrElse(Long.MaxValue) != dest) // 종료 정점에 도달하면 currVertexId에 종료 정점 ID 할당

// 1. 현재 정점을 방문 완료로 변경
// 2. 현재 정점과 인접한 이웃 정점들의 F, G, H값 계산
// 3. 미방문 그룹에서 다음 반복 차수의 현재 정점 선정
```

> Graph의 cache 메서드를 사용해 정점과 간선을 캐싱할 수 있음

#### 9.3.2.3 현재 정점을 방문 완료로 변경

```scala
gwork = gwork.mapVertices((vid: VertexId, v: WorkNode) => {
    if (vid != currVertexId.get)
        v
    else
        WorkNode(v.origNode, v.g, v.h, v.f, true, v.predec) // 현재 정점의 visited 속성만 true로 변경
}).cache()
```

#### 9.3.2.4 작업그래프의 체크포인트 저장

체크포인팅을 하지 않으면 메인 루프를 반복할수록 DAG가 계속 늘어나서 연산량이 증가\(스택오버플로 오류 발생 가능\)하기 때문에 체크 포인트를 저장하자.

```scala
if (iter & checkpointFrequency == 0)
    gwork.checkpoint()
```

#### 9.3.2.5 이웃 정점의 비용 계산

```scala
// 현재 정점과 연결된 간선만 선택해 서브그래프 생성
val neighbors = gwork.subgraph(trip => trip.srcId == currVertexId.get || trip.dstId == currVertexId.get)

// 서브그래프의 정점 중 방문하지 않았으면서 shouldVisitSource or souldVisitDestination이 true인 정점에 메시지 전송
// 현재 정점의 G에 현재 정점부터 대상 정점까지의 가중치 값을 더해 새로운 G 계산해서 전송
val newGs = neighbors.aggregateMessages[Double](ctx => {
    if (ctx.srcId == currVertexId.get && !ctx.dstAttr.visited && shouldVisitDestination(ctx.attr)) {
        ctx.sendToDst(ctx.srcAttr.g + edgeWeight(ctx.attr))
    } else if (ctx.dstId == currVertexId.get && !ctx.srcAttr.visited && shouldVisitSource(ctx.attr)) {
        ctx.sendToSrc(ctx.dstAttr.g + edgeWeight(ctx.attr))
    }}, (a1: Double, a2: Double) => a1, TripletFields.All)
// 새 G 값을 전달받은 정점들로만 구성됨 VertexRDD

// newGs를 작업 그래프와 join
// 새로운 F 값이 기존 F 값보다 작을 때 predec 필드에 현재 정점 ID를 설정하고 정점의 속성 객체 변경
val cid = currVertexId.get
gwork = gwork.outerJoinVertices(newGs)((nid, node, totalG) =>
    totalG match {
        case None => node
        case Some(newG) => {
            if (node.h == Double.MaxValue) {
                val h = estimateDistance(node.origNode, destNode)
                WorkNode(node.origNode, newG, h, newG + h, false, Some(cid))
            } else if (node.h + newG < node.f) {
                WorkNode(node.origNode, newG, node.h, newG + node.h, false, Some(cid))
            } else
                None
        }
    }
)
```

#### 9.3.2.6 다음 반복 차수의 현재 정점 선정

```scala
// 미방문 그룹에 속한 계산된 H값을 가진 정점
val openList = gwork.vertices.filter(v => v._2.h < Double.MaxValue && !v._2.visited)

if (openList.isEmpty)
    currVertexId = None // 목적지 도달 불가
else {
    val nextV = openList.map(v => (v._1, v._2.f)).
        reduce((n1, n2) => if (n1._2 < n2._2) n1 else n2)
    currVertexId = Some(nextV._1)
}
```

#### 9.3.2.7 최종 경로 구성

```scala
// 종료 정점 ID와 `currVertexId` 값이 동일하다면 경로를 찾은 것
if (currVertexId.isDefined && currVertexId.get == dest) {
    println("Found!")

    // 종료 정점에서부터 각 정점의 predec 필드를 따라 시작 정점까지 되돌아가면서 최종 경로 구성
    var currId: Option[VertexId] = Some(dest)
    var it = lastIter
    while (currId.isDefined && is >= 0) {
        val v = gwork.vertices.filter(x => x._1 == currId.get).collect()(0)
        resbuf += v._2.origNode
        currId = v._2.predec
        it = it -1
    }
}

resbuf.toArray.reverse // 최단 경로
```

### 9.3.3 구현된 알고리즘 테스트

3차원 공간의 점을 연결한 그래프로 각 정점 속성에 X, Y, Z 좌표를 가짐

```scala
case class Point(x:Double, y:Double, z:Double)

// 예제 그래프 생성
import org.apache.spark.graphx._
val vertices3d = sc.parallelize(Array((1L, Point(1,2,4)),
    (2L, Point(6,4,4)), (3L, Point(8,5,1)),
    (4L, Point(2,2,2)), (5L, Point(2,5,8)),
    (6L, Point(3,7,4)), (7L, Point(7,9,1)),
    (8L, Point(7,1,2)), (9L, Point(8,8,10)),
    (10L, Point(10,10,2)), (11L, Point(8,4,3))))

val edges3d = sc.parallelize(Array(Edge(1, 2, 1.0), Edge(2, 3, 1.0),
    Edge(3, 4, 1.0), Edge(4, 1, 1.0),
    Edge(1, 5, 1.0), Edge(4, 5, 1.0),
    Edge(2, 8, 1.0), Edge(4, 6, 1.0),
    Edge(5, 6, 1.0), Edge(6, 7, 1.0),
    Edge(7, 2, 1.0), Edge(2, 9, 1.0),
    Edge(7, 9, 1.0), Edge(7, 10, 1.0),
    Edge(10, 11, 1.0), Edge(9, 11, 1.0)))

val graph3d = Graph(vertices3d, edges3d)

// 3차원 좌표에서 거리 계산
val calcDistance3d = (p1: Point, p2: Point) => {
    val x = p1.x - p2.x
    val y = p1.y - p2.y
    val z = p1.z - p2.z
    Math.sqrt(x * x + y * y + z * z)
}

// 그래프의 간선 매핑
val graph3dDst = graph3d.mapTriplets(t => calcDistance3d(t.srcAttr, t.dstAttr))

// checkpoint 디렉터리 지정
sc.setCheckpointDir("/home/spark/ch09checkpoint")

// 1번 정점에서 10번 정점으로 가는 최단 경로
AStar.run(graph3dDst, 1, 10, 50, calcDistance3d, (e: Double) => e)
```

