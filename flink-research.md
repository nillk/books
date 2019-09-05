# flink-research

## Streaming

### Important aspects of stream processing

* Delivery Guarantees: 입력된 데이터가 처리될 것이라는 보장
  * Atleast-once: 장애에 상관 없이 _최소 한 번_은 처리 됨
  * Atmost-once: 장애의 경우 처리되지 않을 수 있음
  * Exactly-once: 장애에 상관 없이 딱 한 번 처리됨 당연히 이게 가장 바람직하지만 분산 시스템에서 달성하기가 어려움 보통은 성능과 trade-off
* Fault Tolerance: 노드나 네트워크 등 뭔가 장애가 있을 때 프레임워크가 복구할 수 있어야 하고, 실패한 시점부터 다시 시작할 수 있어야 함. state checkpointing을 통해 구현될 수 있음.
* State Management: 상태를 저장해야 하는 프로세싱의 경우 프레임워크는 데이터의 상태 정보를 저장하고 갱신할 수 있는 기능을 제공해야 함
* Performance
  * latency: 얼마나 빨리 입력 데이터가 처리 될 수 있는가. 작을 수록 좋음
  * throughput: 시간당 처리된 데이터의 양. 클수록 좋음
  * scalability
* Advanced Features: Event time processing, watermarks, windowing
* Maturity: 얼마나 성숙했는가는 프레임워크를 도입할 때 중요한 관점

### Two types of stream processing

* Native streaming: 모든 records\(events\)를 스트리밍 시스템에 도착하는 시점에 처리. 실행되고 있는 프로세스\(operators/tasks/bolts 등으로 불리는\)가 있으며 모든 record는 이 프로세스들을 통과한다. 모든 record가 도착 즉시 처리되므로 streaming이라고 했을 때 자연스럽게 느껴지며, 프레임워크가 최소한의 latency를 가질 수 있게 한다. 하지만, 처리량에 대한 타협이 없으면 fault tolerance를 보장하기 어렵고 한 번 처리되면 checkpoint를 관리해야 한다. 또한 계속해서 실행되는 프로세스가 있으면 필요한 상태를 유지하기 쉽다. ex\) storm, flink, kafka streams, samza...
* Micro-batching: Fast batching 이라고도 함. 유입되는 records\(events\)를 짧은 주기\(일반적으로 수초\)의 batch 처리가 가능한 단위로 묶어서 처리. 본질적으론 batch작업이므로 fault tolerance를 보장하기기가 쉽다. 또 한 단위의 record를 묶어서 처리하고 checkpointing 하므로, 처리량도 높다. 하지만 당연히 latency 있고, streaming이라고 생각하기에 자연스럽지 않다. ex\) spark streaming, storm-trident

### Stateless vs Stateful in stream processing

* Stateless: 모든 입력 데이터가 독립적. 입력되는 데이터 사이에는 관계가 없으므로 상태 없이 독립적으로 처리될 수 있다. ex\) map, filter, join...
* Stateful: 입력 데이처 처리가 이전 데이터의 처리 결과와 관계가 있음. 그래서 데이터를 처리할 때 중간 정보\(State\)를 저장할 필요가 있다. ex\) 레코트의 키 별로 aggregating count, deduplicating records, etc

State는 기본적으로 _stream을 처리할 때 유지되어야 하는 intermediate information_

#### 2 types of state in stream processing

1. State of Progress \(of Stream Processing\)

   stream processing의 metadata. checkpointing/saving of offsets of incoming data -&gt; fault tolerance -&gt; restart, upgrade, task failures...  

   stateless, stateful 둘 다 필요로 함

2. State of Data \(being processed in Stream Processing\)

   데이터에서 파생된 intermediate information  

   stateful 에서만 필요함

## Flink

* Stateful Computations over Data Streams
* Native streaming -&gt; low latency
* Exactly-once
* Stateful operations: 각 operator들이 데이터 처리 상태를 관리함
* Long running operator  

  each function like map, filter, reduce, etc is implemented as long running operator

Spark가 Hadoop batch의 성공적 후계자\(?\)라면 Flink는 Storm의 후계자라고 볼 수 있다

