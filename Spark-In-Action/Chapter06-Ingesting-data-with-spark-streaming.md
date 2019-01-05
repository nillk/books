# 6장 스파크 스트리밍으로 데이터를 흐르게 하자
실시간 데이터 분석을 위해서는 확장성과 장애 내성을 갖춘 시스템이 필요(스파크의 특장점)
* 동일한 API로 스트리밍 프로그램과 일괄 처리 프로그램을 모두 지원하는 통합 플랫폼 제공
  * 실시간 레이어와 배치 레이어를 결합한 람다 아키텍처 구축 가능
* 하둡과 호환되는 여러 파일 시스템과 다양한 분산 시스템에서 데이터를 읽어들일 수 있는 커넥터를 제공

## 6.1 스파크 스트리밍 애플리케이션 작성
**미니 배치(mini-batch)**: 스파크 스트리밍은 특정 시간 간격 내에 유입된 데이터 블록을 RDD로 구성. 즉, *입력 데이터 스트림을 미니배치 RDD로 시분할*하고, 다른 스파크 컴포넌트는 이 *미니배치 RDD을 일반 RDD처럼 처리*

데이터 소스 -> 입력 데이터 스트림 -> 스파크 스트리밍 데이터 리시버 -> 스파크 스트리밍 -> 미니 배치 RDD -> 다른 스파크 컴포넌트

### 6.1.1 예제 애플리케이션
아래의 정보를 표시하는 증권사 대시보드 애플리케이션 구축
* 초당 거래 주문 건수
* 누적 거래액이 가장 많은 고객 1~5위
* 지난 1시간 동안 거래량이 가장 많았던 유가 증권 1~5위

### 6.1.2 스트리밍 컨텍스트 생성
어떤 클러스터를 사용하든 코어를 두 개 이상 실행자에 할당해야 한다. 스파크 스트리밍의 각 데이터 리시버가 입력 데이터를 처리하기 위한 코어(엄밀히 말하면 thread) 하나와 프로그램 연산을 수행하기 위한 코어 하나이다.

먼저 `StreamingContext` 인스턴스를 생성해야 한다. `SpackContext`와 `Duration` 객체(미니배치 RDD를 생성할 시간 간격 지정)를 이용해 초기화할 수 있다.

미니배치 간격은 최신 데이터를 얼마나 빨리 조회해야 할지, 성능 요구 사항, 클러스터 용량 등에 따라 다르다.

```scala
import org.apache.spark._
import org.apache.spark.streaming._
val ssc = new StreamingContext(sc, Seconds(5))

// SparkConf를 전달하면 새로운 SparkContext를 시작함
val conf = new SparkConf().setMaster("local[4]").setAppName("App Name")
val ssc = new StreamingContext(conf, Seconds(5))
```

### 6.1.3 이산 스트림 생성
#### 6.1.3.1 예제 데이터 내려받기
거래 주문 50만 건을 기록한 예제 파일을 입력 데이터로 사용

```bash
cd first-edition/ch06
tar xvzf orders.tar.gz
```

`StreamingContext`의 `textFileStream` 메서드는 지정된 디렉터리(HDFS, 아마존 S3, GlusterFS, 로컬 디렉터리 등 하둡과 호환되는 모든 유형의 디렉터리 가능)를 모니터링하고, 디렉터리에 *새로 생성된 파일*(즉, 이미 디렉터리에 존재하던 파일은 처리하지 않음. 파일에 데이터를 추가해도 마찬가지)을 읽어들여 스트리밍으로 전달한다.

여기선 셸 스크립트를 사용해 orders.txt파일을 50개로 분할(각 파일에 1만개 라인의 데이터)한 후, 각 파일을 입수로 지정된 HDFS 디렉터리에 3초 주기로 복사한다.

#### 6.1.3.2 DStream 객체 생성
`textFileStream` 메서드는 `DStream` 클래스의 인스턴스(이산스트림)를 반환

`DStream`: 스파크 스트리밍의 기본 추상화 객체. 입력 데이터 스트림에서 주기적으로 생성하는 일련의 RDD 시퀀스 표현. RDD와 동일하게 *지연 평가*

```scala
val filestream = ssc.textFileStream("/home/spark/ch06input/")
```

### 6.1.4 이산 스트림 사용
RDD와 마찬가지로 `DStream`은 이산 스트림을 다른 `DStream`으로 변환하는 다양한 메서드 제공(필터링, 매핑, 리듀스, 조인, 결합 등)

