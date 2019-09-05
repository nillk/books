# 4장 스파크 API 깊이 파헤치기

* Spark Core API
* Pair RDD \(Key-Value pair\)
* 스파크의 데이터 파티셔닝 + 셔플링
* 데이터의 그루핑, 정렬, 조인
* 누적 변수, 공유 변수 \(잡 실행 도중 스파크 실행자간의 데이터 공유 방법\)
* RDD 의존 관계와 스파크의 내부 동작 원리

## 4.1 Pair RDD 다루기

간단하고 확장성이 뛰어난 키-값 쌍 모델은 오늘날 캐싱 시스템이나 NoSQL등에 많이 사용되며, 하둡의 맵리듀스 또한 키-값 쌍을 기반으로 동작한다. 키-값 쌍은 전통적으로 연관 배열\(Associative array\)이라는 자료구조를 사용해 표현한다.

스파크에서는 키-값 쌍\(or 키-값 튜플\)로 구성된 RDD를 **Pair RDD**라고 한다. 일반 RDD를 사용해도 되지만 Pair RDD를 사용했을 때 바람직한 경우가 있다.

### 4.1.1 Pair RDD 생성

스파크에서는 다양한 방법으로 _Pair RDD_를 생성할 수 있다. 어떤 방법이든 2-요소 튜플로 RDD를 구성하면 암시적 변환을 통해 `PairRDDFunctions` 클래스에 정의되어 있는 _Pair RDD_ 함수를 사용할 수 있다.

### 4.1.2 기본 Pair RDD 함수

한 쇼핑 사이트에서 특정 규칙에 따라 사은품을 추가하는 프로그램을 개발해보자. 사은품은 구매 금액이 0.00달러인 추가 거래로 기입해야 한다.