Spark와 반대로 data를 _batch로 처리하는 것을 예외적인 케이스_로 생각하며, streaming과 batch를 처리하기 위한 API가 다름

* `DataStream`: Streaming data\(unbounded streams\)를 처리하기 위한 클래스. immutable
* `DataSet`: Batch\(bounded streams\) 처리하기 위한 클래스. immutable. Bounded stream 데이터를 streaming 형식으로 처리. Checkpoint를 사용하지 않고 장애시 모두 재실행

이 두 클래스에서 사용할 수 있는 elements 타입에 제약이 있어서 아래 elements 만 가능

1. Java tuples and scala case classes
2. Java POJOs
3. Primitive types
4. Regular classes
5. Values
6. Hadoop Writables
7. Special types

### Flink 프로그램 실행 순서

1. Obtain an execution environment: ExecutionEnvironment를 생성해 DataStream, DataSet을 만들기 위한 준비
2. Load/Create the initial data: Data source를 생성해 input 데이터를 가져옴
3. Specify transformations on this data: 데이터를 변환 및 가공
4. Specify where to put the results of your computations: 계산된 결과를 저장하거나 활용
5. Trigger the program execution: 주기적으로 프로그램 실행

### Levels of Abstraction

* SQL: High level language
* Table API: Declarative DSL \(select, join, aggregate등의 고차원 함수 사용 가능\)
* DataStream/DataSet API: Core APIs
* Stateful Stream Processing: Low level building block \(streams, state, \[event\] time\)

### Dataflow

Source -&gt; Transformation\(s\) -&gt; Sink

Lazy evaluation이기 때문에 Sink가 실행되는 순간 Transformation\(s\)들이 실행됨

```java
// Source
DataStream<String> lines = env.addSource(new FlinkKafkaConsumer<>(...));
// Transformation
DataStream<Event> events = lines.map((line) -> parse(line));
DataStream<Statistics> stats = events
    .keyBy("id")
    .timeWindow(Time.seconds(10))
    .apply(new MyWindowAggregationFunction());
// Sink
stats.addSink(new RollingSink(path));
```

#### Data source 예시

기본적으로 `StreamExecutionEnvironment`로 stream source api 제공

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)

// File based - local, HDFS, S3...
val inputText = env.readTextFile("src/main/resources/log.txt")
inputText.print()

// Socket based
val socketStream = env.socketTextStream("localhost", 9999)
socketStream.print()

// Collection based
/**
 * fromCollection(Seq)
 * fromCollection(Iterator)
 * fromElements(elements: _*)
 * fromParallelCollection(SplittableIterator)
 * generateSequence(from, to)
 */
val collectionStream = env.fromCollection[Int](List(1, 2, 3, 4, 5))

// Custom
class CustomContinueSource(from: Int, to: Int) extends RichSourceFunction[Int] {
    val list = List.range(from, to)

    override def cancel(): Unit = ???

    override def run(ctx: SourceContext[Int]): Unit = {
        while (true) {
            for (i <- list) {
                Thread.sleep(100)
                ctx.collect(i)
            }
        }
    }
}

val stream = env.addSource(new CustomContinueSource(1, 10))

// 실행
env.execute("example-data-source")
```

#### Transformation functions

일반적으로 streaming 데이터 변환에 필요한 API를 대부분 제공한다.

Java

```java
// Implementing an interface
class MyMapFunction implements MapFunction<String, Integer> {
    public Integer map(String value) {
        return Integer.parseInt(value);
    }
}

data.map(new MyMapFunction());

// Anonymous classes
data.map(new MapFunction<String, Integer>() {
    public Integer map(String value) {
        return Integer.parseInt(value);
    }
});

// Java 8 Lambdas
data.filter(s -> s.startsWith("http://"));
data.reduce((a, b) -> a + b)

// Rich function
// MapFunction interface대신 RichMapFunction class를 extends할 수 있다
// RichFunction은 open, close, getRuntimeContext, setRuntimeContext 네가지 메서드를 추가로 사용할 수 있다
```

Scala

```scala
// Lambda functions
data.filter { _.startsWith("http://") }
data.reduce { (a, b) => a + b } // data.reduce { _ + _ }

