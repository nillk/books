# 9장 점을 연결하는 GraphX

> 스파크의 그래프 처리 API  

이번 장에선 최단 경로, 페이지 랭크, 연결 요소 및 강연결 요소 등의 알고리즘과 스파크에서 그래프 알고리즘을 사용하는 예제를 소개한다. 스칼라에서만 제공하며, 파이썬이나 자바에서는 GraphX API를 사용할 수 없다.

## 9.1 스파크의 그래프 연산

그래프는 서로 연결된 객체들을 수학적으로 표현한 개념으로 객체를 정점이라 하고 각 객체를 잇는 연결선을 간선이라 한다. 스파크의 그래프는 directed이며, 각 간선과 정점에 *속성 객체(property object)를 부여*할 수 있다.

이 절에서는 심슨 가족의 주인공들을 이용해 정점 네 개와 간선 네 개로 구성된 그래프로 실습한다. 정점은 각 인물이며, 간선은 각 인물들 간의 관계이다. 엄밀히 말하자면 그래프가 양방향이어야 하지만 책에서는 단순하게 하나의 방향만 표현하기로 했다. (스파크에서 같은 정점 사이에 여러 개의 간선을 추가하는 기능 자체는 가능)

* 정점 속성: ID, 인물 이름 및 연령 정보
* 간선 속성: 관계 유형 정보

### 9.1.1 GraphX API를 사용해 그래프 만들기

[`Graph`](http://mng.bz/078M)는 GraphX의 핵심 클래스로 정점 및 간선 객체와 다양한 그래프 변환 연산을 제공

GraphX는 정점과 간선을 별도의 RDD클래스로 구현함

* `VertexRDD`: 2-요소 tuple(정점 ID: Long, 속성: Custom Object)로 구성된 RDD
* `EdgeRDD`: `Edge` 객체(출발 정점 ID: Long, 도착 정점 ID: Long, 속성: Custom Object)로 구성된 RDD

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
* 스파크 Pregel 클래스(대규모 그래프를 처리하는 구글 시스템) 소개
* 그래프의 조인과 필터링 연산

GraphX는 `Graph` 클래스를 사용하지만, 이를 확장한 [`GraphOps`](http://mng.bz/J4jv) 클래스도 사용함

#### 9.1.2.1 간선 및 정점 매핑

예제의 간선 속성 객체를 문자열 대신 새로운 `Relationship` 클래스 인스턴스로 변경하려면 어떻게 해야 할까?

그래프의 간선 속성 객체와 정점 속성 객체는 각각 `mapEdges`와 `mapVertices` 메서드로 변환할 수 있다.

`mapEdges`는 파티션 ID와 파티션 내 간선의 `Iterator`를 받아 각 간선의 새 간선 속성 객체를 담은 `Iterator`를 반환하는 *간선의 속성 객체를 변환하는 매핑 함수를 인수로 전달.* (간선 추가 X, 그래프 구조 변경 X)

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

`aggregateMessages` 메서드는 *그래프의 각 정점에서 임의의 함수를 실행하고 필요에 따라 인접 정점으로 메시지를 전송*할 수 있다. 그리고 각 정점에 도착한 메시지를 모두 집계하고 그 결과를 새로운 `VertexRDD`에 저장한다.

***`aggregateMessages` 메서드 시그니처***

```scala
def aggregateMessages[A: ClassTag](
    sendMsg: EdgeContext[VD, ED, A] => Unit,
    mergeMsg: (A, A) => A,
    tripletFields: TripletFields = TripletFields.All): VertexRDD[A]
```

* `sendMsg`: 그래프의 각 간선별로 전달된 `EdgeContext` 객체를 사용해 각 간선의 메시지 전송 여부 검토. 필요시 해당 간선에 연결된 인접 정점으로 메시지를 전송하는 로직 구현
  * `EdgeContext`: 출발 정점 및 도착 정점의 ID, 정점 속성 객체, 간선 속성 객체, 인접 정점에 메시지를 보낼 수 있는 두 가지 메서드(`sendToSrc`, `sendToDst`) 제공
* `mergeMsg`: 각 정점에 도착한 메시지를 집계하는 로직 구현
* `tripletFields`: `EdgeContext`에 어떤 데이터 필드를 전달할지 결정 (`None`, `EdgeOnly`, `Src`, `Dst`, `All`)

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

`outerJoinVertices` 메서드를 사용해 *기존 그래프와 새로운 정점을 조인*할 수 있다.

***`outerJoinVertices` 메서드 시그니처***

```scala
def outerJoinVertices[U: ClassTag, VD2: ClassTag](other: RDD[(VertexId, U)])(mapFunc: (VertexId, VD, Option[U]) => VD2): Graph[VD2, ED]
```

* `other`: (정점 ID, 정점 속성 객체) tuple로 구성된 새로운 정점 RDD 전달
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

##### GraphX에 구현된 프리겔

구글이 개발한 대규모 그래프 처리 시스템으로 `superstep`이라는 일종의 *반복 시퀀스*를 실행해 메시지를 전달. GraphX의 프리겔 API는 `Pregel` 클래스로 사용

***`Pregel.apply` 메서드 시그니처***

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
* `sendMsg`: `EdgeTriplet`을 받고 (정점 ID, 메시지) 튜플의 `Iterator`를 반환하는 함수 전달. 메시지는 tuple에 지정된 정점으로 전송
* `mergeMsg`: 동일한 정점에 도착한 메시지를 병합하는 함수 전달

#### 9.1.2.4 그래프 부분 집합 선택

그래프의 일부분을 선택하는 연산은 다음 세 가지 방법으로 수행할 수 있다.

* `subgraph`: 주어진 조건(predicate)을 만족하는 정점과 간선 선택. 정점 및 간선 조건 함수에서 `true`를 반환하는 정점과 간선만 포함
* `mask`: 주어진 그래프에 포함된 정점만 선택. 그래프를 또 다른 그래프에 *투영해 두 그래프에 모두 존재*하는 정점과 간선만 유지 (속성 객체는 고려하지 않음)
* `filter`: `subgraph`와 `mask` 메서드를 조합한 메서드. *전처리 함수*를 사용해 그래프를 변환한 후, 정점 및 간선 조건 함수를 기준으로 그래프 일부 선택. 선택한 그래프를 원본 그래프의 *마스크*로 사용. 다시 말해 전처리와 `mask`를 한 번에 수행하는 메서드이다.

***`subgraph` 메서드 시그니처***

```scala
def subgraph(
    epred: EdgeTriplet[VD, ED] => Boolean = (x => true), // 간선 조건 함수
    vpred: (VertexId, VD) => Boolean = ((v, d) => true)) // 정점 조건 함수
    : Graph[VD, ED]
```

결과 그래프에서 *제외된 정점에 연결된 간선은 자동으로 제거*됨

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