#### 6.1.4.1 데이터 파싱
```scala
// 데이터를 담을 case class 정의
import java.sql.Timestamp
case class Order(time: java.sql.Timestamp, orderId: Long, clientId: Long, symbol: String, amount: Int, price: Double, buy: Boolean)

// orders의 DStream의 각 RDD에 Order객체 저장
import java.text.SimpleDateFormat
val orders = filestream.flatMap(line => {
    val dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss")
    val s = line.split(",")
    try {
        assert(s(6) == "B" || s(6) == "S") // B: 매수, S: 매도
        List(Order(new Timestamp(dateFormat.parse(s(0)).getTime()), s(1).toLong, s(2).toLong, s(3), s(4).toInt, s(5).toDouble, s(6) == "B"))
    } catch {
        case e : Throwable => println("Wrong line format (" + e + "): " + line)
        List()
    }
})
```

#### 6.1.4.2 거래 주문 건수 집계
초당 거래 주문 건수를 `PairDStreamFunctions` 객체를 이용해 구할 수 있다. (2요소 튜플로 구성된 `DStream`은 암시적으로 `PairDStreamFunctions` 인스턴스로 변환됨)

```scala
// 주문 유형과 주문 건수로 이루어진 RDD
val numPerType = orders.map(o => (o.buy, 1L)).reduceByKey((c1, c2) => c1 + c2)
```

### 6.1.5 결과를 파일로 저장
`DStream`의 `saveAsTextFiles` 메서드에 prefix(필수)와 suffix(옵션)인수를 넘겨 *데이터를 주기적으로 저장할 경로를 구성*할 수 있다.

미니배치 RDD의 주기마다 아래와 같은 디렉터리를 생성해 데이터를 저장한다. 미니배치 RDD의 각 파티션은 *part-xxxxx* 라는 파일로 저장된다. (xxxxx는 파티션 번호)

**<접두어>-<밀리초 단위 시각>(.<접미어>)**

`saveAsTextFiles`는 하둡과 호환되는 분산 파일 시스템에 출력 파일을 저장한다. 로컬 클러스터에서 실행하면 로컬 파일 시스템에 저장한다.

```scala
// RDD폴더 별로 파티션 파일을 하나만 생성하기 위해 repartition
numPerType.repartition(1).saveAsTextFiles("/home/spark/ch06output/output", "txt")
```

### 6.1.6 스트리밍 계산 작업의 시작과 종료
아직 스트리밍 컨텍스트를 시작하지 않았기 때문에 바로 전 챕터까지의 코드를 실행해도 아무런 변화가 없다. 아래 코드를 이용해 스트리밍 컨텍스트를 시작할 수 있다.

```scala
ssc.start()
```

스트리밍 컨텍스트를 시작하면 `DStream`을 평가하고, 리시버를 *별도의 스레드에서 구동*한 후, `DStream` 연산을 수행한다.

> 동일한 `SparkContext` 객체를 사용해 `StreamingContext` 인스턴스를 여러 개 생성할 수는 있지만, 동일 JVM에서는 `StreamingContext`를 한 번에 하나 이상 시작할 수 없다.

스파크 독립형 애플리케이션에서도 `StreamingContext.start`를 호출해 스트리밍 컨텍스트와 리시버를 시작하는데, `ssc.awaitTermination()`을 호출하지 않으면 리시버 스레드를 시작해도 *드라이버의 메인 스레드가 종료*된다. 혹은 `awaitTerminationOrTimeout(<밀리초>)` 메서드를 사용할 수도 있다. 이 두 메서드는 스파크 스트리밍의 계산 작업을 종료할 때까지 스파크 애플리케이션을 대기시킨다.

#### 6.1.6.1 스파크 스트리밍으로 데이터 전송
이제 *splitAndSend.sh* 스크립트를 사용해 스파크 스트리밍 애플리케이션으로 데이터를 밀어 넣자. 스크립트는 *orderes.txt* 파일을 분할하고 지정된 디렉터리로 하나씩 복사한다.

```bash
chmod +x first-edition/ch06/splitAndSend.sh
mkdir /home/spark/ch06input # 스파크 스트리밍 코드에 사용한 입력 폴더
cd first-edition/ch06 # 디렉터리에 orderes.txt 파일이 있어야 함
./splitAndSend.sh /home/spark/ch06input local
```

#### 6.1.6.2 스파크 스트리밍 컨텍스트 종료
```scala
ssc.stop(false)
```

`stop` 메서드에 `false` 인수를 전달하면 스트리밍 컨텍스트는 스파크 컨텍스트를 중지하지 않는다. 전달하지 않으면 `spark.streaming.stopSparkContextByDefault`에 설정된 값(기본값: `true`)을 사용한다.