* 데이터 셋  

  [https://github.com/spark-in-action/first-edition/tree/master/ch04/ch04\_data\_transactions.txt](https://github.com/spark-in-action/first-edition/tree/master/ch04/ch04_data_transactions.txt)

* 파일 포맷  

  구매날짜\#시간\#고객ID\#상품ID\#구매수량\#구매금액

```scala
scala> val tranFile = sc.textFile("first-edition/ch04" + "ch04_data_transactions.txt")
tranFile: org.apache.spark.rdd.RDD[String]
scala> val tranData = tranFile.map(_.split("#"))
tranData: org.apache.spark.rdd.RDD[Array[String]]
scala> var transByCust = tranData.map(tran => (tran(2).toInt, tran))
transByCust: org.apache.spark.rdd.RDD[(Int, Array[String])]
```

#### 4.1.2.1 키 및 값 가져오기

_Pair RDD_의 키 또는 값으로 구성된 새로운 RDD를 가져오려면 `keys` 또는 `values` 변환 연산자를 사용한다. 혹은 튜플이기 때문에 `map(_._1)` 또는 `map(_._2)` 를 사용할 수도 있다.

```scala
scala> transByCust.keys.count()
res0: Long = 1000
scala> transByCust.keys.distinct().count()
res1: Long = 100
```

#### 4.1.2.2 키 별 개수 세기

데이터 파일에서 한 줄이 구매 한 건이기 때문에 각 고객 별로 줄 개수가 곧 각 고객의 구매 횟수이다. 이럴 때 `countByKey` _행동 연산자_를 사용할 수 있다.

```scala
scala> transByCust.countByKey()
res2: scala.collection.Map[Int,Long] = Map(69 -> 7, 88 -> 5, 5 -> 11, 10 -> 7, 56 -> 17, 42 -> 7, ...
scala> transByCust.countByKey().values.sum // values 와 sum 은 스칼라 표준 메서드
res3: Long = 1000
scala> val (cid, purch) = transByCust.countByKey().toSeq.sortBy(_._2).last
cid: Int = 53
purch: Long = 19
```

`countByKey`의 결과를 모두 더하면 전체 구매 횟수인 1000이 나온다는 것을 확인할 수 있으며, 결과를 구매 횟수로 정렬해 구매 횟수가 가장 많은 고객의 아디가 53이며 총 구매 횟수는 19번인 것을 확인할 수 있다. 이 고객에게 곰 인형\(ID 4\)를 사은품으로 발송하면 된다.

```scala
scala> var complTrans = Array(Array("2015-03-30", "11:59 PM", "53", "4", "1", "0.00"))
```

#### 4.1.2.3 단일 키로 값 찾기

사은품을 받을 고개이 어떤 상품을 구매했는지도 알아야 한다. `PairRDDFunctions`의 `lookup` 행동 연산자를 사용해 모든 구매기록을 가져올 수 있다. 다만 `lookup` 연산자는 _결과 값을 드라이버로 전송_하므로 이를 메모리에 적재할 수 있는지 확인해야 한다.

_**lookup 메서드 시그니처**_

```scala
def lookup(key: K): DataFrame
// Return the list of values in the RDD for key key. This operation is done efficiently if the RDD has a known partitioner by only searching the partition that the key maps to.
```

53번 구매 고객의 구매 내역

```scala
scala> transByCust.lookup(53)
res4: Seq[Array[String]] = WrappedArray(Array(2015-03-30, 6:18 AM, 53, 42, 5, 2197.85), Array(2015-03-30, 4:42 AM, 53, 44, 6, 9182.08), Array(2015-03-30, 2:51 AM, 53, 59, 5, 3154.43), Array(2015-03-30, ...
scala> transBycust.lookup(53).foreach(tran => println(tran.mkString(", ")))
```

#### 4.1.2.4 `mapValues` 변환 연산자로 Pair RDD 값 바꾸기

`mapValues` 변환 연산자를 사용하면 _키를 변경하지 않고 값만 변경_할 수 있다.

바비 쇼핑몰 놀이 세트\(ID 25\)를 두 개 이상 구매하면 청구 금액을 5% 할인해보자.

```scala
scala> transByCust = transByCust.mapValues(tran => {
    if (tran(3).toInt == 25 && tran(4).toDouble > 1) {
        tran(5) = (tran(5).toDouble * 0.95).toString
    }
    tran
})
```

#### 4.1.2.4 `flatMapValues` 변환 연산자로 키에 값 추가

사전\(ID 81\)을 다섯 권 이상 구매한 고객에게 사은품으로 칫솔\(ID 70\)을 보내야 한다. 사은품은 구매 금액이 0.00인 추가거래로 기입해야하므로, `transByCust`에 배열 형태의 새로운 구매 기록을 추가해야 한다는 말이다.

여기에는 `flatMapValues` 변환연산자를 사용할 수 있다. 이 연산자는 각 키 값을 _0개 또는 한 개 이상 값으로 매핑해 RDD에 포함된 요소 개수를 변경_할 수 있다. `flatMapValues`는 변환 함수를 전달 받아 변환한 컬렉션 값들을 원래 키와 합쳐 새로운 키-값 쌍으로 생성한다. 만약 새로운 컬렉션이 빈 컬렉션이라면 해당 키-값 쌍을 결과에서 제외한다.

_**flatMapValues 메서드 시그니처**_

```scala
def
flatMapValues[U](f: (V) ⇒ TraversableOnce[U]): RDD[(K, U)]
// Pass each value in the key-value pair RDD through a flatMap function without changing the keys; this also retains the original RDD's partitioning.
```

```scala
scala> transByCust = transByCust.flatMapValues(tran => {
    if (tran(3).toInt == 81 && tran(4).toDouble >= 5) {
        val cloned = tran.clone()
        cloned(5) = "0.00"
        cloned(3) = "70"
        cloned(4) = "1"
        List(tran, cloned)
    } else {
        List(tran)
    }
})
```

#### 4.1.2.6 `reduceByKey` 변환 연산자로 키의 모든 값 병합

`reduceByKey`는 _각 키의 모든 값을 동일한 타입의 단일 값으로 병합_한다. 따라서 결합 법칙을 만족하는 두 값을 하나로 병합할 `merge` 함수를 전달해야 한다.

#### 4.1.2.7 `reduceByKey` 대신 `foldByKey` 사용

`foldByKey`는 `reduceByKey`와 기능은 같지만, `zeroValue`인자를 받는다. `zeroValue`는 반드시 항등원이어야 한다.

_**foldByKey 메서드 시그니처**_

```scala
def foldByKey(zeroValue: V)(func: (V, V) => V): RDD[(K, V)]
```

여기서 하나 주의할 점이 있는데, 스칼라의 `foldLeft` 나 `foldRight` 메서드와는 달리 스파크에서는 RDD연산이 병렬로 실행되므로 `zeroValue`가 여러번 쓰일 수도 있다는 점이다.

이제 가장 많은 금액을 지출한 고객을 찾아보자.

```scala
scala> val amounts = transByCust.mapValues(t => t(5).toDouble)
scala> val totals = amounts.foldByKey(0)((p1, p2) => p1 + p2).collect()
res5: Array[(Int, Double)] = Array((84,53020.619999999995), (39,49842.78), (96,36928.57), (81,40219.15), (15,55853.280000000006), (21,62274.25), (66,52130.009999999995), (54,36307.04), (48,17949.850000000002), (30,19194.91), (27,57023.96000000001), ...
scala> totals.toSeq.sortBy(_._2).last
res6: (Int, Double) = (76,100049.0)
```

76번 고객이 10만 49달러로 가장 많은 금액을 지출했음을 확인할 수 있다.

`zeroValue`가 여러번 쓰이는 상황을 고려하지 않은 경우 아래와 같이 이상한 결과를 얻게 된다. 원하던 값이 아니라 RDD의 파티션 개수만큼 `zeroValue`가 더해져있다.

```scala
scala> amounts.foldByKey(100000)((p1, p2) => p1 + p2).collect()
res6: Array[(Int, Double)] = Array((84,253020.62), (39,249842.77999999997), (96,236928.57), (81,240219.15000000002), (15,255853.28), (21,262274.25), (66,252130.00999999998), (54,236307.04), (48,217949.85), (30,219194.91), (27,257023.96),
```

이제 ID 76번 고객에게 커플 잠옷 세트\(ID 63\)을 사은품으로 보내기 위한 기록을 추가하자. 전에 임시로 만든 `complTrans`에 새로운 사은품 구매 기록을 추가하고, 각각의 구매 기록을 고객 ID를 키로 설정해 `transByCust`에 추가하고 파일에 저장하자.

```scala
scala> complTrans = complTrans :+ Array("2015-03-30", "11:59 PM", "76", "63", "1", "0.00")
scala> transByCust = transByCust.union(sc.parallelize(complTrans).map(t => t(2).toInt, t)))
scala> transByCust.map(t => t._2.mkString("#")).saveAsTextFile("ch04output-transByCust")
```

#### 4.1.2.8 `aggregateByKey`로 키의 모든 값 그루핑

`aggregateByKey`는 `zeroValue`를 받아 RDD 값을 병합한다는 점에서 `foldByKey`나 `reduceByKey` 와 비슷하지만 _값 타입을 바꿀 수 있다._ 따라서 `aggregateByKey`는 `zeroValue`와 함께 _변환 함수_와 _병합 함수_를 받는다. 여기서 인자를 여러 목록으로 받는데 이를 [커링](https://www.scala-lang.org/old/node/135)이라고 한다. `aggregateByKey` 연산자에 `zeroValue`만 넣으면 `seqOp`와 `combOp`를 받는 새로운 함수가 반환된다.

_**aggregateByKey 메서드 시그니처**_

```scala
def
aggregateByKey[U](zeroValue: U)(seqOp: (U, V) ⇒ U, combOp: (U, U) ⇒ U)(implicit arg0: ClassTag[U]): RDD[(K, U)]
// Aggregate the values of each key, using given combine functions and a neutral "zero value". This function can return a different result type, U, than the type of the values in this RDD, V. Thus, we need one operation for merging a V into a U and one operation for merging two U's, as in scala.TraversableOnce. The former operation is used for merging values within a partition, and the latter is used for merging values between partitions. To avoid memory allocation, both of these functions are allowed to modify and return their first argument instead of creating a new U.
```

각 고객이 구매한 제품의 전체 목록을 가져오는 예제. `:::` 는 두 리스트를 이어 붙이는 연산자이다.

```scala
scala> val prods = transByCust.aggregateByKey(List[String]())((prods, tran) => prods ::: List(tran(3)), (prods1, prods2) => prods1 ::: prods2)
scala> prods.collect()
res7: Array[(Int, List[String])] = Array((84,List(55, 10, 14, 62, 79, 59, 92, 81, 50)), (39,List(28, 92, 87, 58, 4, 28, 11, 56, 5, 4, 50)), (96,List(37, 31, 17, 88, 97, 81, 58, 78)), (81,List(44, 6, 29, 82, 12, 69, 58, 91, 36)), (15,List(13, 24, 16, 4, 48, 44, 59, 39, 4, 57)), (21,List(44, 46, 21, 28, 7, 44, 89, 38, 27, 69, 84, 64, 59)), (66,List(38, 78, 17, 29, 62, 44, 55, 19, 62, 86, 58)), (54,List(78, 57, 51, 4, 98, 4 ...
```

## 4.2 데이터 파티셔닝을 이해하고 데이터 셔플링 최소화

* 데이터 파티셔닝: 데이터를 여러 클러스터 노드로 분할하는 메커니즘
* RDD의 파티션: RDD 데이터의 일부\(조각 또는 슬라이스\)

로컬 파일 시스템에 저장된 텍스트 파일을 스파크에 로드하면, 스파크는 파일 내용을 여러 파티션으로 분할해 클러스터 노드에 고르게 분산 저장하며, 이렇게 분산된 파티션이 모여서 RDD 하나를 형성한다. 스파크는 RDD의 파티션 목록을 보관하며, 각 파티션의 데이터를 처리할 최적 위치를 추가로 저장할 수 있다.

RDD의 파티션 목록은 `partitions` 필드로 확인할 수 있다.

스파크에서는 RDD 파티션 개수가 매우 중요하다. 개수가 너무 적으면 클러스터를 충분히 활용할 수 없고 각 태스크가 처리할 데이터가 Executor의 메모리를 초과하면 문제가 발생한다. 그렇다고 너무 많으면 태스크 관리 작업에 병목 현상이 발생하므로 일반적으로 클러스터의 코어 개수보다 서너 배 정도의 파티션을 사용하는 것이 좋다.

### 4.2.1 스파크의 데이터 Partitioner

스파크는 데이터 파티셔닝을 `Partitioner` 객체가 _RDD의 각 요소에 파티션 번호를 할당_하는 방식으로 수행한다. 스파크가 기본적으로 제공하는 구현체는 `HashPartitioner`와 `RangePartitioner`가 있으며 사용자 정의 `Partitioner`를 사용할 수도 있다.

#### 4.2.1.1 HashPartitioner

스파크의 기본 `Partitioner`. 각 요소의 자바 hash코드\(Pair RDD는 Key의 hash\)를 단순한 mod 공식 \(`partitionIndex = hashCode % numberOfPartitions`\)에 대입해 파티션 번호를 계산. 데이터 파티션의 기본 개수는 스파크의 `spark.default.parallelism` 매개 변수 값으로 결정된다. 이 값이 결정되지 않으면 스파크는 클러스터의 코어 개수를 대신 사용한다.

#### 4.2.1.2 RangePartitioner

정렬된 RDD 데이터를 거의 같은 범위 간격으로 분할할 수 있다. `RangePartitioner`는 대상 RDD에서 샘플링한 데이터를 기반으로 범위 경계를 결정한다.

#### 4.2.1.3 Pair RDD의 사용자 정의 Partitioner

사용자 정의 `Partitioner`는 Pair RDD에만 사용할 수 있다. `mapValues`와 `flatMapValues`는 항상 파티셔닝을 보존하므로 이 두 메서드를 제외한 Pair RDD _변환 연산자_는 `Int`인수를 받거나 `Partitioner`를 받는 아래 두 가지 종류의 추가 메서드를 제공한다.

1. `Int` 인수 \(변경할 파티션 개수\) - `HashPartitioner` 사용

   ```scala
    rdd.foldByKey(afunction, 100)
   ```

2. 사용할 `Partitioner` \(스파크 지원 Partitioner or 사용자 정의 Partitioner\)

   ```scala
    rdd.foldByKey(afunction, new HashPartitioner(100))
   ```

만약 Pair RDD 변환 연산자를 호출할 때 `Partitioner`를 지정하지 않으면 _부모 RDD\(현재 RDD를 만드는 데 사용한 RDD들\)에 지정된 파티션 개수 중 가장 큰 수를 사용_한다. 만약 `Partitioner`를 정의한 부모 RDD가 없다면 `spark.default.parallelism`에 지정된 파티션 개수를 사용한다.

### 4.2.2 불필요한 셔플링 줄이기

셔플링은 파티션 간의 _물리적인 데이터 이동_을 의미하며, 새로운 RDD의 파티션을 만들려고 _여러 파티션의 데이터를 합칠 때 발생_한다.

지난 장에서 살펴본 `aggregateByKey`예제를 들어 설명하면 스파크는 첫번째 변환 함수에서 각 파티션에 속한 요소들에 변환 함수를 적용해 중간 파일로 저장해두었다가, 이 중간 파일을 병합 함수의 입력으로 사용해 여러 파티션의 값을 최종 병합한다. 다시 말해 셔플링을 수행한다. 그리고 마지막으로 `Partitioner`를 적용해 각 키를 적절한 파티션에 할당한다.

셔플링 바로 전에 수행한 태스크를 **맵** 태스크라고 하며, 바로 다음에 수행한 태스크를 **리듀스** 태스크라고 한다.

셔플링에는 부담이 많이 가므로 웬만하면 스파크 잡의 셔플링 횟수를 최소한으로 줄여야 한다.

### 4.2.2.1 셔플링 발생 조건: `Partitioner`를 명시적으로 변경하는 경우

1. 사용자 정의 `Partitioner`를 사용하면 **반드시** 셔플링이 발생

   ```scala
    rdd.aggregateByKey(zeroValue, new CustomPartitioner())(seqFunc, comboFunc).collect()
   ```

2. 이전 `HashPartitioner`와 다른 `HashPartitioner`를 사용하면 셔플링 발생 \(`HashPartitioner`객체가 달라도 동일한 파티션 개수를 지정하면 같다고 간주 - 파티션 개수가 같으면 동일한 요소에서 항상 동일한 파티션 번호 선택\)

   ```scala
    rdd.aggregateByKey(zeroValue, 100)(seqFunc, comboFunc).collect()
   ```

#### 4.2.2.2 셔플링 발생 조건: `Partitioner`를 제거하는 경우

대표적으로 `map`과 `flatMap`은 RDD의 `Partitioner`를 **제거**한다. 이 자체로는 셔플링이 발생하지 않지만, 결과 RDD에 다른 변환 연산자를 사용하면 _기본 `Partitioner`를 사용했다 하더라도 여전히 셔플링이 발생_한다.

```scala
scala> rdd.map(x => (x, x*x)).map(_.swap).count() // 셔플링 발생하지 않음
scala> rdd.map(x => (x, x*x)).reduceByKey((v1, v2) => v1 + v2).count() // 셔플링 발생
```

`map`이나 `flatMap` 뒤에 사용하면 셔플링이 발생하는 연산자들

* RDD의 `Partitioner`를 변경하는 Pair RDD 변환 연산자: `aggregateByKey`, `foldByKey`, `reduceByKey`, `groupByKey`, `join`, `leftOuterJoin`, `rightOuterJoin`, `fullOuterJoin`, `subtractByKey`
* 일부 RDD 변환 연산자: `subtract`, `intersection`, `groupwith`
* `sortByKey` 변환 연산자\(셔플링이 무조건 발생\)
* `partitionBy` 또는 `suffle=true`로 설정한 `coalesce` 연산자

#### 4.2.2.3 외부 셔플링 서비스로 셔플링 최적화

셔플링을 수행하면 Executor는 다른 Executor의 파일을 읽어야 하는데, 일부 Executor에 장애가 발생하면 해당 Executor가 처리한 데이터를 더이상 가져올 수 없게 된다. 이럴 때 외부 셔플링 서비스\(`spark.shuffle.service.enabled`를 `true`로 설정\)를 사용하면 중간 셔플 파일을 읽을 수 있는 단일 지점을 제공한다.

#### 4.2.2.4 셔플링 관련 매개변수

스파크가 지원하는 셔플링은 아래 두 가지가 있다. `spark.shuffle.manager`를 `hash`또는 `sort`로 지정 1. 정렬 기반 셔플링: 파일을 더 적게 생성하고 메모리를 더욱 효율적으로 사용할 수 있어 스파크 1.2부터 기본 셔플링 알고리즘으로 사용 2. 해시 기반 셔플링

셔플링 관련 매개변수들

* `spark.shuffle.consolidateFiles`: 중간 파일의 통합 여부. `ext4`나 `XFS` 파일 시스템을 사용하면 `true`로 변경하는 것이 좋음 \(기본: `false`\)
* `spark.shuffle.spill`: 셔플링에 쓸 메모리 리소스 제한 여부. 메모리 임계치를 초과한 데이터는 디스크로 내보낸다. \(기본: `true`\)
* `spark.shuffle.memoryFraction`: 메모리 임계치. 너무 높으면 OOM이 뜰 수 있고, 너무 낮으면 데이터를 자주 내보내므로 균형을 맞추는 것이 중요. \(기본: 0.2\)
* `spark.shuffle.spill.compress`: 디스크로 내보낼 데이터의 압축 여부. \(기본: `true`\)
* `spark.shuffle.compress`: 중간 파일의 압축 여부 \(기본: `true`\)
* `spark.shuffle.spill.batchSize`: 데이터를 디스크로 내보낼 때 일괄로 직렬화 또는 역직렬화할 객체 개수\(기본: 1만\)
* `spark.shuffle.service.port`: 외부 셔플링 서비스를 활성화할 경우 서비스 서버가 사용할 포트 번호 \(기본: 7337\)

### 4.2.3 RDD 파티션 변경

작업 부하를 효율적으로 분산시키거나 메모리 문제를 방지하기 위해 RDD 파티셔닝을 명시적으로 변경해야 할 때가 있다.

#### 4.2.3.1 partitionBy

Pair RDD에서만 사용 가능하다. `Partitioner` 객체만 인자로 받는데 기존과 동일하면 파티셔닝을 유지하고 변경되면 셔플링 하고 새로운 RDD를 생성한다.

#### 4.2.3.2 coalesce와 repartition

_**coalesce 메서드 시그니처**_

```scala
def coalesce(numPartitions: Int, shuffle: Boolean = false)
```

파티션 개수를 변경하는 데 사용하는 연산자이며, 파티션 개수를 늘릴 때는 `shuffle`을 `true`로 줘야 하지만 파티션 개수를 줄일 때는 지정된 숫자의 파티션을 그대로 두고, 나머지 파티션을 고루 분배해 병합하는 방식으로 파티션의 개수를 줄이므로 `false`로 설정할 수 있다. `repartion`은 단순히 `shuffle`을 `true`로 설정한 메서드이다.

`shuffle`이 `false`인 경우, 셔플링이 발생하지 않는 `coalesce` 연산자 이전의 모든 변환 연산자는 여기서 지정된 파티션 개수를 사용하지만, `true`인 경우에는 `coalesce`연산자 이후에 파티션 개수가 바뀐다.

#### 4.2.3.3 repartitionAndSortWithinPartition

정렬 가능 RDD\(정렬 가능한 키로 구성된 Pair RDD\)에서만 사용 가능. `Partitioner`를 받아서 셔플링 단계에서 각 파티션 내의 요소를 정렬.

### 4.2.4 파티션 단위로 데이터 매핑

파티션 단위로 데이터를 매핑하는 연산자를 잘 호라용해 셔플링을 억제할 수 있다.

#### 4.2.4.1 mapPartitions와 mapPartitionsWithIndex

이전의 `map` 연산자가 받는 매핑 함수가 `T => U`였다면 `mapPartitions`의 매핑 함수는 `Iterator[T] => Iterator[U]` 형태이다. `mapParitionWithIndex`는 매핑 함수에 _파티션 번호가 함께 전달_된다.

Scala가 제공하는 `Iterator` 관련 함수

* `map`, `flatMap`, `zip`, `zipWithIndex` 등
* `take(n)`, `takeWhile(condition)` : 일부 요소 가져오기
* `drop(n)`, `dropWhile(condition)` : 일부 요소 제외하기
* `filter`
* `slice(m, n)` : 부분 집합 가져오기

`preservePartitioning` 인자를 받는데, 이 값을 `true`로 설정하면 부모 RDD의 파티셔닝이 유지된다. `false`로 지정하면 `Partitioner`를 제거하므로 셔플링이 발생한다.

`mapPartitions`를 사용하면 전체 파티션 당 한 번만 일어나도 되는 작업이 있는 경우\(데이터베이스 커넥션 등\)를 효율적으로 관리할 수 있다.

#### 4.2.4.2 파티션의 데이터를 수집하는 glom 변환 연산자

_각 파티션의 모든 요소를 배열 하나로 모으고, 이 배열들을 요소로 포함하는 새 RDD 반환_

`glom`은 기존의 `Partitioner`를 제거한다. RDD를 손쉽게 단일 배열로 가져올 수 있지만, 모든 요소를 단일 파티션에 저장할 수 있는지를 고려해야 한다.

## 4.3 데이터 조인, 정렬, 그루핑

* 어제 판매한 상품 이름과 각 상품별 매출액 합계
* 어제 판매하지 않은 상품 목록
* 전일 판매 실적 통계: 각 고객이 구입한 상품의 평균 가격, 최저 가격 및 최고 가격, 구매 금액 합계

구해보자!

### 4.3.1 데이터 조인

```scala
// tranData : 4.1에서 구한 전일 구매 기록
scala> val transByProd = tranData.map(tran => (tran(3).toInt, tran))
// 각 상품의 매출액 합계 (상품 ID, 매출액 합계)
scala> val totalsByProd = transByProd.mapValues(t => t(5).toDouble).reduceByKey{ case(tot1, tot2) => tot1 + tot2 }
// 각 상품 ID와 정보
scala> val products = sc.textFile("first-edition/ch04/ch04_data_products.txt").map(line => line.split("#")).map(p => (p(0).toInt, p))
```

#### 4.3.1.1 RDBMS와 유사한 조인 연산자

스파크의 조인 연산은 Pair RDD 에서만 사용 가능

* join: RDBMD의 inner join과 동일. 두 PairRDD에 모두 포함된 Key의 요소만을 결과에 포함한다.
  * PairRdd\[\(K, V\)\] `join` PairRdd\[\(K, W\)\] =&gt; PairRdd\[\(K, \(V, W\)\)\]
* leftOuterJoin: _첫 번째 PairRDD_에 있는 Key 요소는 모두 포함
  * PairRdd\[\(K, V\)\] `leftOuterJoin` PairRdd\[\(K, W\)\] =&gt; PairRdd\[\(K, \(V, Optioin\(W\)\)\)\]
  * 첫 번째 RDD에만 있는 Key의 요소는 \(K, \(V, None\)\)
* rightOuterJoin: _두 번째 PairRDD_에 있는 Key 요소는 모두 포함
  * PairRdd\[\(K, V\)\] `rightOuterJoin` PairRdd\[\(K, W\)\] =&gt; PairRdd\[\(K, \(Option\(V\), W\)\)\]
  * 두 번째 RDD에만 있는 Key의 요소는 \(K, \(None, W\)\)
* fullOuterJoin: 둘 중 한 쪽에만 있는 Key도 모두 포함
  * PairRdd\[\(K, V\)\] `fullOuterJoin` PairRdd\[\(K, W\)\] =&gt; PairRdd\[\(K, \(Option\(V\), Option\(W\)\)\)\]
  * 둘 중 한 쪽에만 있는 Key의 요소는 \(K, \(V, None\)\) or \(K, \(None, W\)\)

`Partitioner`나 파티션 개수를 지정할 수 있다. 둘 다 지정하지 않으면 첫 번째 RDD의 `Partitioner`를 사용한다. 파티션의 개수로는 `spark.default.partitions` 값을 사용하거나, 두 RDD 의 파티션 개수 중 더 큰 수를 사용한다.

```scala
// 상품 매출액 합계 RDD에 상품 정보 RDD 결합
scala> val totalsAndProds = totalsByProd.join(products)
totalsAndProds: org.apache.spark.rdd.RDD[(Int, (Double, Array[String]))] = MapPartitionsRDD[12]
// 어제 판매하지 않은 상품 목록
scala> val totalsWithMissingProds = products.leftOuterJoin(totalsByProd)
// OR
scala> val totalsWithMissingProds = totalsByProd.rightOuterJoin(products)
// 매출 데이터에 없는 요소(None)만 필터링해서 상품 정보만 가져오기
scala> val missingProds = totalsWithMissingProds.filter(x => x._2._1 == None).map(x => x._2._2)
```

#### 4.3.1.2 subtract subtractByKey 변환 연산자로 공통 값 제거

* `subtract`: 첫 번째 RDD에서 두 번째 RDD의 요소를 제거한 여집합 반환
* `subtractByKey`: Pair RDD 에서만 사용할 수 있으며, 첫 번째 RDD에서 두 번째 RDD에 _포함되지 않은 키_의 요소들을 RDD로 반환

```scala
// 어제 판매되지 않은 상품 목록
val missingProds = products.subtractByKey(totalsByProd).values
```

#### 4.3.1.3 cogroup 변환 연산자로 RDD 조인

`cogroup`은 여러 RDD 값을 키로 그루핑하고 각 RDD의 키별 값을 담은 `Iterable` 객체를 생성한 후, 이 `Iterable` 객체 배열을 Pair RDD로 반환한다. 어느 한 쪽에만 등장한 키의 경우 다른 쪽 RDD `Iterator`는 비어있다.

_**cogroup 메서드 시그니처**_

```scala
def cogroup[W1, W2](other1: RDD[(K, W1)], other2: RDD[(K, W2)]): RDD[(K, (Iterable[V], Iterable[W1], Iterable[W2]))]
```

`cogroup`을 이용해 어제 판매되지 않은 상품과 판매된 상품의 목록을 한꺼번에 찾을 수도 있다.

```scala
scala> val prodTotCogroup = totalsByProd.cogroup(products)
prodTotCogroup: org.apache.spark.rdd.RDD[(Int, (Iterable[Double], Iterable[Array[String]]))]
scala> val totalsAndProds = prodTotCogroup.filter(x => !x._2._1.isEmpty).map(x => (x._2._2.head(0).toInt, (x._2._1.head, x._2._2.head)))
```

#### 4.3.1.4 intersection 변환 연산자 사용

타입이 동일한 두 RDD에서 양쪽 모두에 포함된 공통 요소, 즉 교집합을 새로운 RDD로 반환

#### 4.3.1.5 cartesian 변환 연산자로 RDD 두 개 결합

두 RDD의 데카르트 곱\(cartesian product\) 계산을 튜플 형태의 RDD로 반환. 모든 파티션의 데이터를 결합해야 하므로 대량의 데이터가 네트워크로 전송되며, 메모리의 양도 많이 필요하다.

```scala
scala> val rdd1 = sc.parallelize(List(7, 8, 9))
scala> val rdd2 = sc.parallelize(List(1, 2, 3))
scala> rdd1.cartesian(rdd2).collect()
Array[(Int, Int)] = Array((7,1), (7,2), (7,3), (8,1), (8,2), (8,3), (9,1), (9,2), (9,3))
```

#### 4.3.1.6 zip 변환 연산자로 RDD 조인

RDD\[T\]의 `zip` 연산자에 RDD\[U\]를 전달하면 `zip` 연산자는 두 RDD의 각 요소를 순서대로 짝 지은 새로운 `RDD[(T, U)]`를 반환

스칼라의 `zip`과 유사하지만 _두 RDD의 파티션 개수_가 다르거나 _모든 파티션이 동일한 개수의 요소_를 가지지 않으면 오류가 발생한다.

#### 4.3.1.7 zipPartitions 변환 연산자로 RDD 조인

`zipPartitions`는 RDD를 네 개까지 결합할 수 있으며, 모든 RDD는 파티션 개수가 동일해야 하지만, _파티션에 포함된 요소 개수가 반드시 같은 필요는 없다_.

`zipPartitions`는 결합할 RDD외에도 입력 RDD 별로 각 파티션의 요소를 담은 `Iterator`를 받아 결과를 `Iterator`로 반환하는 조인 함수를 받는다. 조인 함수는 각 파티션 별로 요소 개수가 다를 수 있다는 점을 감안해 `Iterator` 길이를 초과하지 않게 작성해야 한다.

```scala
scala> val rdd1 = sc.parallelize(1 to 10, 10)
scala> val rdd2 = sc.parallelize((1 to 8).map(x => "n" + x), 10)
scala> rdd1.zipPartitions(rdd2, true)((iter1, iter2) => {
    iter1.zipAll(iter2, -1, "empty").map({ case (x1, x2) => x1 + "-" + x2 })
}).collect()
Array[String] = Array(1-empty, 2-n1, 3-n2, 4-n3, 5-n4, 6-empty, 7-n5, 8-n6, 9-n7, 10-n8)
```

### 4.3.2 데이터 정렬

RDD 데이터를 정렬하는데 주로 사용하는 변환 연산자는 `repartitionAndsortWithinPartition`, `sortByKey`, `sortBy` 등이 있다.

```scala
scala> val sortedProds = totalsAndProds.sortBy(_._2._2(1))
scala> val sortedProds = totalsAndProds.keyBy(_._2._2(1)).sortByKey()
```

#### 4.3.2.1 Ordered 트레잇으로 정렬 가능한 클래스 생성

`Ordered` 트레잇은 자바의 `Comparable` 인터페이스와 유사하다. `Ordered` 트레잇을 확장해 `compare` 함수를 구현하면 정렬 가능한 클래스를 만들 수 있다.

원래 `sortByKey` 변환 연산자를 사용하려면 `Ordering` 타입의 인수가 필요하지만, 스칼라가 암시적 변환으로 `Ordered`를 `Ordering`으로 변환하기 때문에 큰 문제는 없다.

#### 4.3.2.2 Ordering 트레잇으로 정렬 가능한 클래스 생성

`Ordering` 트레잇은 자바의 `Comparator` 인터페이스와 유사하다. 기본 타입에 대한 `Ordering` 객체는 스칼라 표준 라이브러리에 포함되어 있지만, 복합 객체를 키로 사용하려면 `Ordering`을 구현해야 한다.

#### 4.3.2.3 이차 정렬

고객 ID로 구매 기록을 그루핑한 후 구매 시작을 기준으로 각 고객의 구매 기록을 정렬하는 작업을 수행하고 싶다면? `groupByKeyAndSortValues` 변환 연산자를 사용할 수 있다. `groupByKeyAndSortValues`는 \(K, Iterable\(V\)\) 타입의 RDD를 반환한다. 하지만 키별로 값을 그루핑하기 때문에 상당한 메모리 및 네트워크 리소스를 사용한다.

혹은 다음과 같은 방법으로 그루핑없이 수행할 수 있다. 1. RDD\[\(K, V\)\]를 RDD\[\(\(K, V\), null\)\]로 매핑 2. 사용자 정의 `Partitioner`를 이용해 \(K, V\)에서 K 부분만으로 파티션을 나눈다. 3. `repartionAndSortWithinPartition` 변환 연산자에 사용자 정의 `Partitioner`를 전달해 호출하면, 전체 복합 키\(K, V\)를 기준으로 각 파티션 내 요소들을 정렬한다. 4. 결과를 다시 RDD\[\(K, V\)\]로 매핑한다.

#### 4.3.2.4 top과 takeOrdered로 정렬된 요소 가져오기

`takeOrdered(n)`이나 `top(n)` 행동 연산자를 사용해 _RDD 상위 또는 하위 n개 요소_를 가져올 수 있다. 여기서 상위 또는 하위 요소는 `Ordering[T]` 객체를 기준으로 결정된다.

`top`과 `taskOrdered`는 전체 데이터를 정렬하는 게 아니라, 각 파티션에서 n개 요소를 가져온 후 결과를 병합해 이 중 n개 요소를 최종적으로 반환한다.

### 4.3.3 데이터 그루핑

데이터를 특정 기준에 따라 단일 컬렉션으로 집계하는 연산

#### 4.3.3.1 groupByKey나 groupBy 변환 연산자로 데이터 그루핑

`groupByKey` 변환 연산자는 동일한 키를 가진 모든 요소를 단일 키-값 쌍으로 모은 Pair RDD 반환하며, 결과적으로 RDD는 `(K, Iterable[V])`로 구성된다. 대신 각 키의 모든 값을 메모리로 가져오기 때문에 주의해야 한다.

`groupBy`는 일반 RDD에도 사용할 수 있으며, 일반 RDD를 Pair RDD로 변환하고 `groupByKey`를 호출하는 것과 같은 결과를 만들 수 있다. 아래 두 결과는 같다.

```scala
rdd.map(x => (f(x), x)).groupByKey()
rdd.groupBy(f)
```

#### 4.3.3.2 combineByKey 변환 연산자로 데이터 그루핑

_**combineByKey 메서드 시그니처**_

```scala
def combineByKey[C](createCombiner: V => C, mergeValue: (C, V) => C, mergeCombiners: (C, C) => C, partitioner: Partitioner, mapSideCombine: Boolean = true, serializer: Serializer = null): RDD[(K, V)]
```

* `createCombiner`: 각 파티션 별로 키의 첫 번째 값\(C\)에서 최초 결합 값\(V\)을 생성하는 데 사용
* `mergeValue`: 동일 파티션 내에서 해당 키의 다른 값을 결합 값에 추가해 병합하는 데 사용
* `mergeCombiner`: 여러 파티션의 결합 값을 최종 결과로 병합하는 데 사용
* `partitioner`: `Partitioner`객체. 기존 `Partitioner`와 같은 경우 셔플링이 발생하지 않음
* `mapSideCombine`: 각 파티션의 결합 값을 계산하는 작업을 셔플링 단계 이전에 수행할지 여부
* `serializer`: 지정하지 않으면 `spark.serializer` 환경 변수에 지정된 기본 `Serializer` 사용

셔플링이 발생하지 않을 때는 `mergeCombiner`도 호출되지 않는다. 또는 셔플링 단계에서 데이터를 디스크로 내보내는 기능을 사용하지 않아도 호출되지 않는다.

**전일 판매 실적 통계\(각 고객이 구매한 상품의 평균, 최저 및 최고 가격과 구매 금액 합계\)를 구하는 예제**

```scala
def createComb = (t: Array[String]) => {
    val total = t(5).toDouble // 구매 금액
    val q = t(4).toInt // 구매 수량
    (total/q, total/q, q, total)
}
// 결합 값을 생성

def mergeVal: ((Double, Double, Int, Double), Array[String]) => (Double, Double, Int, Double) = {
    case((mn, mx, c, tot), t) => {
        val total = t(5).toDouble
        val q = t(4).toInt
        (scala.math.min(mn, total/q), scala.math.max(mx, total/q), c+q, tot + total)
    }
}
// 결합 값과 값을 병합

def mergeComb: ((Double, Double, Int, Double), (Double, Double, Int, Double)) => (Double, Double, Int, Double) = {
    case((mn1, mx1, c1, tot1), (mn2, mx2, c2, tot2)) => (scala.math.min(mn1, mn2), scala.math.max(mx1, mx2), c1 + c2, tot1 + tot2)
}
// 결합 값을 서로 병합

val avgByCust = transByCust.combineByKey(createComb, mergeVal, mergeComb, new org.apache.spark.HashPartitioner(transByCust.partitions.size)).mapValues({
    case(mn, mx, cnt, tot) => (mn, mx, cnt, tot, tot/cnt) // 마지막으로 상품의 평균 가격을 추가
})

avgByCust.first()
// (Int, (Double, Double, Int, Double, Double)) = (34,(3.942,2212.7425,82,77332.59,943.0803658536585))
```

**결과를 파일로 저장하는 예시**

```scala
scala> totalsAndProds.map(_._2).map(x => x._2.mkString("#") + ", " + x._1).saveAsTextFile("ch04output-totalsPerProd")
scala> avgByCust.map {
    case (id, (min, max, cnt, tot, avg)) => "%d#%.2f#%.2f#%d#%.2f#%.2f".format(id, min, max, cnt, tot, avg)}.saveAsTextFile("ch04output-avgByCust")
```

## 4.4 RDD 의존 관계

### 4.4.1 RDD 의존 관계와 스파크 동작 메커니즘

스파크의 실행 모델은 _RDD를 정점으로 RDD 의존 관계를 간선으로 정의한 **방향성 비순환 그래프\(Directed Acyclic Graph, DAG\)**_ 에 기반한다. RDD의 변환 연산자를 호출할 때마다 _새로운 정점\(RDD\)_ 와 _새로운 간선\(의존 관계\)_ 이 생성된다. 새로 생성된 RDD가 이전 RDD에 의존하므로 간선 방향은 자식 RDD\(새 RDD\)에서 부모 RDD\(이전 RDD\)로 향한다. 이런 _RDD 의존 관계 그래프를 **RDD 계보\(lineage\)**_ 라고 한다.

**RDD 의존 관계**

* 좁은\(narrow\) 의존 관계: 데이터를 다른 파티션으로 전송할 필요가 없는 변환 연산
  * 1-대-1\(one-to-one\) 의존 관계: 셔플링이 필요하지 않은 모든 변환 연산자에 사용
  * 범위형\(range\) 의존 관계: 여러 부모 RDD에 대한 의존 관계를 하나로 결합한 경우로 `union` 변환 연산자만 해당
* 넓은\(wide\)\(또는 셔플\(shuffle\)\) 의존 관계: 셔플링을 수행할 때 형성. RDD를 조인하면 무조건 셔플링이 발생

스파크 셸에서는 `toDebugString`을 이용해 각 RDD의 의존 관계를 확인할 수 있다. 여기서 프린트하는 계보는 아래에서 위로 흐르며, \(5\), \(3\)같은 괄호 안의 숫자는 해당 RDD의 파티션 개수를 의미한다.

```scala
scala> println(finalrdd.toDebugString)
(3) MapPartitionsRDD[33] at mapPartitions at <console>:32 []
 |  ShuffledRDD[32] at reduceByKey at <console>:30 []
 +-(5) MapPartitionsRDD[31] at map at <console>:28 []
    |  ParallelCollectionRDD[30] at parallelize at <console>:26 []
```

### 4.4.2 스파크의 스테이지와 태스크

스파크는 _셔플링이 발생하는 지점을 기준_으로 스파크 잡 하나를 여러 스테이지\(stage\)로 나눈다. 각 _스테이지의 결과는 중간 파일의 형태로 실행자의 로컬 디스크에 저장_된다. 그 다음 스테이지에서는 중간 파일 데이터를 적절한 파티션으로 읽어들인 후 연산을 수행한다.

스파크는 _스테이지와 파티션 별로 태스크를 생성_해 실행자에 전다한다. 스테이지가 셔플링으로 끝나는 경우 이 단계를 _셔플-맵\(shuffle-map\) 태스크_라고 한다. 드라이버는 태스크가 완료 되면 다음 스테이지의 태스크를 생성해 실행자에 전달한다. 마지막 스테이지에 생성된 태스크는 _결과 태스크\(result task\)_ 라고 하며 결과는 드라이버로 반환된다.

### 4.4.3 체크포인트로 RDD 계보 저장

스파크는 일부 노드에 장애가 발생해도 처음부터 다시 계산하는 게 아니라, 장애 발생 이전에 저장된 스냅샷을 사용해 _이 지점부터 나머지 계보를 다시 계산_하는 **체크포인팅\(checkpointing\)** 기능을 제공한다. 체크포인팅을 실행하면 스파크는 RDD 데이터 뿐 아니라 RDD 계보까지 모든 데이터를 디스크에 저장한다. 그리고 체크포인팅을 완료한 후에는 저장한 RDD를 다시 계산할 필요가 없으므로 해당 RDD 의존 관계와 부모 RDD 정보를 삭제한다.

RDD의 `checkpoint` 메서드를 호출해서 체크포인팅을 할 수 있으며, `checkpoint`는 해당 RDD에 잡이 실행되기 전에 호출해야 하고, 체크포인팅을 완료하려면 RDD에 행동 연산자 등을 호출해 잡을 실행하고 RDD를 구체화\(materialize\)해야 한다.

`SparkContext.setCheckpointDir`에 데이터를 저장할 디렉터리를 지정할 수 있다.

## 4.5 누적 변수와 공유 변수

* 누적 변수: 스파크 프로그램의 전역 상태 유지
* 공유 변수: 태스크 및 파티션이 공통으로 사용하는 데이터 공유

### 4.5.1 누적 변수로 실행자에서 데이터 가져오기

여러 실행자가 공유하는 변수로 값을 더하는 연산만 허용하며, 실행자는 누적 변수에 값을 더할 수는 있지만 누적 변수의 값은 오직 드라이버만 참조할 수 있다.

```scala
scala> val acc = sc.accumulator(0, "acc name")
scala> val list = sc.parallelize(1 to 1000000)
scala> list.foreach(x => acc.add(1)) // 실행자에서 수행
scala> acc.value // 드라이버에서 수행
res0: Int = 1000000
scala> list.foreach(x => acc.value) // 예외 발생!
java.lang.UnsupportedOperationException: Can't read accumulator value in task
```

#### 4.5.1.1 사용자 정의 누적 변수 작성

`Accumulator`값은 `AccumulatorParam` 객체를 사용해 누적한다. 숫자형 외의 커스텀 클래스를 누적 변수 값으로 사용하려면 `AccumulatorParam` 객체를 암시적으로 정의해야 한다.

`AccumulatorParam` 정의할 때 구현해야 하는 메서드

* `zero(initialValue: T)`: 실행자에 전달할 변수의 초깃값을 생성. 전역 초깃값도 동일하게 생성.
* `addInPlace(v1: T, v2: T)`: T: 누적 값 두 개를 병합

`AccumulableParam`에서 추가로 구현해야 하는 메서드

* `addAccumulator(v1: T, v2: V): T`: 누적 값에 새로운 값 누적

**RDD 요소의 개수와 총합을 집계하는 Accumulator**

```scala
scala> val rdd = sc.parallelize(1 to 100)
scala> import org.apache.spark.AccumulableParam
scala> implicit object AvgAccParam extends AccumulableParam[(Int, Int), Int] { // 암시적 AccumulableParam 객체 생성
    def zero(v: (Int, Int)) = (0, 0)
    def addInPlace(v1: (Int, Int), v2: (Int, Int)) = (v1._1 + v2._1, v1._2 + v2._2)
    def addAccumulator(v1: (Int, Int), v2: Int) = (v1._1 + 1, v1._2 + v2)
}

scala> val acc = sc.accumulable((0, 0))
scala> rdd.foreach(x => acc += x)
scala> val mean = acc.value._2.toDouble / acc.value._1
mean: Double = 50.5
```

#### 4.5.1.2 Accumuable 컬렉션에 값 누적

`SparkContext.accumulableCollection()`을 사용하면 암시적 객체 정의 없이 가변 컬렉션에 값을 누적할 수 있다.

```scala
scala> import scala.collection.mutable.MutableList
scala> val colacc = sc.accumulableCollection(MutableList[Int]())
scala> rdd.foreach(x => colacc += x)
scala> colacc.value
scala.collection.mutable.MutableList[Int] = MutableList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 31, 32, 33, ...)
```

순서가 제멋대로인 건 _각 파티션에서 생성된 누적 변수를 드라이버에 순서대로 반환한다는 보장이 없_기 때문이다.

### 4.5.2 공유 변수로 실행자에 데이터 전송

누적 변수와 마찬가지로 여러 클러스터 노드가 공동으로 사용할 수 있지만, 실행자가 수정할 수 없고 읽을 수만 있다.

드라이버에서 생성한 변수를 태스크에서 사용하면 스파크는 이 변수를 직렬화하고 태스크와 함께 실행자로 전송한다. 드라이버 프로그램은 동일한 변수를 여러 잡에 걸쳐 재사용할 수 있고 잡 하나를 수행할 때도 여러 태스크가 동일한 실행자에 할당될 수 있으므로, 실행자 대다수가 대용량 데이터를 공통으로 사용할 때 이 데이터를 공유 변수로 만드는 것이 좋다.

공유 변수는 `SparkContext.broadcast(value)` 메서드로 생성하며 실행자는 공유 변수가 없을 시 드라이버에 데이터를 청크 단위로 나누어 전송해줄 것을 요청한다. 공유 변수 값을 참조할 때는 `value`메서드를 사용해야 한다. 그렇지 않고 직접 접근하면 스파크는 이 변수를 자동으로 직렬화해 태스크와 함께 전송한다.

#### 4.5.2.1 공유 변수를 임시 제거 또는 완전 삭제

* `destroy`: 실행자와 드라이버에서 완전히 삭제. 삭제한 후 다시 접근하려고 하면 예외가 발생한다.
* `unpersist`: 실행자의 캐시에서 제거. 다시 사용하려고 하면 실행자로 다시 전송한다.

스파크는 공유 변수가 스포크를 벗어나면 자동으로 `unpersist` 를 호출

#### 4.5.2.2 공유 변수와 관련된 스파크 매개변수

* `spark.broadcast.compress`: 공유 변수를 전송하기 전에 데이터를 압축할지 여부 \(`true`가 좋음\)
  * `spark.io.compression.codec`: 공유 변수를 압축할 코덱
* `spark.broadcast.blockSize`: 공유 변수를 전송하는 데 사용하는 데이터 청크의 크기 \(기본값 4096KB가 좋음\)
* `spark.python.worker.reuse`: 워커를 재사용하지 않으면 각 태스크별로 공유 변수를 전송해야 하므로 파이썬의 공유 변수 기능에 큰 영향을 준다. 기본값인 `true`를 유지하자.