// Rich function
class MyMapFunction extends RichMapFunction[String, Int] {
    def map(in: String): Int = { in.toInt }
}

data.map(new MyMapFunction())
```

#### Data sink

Flink는 lazy evalution이기 때문에 sink 과정이 없으면 데이터 처리 X

```scala
// File
val stream = env.fromCollection(List.range(1, 10))
stream.writeAsText("src/main/resources/sample.txt")

// Socket
val stream = env.fromCollection(List("A", "B", "C"))
stream.writeToSocket("localhost", 9999, new SimpleStringSchema())

// Custom
class DataSinkCustom extends RichSinkFunction[Int] {
    override def invoke(value: Int): Unit = {
        println(s"Custom Sink: $value")
    }
}

val stream = env.fromCollection(List.range(1, 10))
stream.addSink(new DataSinkCustom())
```

### Simple word count example

```scala
import org.apache.flink.streaming.api.scala._
import org.apache.flink.streaming.api.windowing.time.Time

object WordCount {
    def main(args: Array[String]): Unit = {
        val env = StreamExecutionEnvironment.getExecutionEnvironment
        val text = env.socketTextStream("localhost", 9999)

        val count = text
            .flatMap(_.split(" "))
            .map((_, 1))
            .keyBy(0)
            .timeWindow(Time.seconds(5), Time.seconds(1))
            .sum(1)

        count.print().setParallelism(1)
        env.execute("Socket WordCount")
    }
}
```

Bash에서 아래 명령어 입력 후 단어를 입력하면 WordCount console에서 counting 결과가 나타난다

```bash
nc -lk 9999
```

### 병렬 Dataflow

Flink는 분산 환경에서 Operator들이 parallel하게 처리될 수 있음\(여러 스레드에서 분산처리 한다고 생각하면 OK\)

* Stream -&gt; Stream partitions로 구성됨
* Operator -&gt; Operator subtasks로 구성됨

### 분산 환경

1. Master process\(Job Manager\): Task scheduling, checkpoint, recovery, Worker 관리
2. Worker process\(Task Manager\): Execute task. JVM process 단위로 동작 한 개 이상의 subtask들이 스레드로 실행. Task는 Task slot에서 실행되는데, Task slot은 각각 개별적인 메모리 공간에서 실행된다.

서로 간의 통신은 Actor system 사용

Stadalone, Container, YARN, Mesos 등 가능

### Windows

Streaming data는 unbounded data이기 때문에 각 element를 개별적으로 처리 하는 연산이라면 별 문제가 없는데, 집계 연산을 사용하는 경우 문제가 생긴다. 처음와 끝을 모르는데 평균값을 어떻게 구할까? 그래서 Windows라는 개념이 존재. _특정한 룰에 따라 일정 데이터를 모아 처리하는 개념_으로 Flink에서는 간단하게 `window`만 구현하면 된다.

#### Keyed Windows

```scala
stream
    .keyBy(...) // non keyed에는 없음
    .window(...) // required: assigner
    [.trigger(...)] // optional (else default trigger)
    [.evictor(...)] // optional (else no evictor)
    [.allowedLateness(...)] // optional (else zero)
    .reduce/fold/apply(...) // required: function
```

#### Non-keyed Windows

```scala
stream
    .windowAll(...) // required: assigner
    [.trigger(...)] // optional (else default trigger)
    [.evictor(...)] // optional (else no evictor)
    [.allowedLateness(...)] // optional (else zero)
    .reduce/fold/apply(...) // required: function
```

Flink에서 제공하는 Windows 종류: Tumbling, Sliding, Session, Global -&gt; 각각 시간, 갯수 기반으로 설정 가능

#### 1부터 9까지의 끊임없이 입력되는 숫자를 각각의 Windows 방식으로 처리하는 예시

**Tumbling Windows**

고정된 시간 단위로 중복 데이터 처리 없이 Window 설정

```scala
stream
    .windowAll(TumblingEventTimeWindows.of(Time.seconds(2))) // 2초 단위 window
    .apply(Operators.appendAllFunction)
    .print()