종료된 스트리밍 컨텍스트는 *다시 시작할 수 없*다.

#### 6.1.6.3 출력 결과
`saveAsTextFiles` 메서드에 의해 미니배치 별로 생성된 디렉터리에 존재하는 파일
* *part-00000*: 집계 결과
  * 예시  
  (false,9969)  
  (true, 10031)
* *_SUCCESS*: 폴더의 쓰기 작업을 성공적으로 완료했음을 표시

여러 디렉터리에 분할 저장되어 있지만, 스파크 API를 사용하면 여러 디렉터리의 파일을 간편하게 읽을 수 있다.

```scala
// * 문자를 사용하면 텍스트 파일 여러 개를 한꺼번에 읽어 들임
val allCounts = sc.textFile("/home/spark/ch06output/output*.txt")
```

### 6.1.7 시간에 따라 변화하는 계산 상태 저장
누적 거래액이 가장 많은 고객 1~5위나 지난 1시간 동안 거래량이 가장 많았던 유가 증권 1~5위 같은 데이터를 계산하려면, *시간과 미니배치에 따라 변화하는 상태(state)를 지속적으로 추적하고 유지*해야 한다.

시간이 흐름에 따라 주기적으로 새로운 데이터가 미니배치로 유입된다. `DStream`은 이 데이터를 처리하고 결과를 계산하는 일종의 *실시간 프로그램*이다. 우리는 스파크 스트리밍의 여러가지 메서드를 사용해 *과거 데이터와 현재 미니배치의 새로운 데이터를 결합하고, 결과를 계산한 후 새로운 데이터로 상태를 갱신*할 수 있다.

#### 6.1.7.1 `updateStateByKey`로 상태 유지
과거의 계산 상태를 현재 계산에 반영할 수 있는 메서드에는 `updateStateByKey`와 `mapWithState` 메서드가 있다. 이 두 메서드는 모두 `PairDStreamFunction` 클래스로 사용할 수 있다. (`Pair DStream`에서만 사용할 수 있다는 의미)

`updateStateByKey` 메서드는 두 가지 기본 버전을 제공하는데, `DStream`의 값만을 처리하거나, 키와 값을 모두 처리(키까지 변경 가능)하는 버전이다. 두 버전 모두 새로운 State DStream을 반환한다.

***`updateStateByKey` 메서드에 전달할 함수 시그니처(첫번째 버전)***
```scala
(Seq[V], Option[S]) => Option[S]
```
첫 번째 인수는 미니배치에 유입된 각 키의 `Seq`값 객체이고, 두 번째 인수는 키의 현재 상태 값이다. `Option`은 키의 상태를 아직 *계산한 적이 없*으면 `None`을 반환한다. 계산한 적은 있지만, 해당 키 값이 현재 미니배치에 유입되지 않은 경우 첫 번째 인수에 빈 `Seq` 객체가 전달된다.

대량의 키와 상태 객체를 유지할 때는 적절한 파티셔닝이 중요한데 `updateStateByKey`에는 결과 `DStream`에 사용할 파티션 개수나 `Partitioner` 객체를 선택 인수로 지정할 수 있다.

```scala
val amountPerClient = orders.map(o => (o.clientId, o.amount * o.price))

val amountState = amountPerClient.updateStateByKey((vals, totalOpt: Option[Double]) => {
    totalOpt match {
        case Some(total) => Some(vals.sum + total) // 키의 상태가 이미 존재할 때는 상태 값에 새로 유입된 값의 합계를 더함
        case None => Some(vals.sum) // 이전 상태 값이 없을 때는 새로 유입된 값의 합계만 반환
    }
})

val top5clients = amountState.transform(_.sortBy(_._2, false).zipWithIndex.filter(x => x._2 < 5).map(x => x._1))
```

#### 6.1.7.2 `union`으로 두 `DStream` 병합
초당 거래 주문 건수 `DStream`과 거래액 1~5위 고객 `DStream`을 배치 간격당 한 번씩만 저장하려면, 하나의 단일 `DStream`으로 병합해야 한다. 스파크 스트리밍에는 `join`, `cogroup`, `union` 메서드를 사용해 두 개의 `DStream`을 하나로 병합할 수 있다.

`union` 메서드를 사용하려면 요소 타입이 서로 동일해야 한다. 따라서 예제에서는 2-요소 튜플로 첫 번째 요소에는 지표를 설명하는 키, 두 번째 요소에는 문자열 리스트(거래액 1~5위 데이터가 리스트이고, 거래량 1~5위 유가 증권 종목 코드가 문자열이기 때문)를 저장하기로 한다.

```scala
val buySellList = numPerType.map(t =>
    if(t._1) ("BUYS", List(t._2.toString))
    else ("SELLS", List(t._2.toString)))

val top5clList = top5clients.repartition(1). // 데이터를 단일 파티션으로 모음
    map(x => x._1.toString). // 고객 ID만 string으로
    glom(). // 고객 ID를 단일 배열로 모음
    map(arr => ("TOP5CLIENTS", arr.toList))

// union으로 병합
val finalStream = buySellList.union(top5clList)

// 병합된 DStream을 파일에 저장
finalStream.repartition(1).saveAsTextFiles("/home/spark/ch06output/output", "txt")
```

#### 6.1.7.3 체크포인팅 디렉터리 지정
```scala
sc.setCheckpointDir("/home/spark/checkpoint")
```

`updateStateByKey`는 *각 미니배치마다 RDD의 DAG를 계속 연장하면서 스택 오버플로 오류를 유발*하기 때문(왜???)에 체크포인팅을 반드시 적용해야 한다.

#### 6.1.7.4 두 번째 출력 결과
스트리밍 컨텍스트를 시작 `ssc.start()`하고 `rm -f /home/spark/ch06input/` 명령으로 입력 폴더를 리셋한 후, *splitAndSend.sh* 스크립트를 재실행하면 아래와 같은 파일이 생성된다.

*part-00000*의 예  
(SELLS,List(4926))  
(BUYS,List(5074))  
(TOP5CLIENTS,List(34, 69, 92, 36, 64))

#### 6.1.7.5 `mapWithState`
`mapWithState`는 `updateStateByKey`의 기능 및 성능을 개선한 메서드로 스파크 버전 1.6부터 사용할 수 있다. 그리고 차이점으로 `mapWithState`는 *상태 값의 타입과 반환 값의 타입을 다르게 적용*할 수 있다.

`mapWithState` 메서드는 `StateSpec` 클래스의 인스턴스만 인수로 받는다. `StateSpec.function` 메서드에 매핑 함수 전달해서 `StateSpec` 객체를 초기화할 수 있다.

***`StateSpec.function` 메서드에 전달할 함수 시그니처***
```scala
(Time, KeyType, Option[ValueType], State[StateType]) => Option[MappedType]
```
* `Time`: 선택 인수이므로 생략 가능
* `KeyType`: 키
* `ValueType`: 새로 유입된 값
* `StateType`: 기존 상태 값

결과값이 `StateType`과 다른 `MappedType`인 게 `updateStateByKey`와 다른 점

`State`는 아래 여러 가지 유용한 메서드를 제공
* `exists`: 상태 값이 존재하면 `true`
* `get`: 상태 값을 가져온다
* `remove`: 상태를 제거
* `update`: 상태 값을 갱신(또는 초기화)

```scala
val updateAmountState = (clientId: Long, amount: Option[Double], state: State[Double]) => {
    val total = amount.getOrElse(0.toDouble) // 새로 유입된 값을 새로운 상태 값으로 설정 기본값은 0
    if (state.exists())
        total += state.get()
    state.update(total)
    Some((clientId, total))
}

val amountState = amountPerClient.mapWithState(StateSpec.function(updateAmountState)).stateSnapshots() // stateSnapshots 를 호출해야 전체 상태를 포함한 DStream을 반환한다. 이 메서드를 호출하지 않으면 mapWithState는 현재 미니배치의 계산값만 포함한다
```

`StateSpec` 객체에는 파티션 개수, `Partitioner` 객체, 초기 상태 값을 담은 RDD, 제한 시간 등을 설정할 수 있다.

```scala
StateSpec.function(updateAmountState).numPartitions(10).timeout(Minutes(30)) // State Spec은 빌더 패턴으로 구현되어 있음
```

### 6.1.8 윈도 연산으로 일정 시간 동안 유입된 데이터만 계산
지난 1시간 동안 거래량이 가장 많았던 유가 증권 1~5위 찾기

이 지표는 *일정 시간 내의 데이터만 대상으로 계산*해야 하는데, 스파크 스트리밍에서는 *윈도 연산(window operation)* 을 사용해 구현할 수 있다. 윈도 연산은 미니배치의 슬라이딩 윈도를 기반으로 수행하며, *슬라이딩 윈도의 길이와 이동 거리(윈도 데이터를 얼마나 자주 계산할지)* 를 바탕으로 *윈도 `DStream`*을 생성한다.

슬라이딩 윈도의 길이와 이동 거리는 *반드시 미니배치 주기의 배수*여야 한다.