/*
 2초 간격이므로 window마다 처리하는 데이터 수는 차이가 있을 수 있음

 예시
 time: 21:44:52 160, Count: 3 -> 123
 time: 21:44:54 062, Count: 4 -> 4567
 time: 21:44:56 137, Count: 2 -> 89
 time: 21:44:58 102, Count: 4 -> 1234
 time: 21:45:00 168, Count: 4 -> 5678
 time: 21:45:02 235, Count: 2 -> 91
 time: 21:45:04 207, Count: 4 -> 2345
 time: 21:45:06 074, Count: 4 -> 6789
 time: 21:45:08 132, Count: 2 -> 12
 */
```

**Sliding Windows**

window 사이즈와 window slide 간격을 지정. 중복 데이터가 허용됨

```scala
stream
    .windowAll(SlidingEventTimeWindows.of(Time.seconds(4), Time.seconds(2))) // 2초마다 4초간 들어온 데이터 처리
    .apply(Operators.appendAllFunction)
    .print()

/*
 예시
 time: 22:16:56 338, Count: 3 -> 123
 time: 22:16:58 039, Count: 7 -> 1234567
 time: 22:17:00 107, Count: 7 -> 4567891
 time: 22:17:02 160, Count: 8 -> 89123456
 time: 22:17:04 124, Count: 8 -> 23456789
 */
```

**Session Windows**

Session gap이라는 개념을 도입해서, 데이터가 꾸준히 들어오다 지정한 session gap값 예를 들어 5초간 데이터가 들어오지 않으면 그 전 window시점부터 마지막으로 데이터가 들어온 시점까지 window를 나눈다. 따라서 window사이즈가 제각각일 수 있고, window에서 처리되는 데이터의 양도 제각각일 수 있다.

```scala
stream
    .windowAll(EventTimeSessionWindows.withGap(Time.milliseconds(500))) // session gap을 500ms로 지정
    .apply(Operators.appendAllFunction)
    .print()

/*
 1 부터 9까지 300ms 간격으로 생성되고, 다시 1을 생성할 때는 600ms의 간격이 있는 source

 예시
 time: 22:26:24 542, Count: 9 -> 123456789
 time: 22:26:27 768, Count: 9 -> 123456789
 time: 22:26:31 188, Count: 9 -> 123456789
 time: 22:26:34 473, Count: 9 -> 123456789
 */
```

**Global Windows**

하나의 window로 모든 데이터를 처리. 따라서 _trigger_와 _evictor_를 설정해야 한다.

* trigger: 가져올 데이터에 대한 정의
* evictor: 처리할 데이터에 대한 정의

```scala
stream
    .windowAll(GlobalWindows.create())
    .trigger(CountTrigger.of(3)) // 입력 데이터 3개마다 처리
    .evictor(CountEvictor.of(5)) // 5개의 데이터를 처리
    .apply(Operators.appendAllFunction)
    .print()

/*
 예시
 time: 22:34:34 507, Count: 3 -> 123
 time: 22:34:37 339, Count: 5 -> 23456
 time: 22:34:40 350, Count: 5 -> 56789
 time: 22:34:45 406, Count: 5 -> 89123
 time: 22:34:48 399, Count: 5 -> 23456
 time: 22:34:51 397, Count: 5 -> 56789

 만약 evictor부분을 빼버리면 아래와 같음
 time: 22:44:37 076, Count: 3 -> 123
 time: 22:44:39 919, Count: 6 -> 123456
 time: 22:44:42 915, Count: 9 -> 123456789
 time: 22:44:47 973, Count: 12 -> 123456789123
 time: 22:44:50 981, Count: 15 -> 123456789123456
 time: 22:44:53 973, Count: 18 -> 123456789123456789
 time: 22:44:59 017, Count: 21 -> 123456789123456789123
 time: 22:45:02 001, Count: 24 -> 123456789123456789123456
 */