#### 6.1.8.1 윈도 연산을 사용해 마지막 지표 계산
우리는 1시간 단위로 거래량 상위 1~5위를 구해야 하므로 슬라이딩 윈도의 길이는 1시간으로 지정하고, 이 값을 각 미니배치마다 계산해야 하므로 슬라이딩 윈도의 이동 거리는 미니 배치 주기(5초)와 동일하게 설정한다.

`reduceByKeyAndWindow` 메서드는 윈도 `DStream`을 생성하고 데이터를 리듀스 함수에 전달한다.

```scala
val stocksPerWindow = orders.
    map(x => (x.symbol, x.amount)).
    reduceByKeyAndWindow((a1: Int, a2: Int) => a1 + a2, Minutes(60)) // reduce 함수와 윈도 길이 지정. 윈도 이동 거리를 지정하지 않았기 때문에 기본값으로 미니배치 주기 사용

val topStocks = stocksPerWindow.transform(_.sortBy(_._2, false).map(_._1).
    zipWithIndex.filter(x => x._2 < 5)).repartition(1).map(x => x._1.toString).glom().map(arr => ("TOP5STOCKS", arr.toList))

val finalStream = buySellList.union(top5clList).union(topStocks)
```

*part-00000*의 예  
(SELLS,List(9969))  
(BUYS,List(10031))  
(TOP5CLIENTS,List(15, 64, 55, 69, 19))  
(TOP5STOCKS,List(AMD, INTC, BP, EGO, NEM))

#### 6.1.8.2 그 외 다른 윈도 연산자
이 중 일부는 일반 `DStream`에서도 사용할 수 있으며, `ByKey`가 포함된 함수들은 `Pair DStream`에서만 사용할 수 있다.

* `window(winDur, [slideDur])`
* `countByWindow(winDur, slideDur)`
* `countByValueAndWindow(winDur, slideDur, [numParts])`
* `reduceByWindow(reduceFunc, winDur, slideDur)`
* `reduceByWindow(reduceFunc, invReduceFunc, winDur, slideDur)`
* `groupByKeyAndWindow(winDur, [slideDur], [numParts/partioner])`
* `reduceByKeyAndWindow(reduceFunc, winDur, [slideDur], [numParts/partioner])`
* `reduceByKeyAndWindow(reduceFunc, invReduceFunc, winDur, [slideDur], [numParts/partioner])`

### 6.1.9 그 외 내장 입력 스트림
`textFileStream` 외에도 다양한 메서드로 데이터를 수신하고 `DStream`을 생성할 수 있다.

#### 6.1.9.1 파일 입력 스트림
아래 두 메서드는 `textFileStream`과 마찬가지로 *특정 폴더 아래 새로 생성된 파일들*을 읽어 들인다. 대신 텍스트 외 *다른 유형의 파일*을 읽을 수 있다.
* `binaryRecordStream`: 폴더 이름과 레코드 크기를 인수로 전달하면, 이진 파일에서 특정 크기의 레코드를 읽어 들여 바이트 배열(`Array[Byte]`)로 구성된 `DStream`을 반환
* `fileStream`: 키의 클래스 타입과 값의 클래스 타입, HDFS 파일을 읽는 데 사용할 입력 포맷의 클래스 타입(하둡의 `NewInputFormat` 클래스를 상속한 하위 클래스) 그리고 파일을 읽어 들일 디렉터리 경로를 지정하면, 키와 값 튜플을 요소로 가지는 `DStream`을 반환
  * `filter` 함수: 각 `Path` 객체(하둡에서 파일을 표현하는 클래스)별로 해당 파일의 처리 여부 판단
  * `newFilesOnly` 플래그: 새로 생성된 파일만 처리할지, 모든 파일을 처리할지 지정
  * 하둡 `Configuration` 객체: HDFS 파일을 읽는 데 필요한 추가 옵션 설정

#### 6.1.9.2 소켓 입력 스트림
TCP/IP 소켓에서 바로 데이터 수신. 다만 소켓 스트림의 리시버는 클러스터의 노드 중 단일 노드의 단일 실행자에서만 실행된다.

`StorageLevel`을 선택 인자로 지정할 수 있는데, 이는 데이터가 보관될 위치와 데이터 복제 여부를 결정한다.
* `socketTextStream`: 각 요소가 UTF-8로 인코딩한 문자열 줄로 구성된 `DStream` 반환
* `socketStream`: 이진 데이터를 읽는 자바 `InputStream` 객체를 결과 `DStream`의 요소가 될 객체로 변환하는 변환 함수를 전달해야 함.