```

더 참고할 만한 페이지

* [https://flink.apache.org/news/2015/12/04/Introducing-windows.html](https://flink.apache.org/news/2015/12/04/Introducing-windows.html)

### Time

* Event Time: 데이터가 발생한 시간
* Ingestion Time: 데이터가 Flink로 유입된 시간
* Processing Time: 데이터가 처리된 시간

### Checkpoint

Data source를 어느 정도 레코드 묶음 단위로 data stream 중간에 _checkpoint barrier_를 끼워 넣는다. 그래서 이 barrier가 Data sink에 도착하면 계산 완료로 간주\(source에서 필요 없는 데이터 삭제함\) -&gt; fault tolerance

중간에 문제가 생기면 checkpoint부터 다시 처리 -&gt; exactly-once 보장

### Spark와 비교표

|  | Flink | Spark |
| :--- | :--- | :--- |
| Data | DataSet | RDD |
| DataFrame | Table | DataFrame |
| Streaming | DataStream | SparkStreaming |
| Machine Learning | Flink ML | MLlib |
| Graph | Gelly | Graphx |
| SQL | - | SparkSQL |

Spark는 데이터 중심이기 때문에 RDD를 변환 -&gt; RDD를 변환 -&gt; RDD를 변환 형태로 감. 장애 시 Lineage를 이용해 _전체 과정을 다시 계산_ 실패한 micro-batch job을 새로운 worker 노드에서 재실행할 수 있음 Flink는 Operator 중심으로 _operator들 사이로 데이터를 흘려보냄_ -&gt; Data sink. 장애 시 마지막 Checkpoint 이후로 DataSource에서 Replay. ACK 처리는 하지 않고 barrier 기반으로 checkpoint 처리함

Spark는 데이터의 덩어리를 처리하므로 배치 모드, but flink는 실시간 데이터의 행 이후의 행을 처리할 수 있음\(?\)

Flink는 데이터 처리의 중간 결과\(!\)를 제공할 수 있음

### 참고

* [https://www.linkedin.com/pulse/spark-streaming-vs-flink-storm-kafka-streams-samza-choose-prakash](https://www.linkedin.com/pulse/spark-streaming-vs-flink-storm-kafka-streams-samza-choose-prakash)
* [https://medium.com/@chandanbaranwal/state-management-in-spark-structured-streaming-aaa87b6c9d31](https://medium.com/@chandanbaranwal/state-management-in-spark-structured-streaming-aaa87b6c9d31)
* [http://gyrfalcon.tistory.com/category/Big Data/Flink](http://gyrfalcon.tistory.com/category/Big%20Data/Flink)
* [https://www.popit.kr/%EC%95%84%ED%8C%8C%EC%B9%98-%EC%8B%A4%EC%8B%9C%EA%B0%84-%EC%B2%98%EB%A6%AC-%ED%94%84%EB%A0%88%EC%9E%84%EC%9B%8C%ED%81%AC-%EB%B9%84%EA%B5%90%EB%B6%84%EC%84%9D-1/](https://www.popit.kr/%EC%95%84%ED%8C%8C%EC%B9%98-%EC%8B%A4%EC%8B%9C%EA%B0%84-%EC%B2%98%EB%A6%AC-%ED%94%84%EB%A0%88%EC%9E%84%EC%9B%8C%ED%81%AC-%EB%B9%84%EA%B5%90%EB%B6%84%EC%84%9D-1/)
* [https://www.popit.kr/%EC%95%84%ED%8C%8C%EC%B9%98-%EC%8B%A4%EC%8B%9C%EA%B0%84-%EC%B2%98%EB%A6%AC-%ED%94%84%EB%A0%88%EC%9E%84%EC%9B%8C%ED%81%AC-%EB%B9%84%EA%B5%90%EB%B6%84%EC%84%9D-2/](https://www.popit.kr/%EC%95%84%ED%8C%8C%EC%B9%98-%EC%8B%A4%EC%8B%9C%EA%B0%84-%EC%B2%98%EB%A6%AC-%ED%94%84%EB%A0%88%EC%9E%84%EC%9B%8C%ED%81%AC-%EB%B9%84%EA%B5%90%EB%B6%84%EC%84%9D-2/)

### 시간이 더 있다면...

Spark streaming 2.3부터 structured streaming은 micro-batch에서 벗어남\(?\) [https://databricks.com/blog/2018/03/20/low-latency-continuous-processing-mode-in-structured-streaming-in-apache-spark-2-3-0.html](https://databricks.com/blog/2018/03/20/low-latency-continuous-processing-mode-in-structured-streaming-in-apache-spark-2-3-0.html)

