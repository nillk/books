# 5장 스파크 SQL로 멋진 쿼리를 실행하자

스파크에서는 DataFrame으로 _정형 데이터\(structured data\)_ 를 다룰 수 있다. 이 정형 데이터를 구성하고 질의\(query\)하는 가장 일반적인 방법은 RDB에서 사용하는 SQL\(Structured Query Language\)이다.

> 정형 데이터: 로우와 칼럼으로 구성되며, 각 칼럼 값이 특정 타입으로 제한된 데이터 구조

참고

* [SQL 튜토리얼](https://www.w3schools.com/SQL)

## 5.1 DataFrame 다루기

RDD는 _데이터를 직접 다룰 수 있는 스파크 하위 레벨 인터페이스이자 스파크 런타임의 핵심!_ DataFrame API는 스파크 버전 1.3에서 소개되었으며, 칼럼 이름과 타입이 지정된 _테이블 형식의 분산 정형 데이터_ 를 손쉽게 다룰 수 있는 상위 레벨 인터페이스를 제공한다. 스파크 2.0은 DataFrame을 DataSet의 일종으로 구현했다.

스파크 DataFrame은 SQL 및 도메인 특화 언어\(DSL\)로 작성된 표현식을 _최적화된 하위 레벨 RDD 연산으로 변환_ 한다. 또한 DataFrame을 테이블 카탈로그에 등록하여 다른 스파크 애플리케이션에서도 DataFrame 이름을 이용해 데이터에 질의를 수행할 수 있게 한다. 이는 데이터 자체를 저장하는 것이 아니라 _정형 데이터로 접근하는 방법만 저장_ 한다.

스파크 쓰리프트 서버를 이용하면 스파크 외부의 비스파크 애플리케이션에서도 표준 JDBC 및 ODBC 프로토콜로 일반적인 SQL쿼리를 전송하여 스파크 DataFrame에서 질의를 수행할 수 있다.

DataFrame을 생성하는 방법

* 기존 RDD를 변환
* SQL 쿼리를 실행
* 외부 데이터에서 로드

### 5.1.1 RDD에서 DataFrame 생성

일반적으로 데이터를 RDD로 로드해 정형화한 후, DataFrame으로 변환하는 방법을 가장 많이 사용한다. 정형화하지 않으면 DataFrame API를 사용할 수 없다.

RDD에서 DataFrame을 생성하는 방법

* 로우의 데이터를 튜플 형태로 저장한 RDD 사용
  * 간단하지만 스키마 속성을 전혀 지정할 수 없으므로 제한적
* 케이스 클래스를 사용
  * 케이스 클래스를 작성해야 하므로 조금 복잡하지만 덜 제한적
* 스키마를 명시적으로 지정
  * 스키마를 명시적으로 지정하며 가장 많이 사용
  * 첫 번째와 두 번째는 스키마를 간접적으로 지정\(즉, 추론\)

#### 5.1.1.1 SparkSession을 생성하고 암시적 메서드 임포트

스파크 DataFrame과 SQL 표현식을 사용하려면 먼저 `SparkSession` 객체를 준비해야 한다. `SparkSession`은 `SparkContext`와 `SQLContext`를 통합한 Wrapper 클래스다.

```scala
import org.apache.spark.sql.SparkSession
val spark = SparkSession.builder().getOrCreate()
```

스파크는 RDD를 DataFrame으로 자동 변환하는 데 필요한 암시적 스칼라 메서드들을 제공하지만, 이 메서드들을 사용하려면 `spark`라는 이름으로 `SparkSession` 객체를 생성할 후, 아래와 같이 import 해야 한다. \(스파크 셸을 시작하면 이 코드를 자동으로 실행\)

```scala
import spark.implicits._
```

`implicits` 객체는 Dataset Encoder가 정의된 객체로 구성한 RDD에 `toDF`라는 메서드를 추가한다. `Encoder`는 스파크 SQL이 내부적으로 JVM객체를 테이블 형식으로 변환하는 데 사용하는 트레잇이다. [기본 `Encoder` 목록 문서](http://spark.apache.org/docs/2.0.0/api/scala/index.html#org.apache.spark.sql.SQLImplicits)

#### 5.1.1.2 예제 데이터셋 로드 및 파악

실습에는 Stack Exchange 이탈리아 커뮤니티 데이터 사용

```scala
scala> val itPostsRows = sc.textFile("first-edition/ch05/italianPosts.csv")
scala> val itPostsSplit = itPostsRows.map(x => x.split("~"))
isPostsSplit: org.apache.spark.rdd.RDD[Array[String]] = ...
```

이제 이 데이터의 문자열을 각각의 컬럼으로 매핑해 DataFrame 으로 변환하자

#### 5.1.1.3 튜플 형식의 RDD에서 DataFrame 생성

```scala
// 1. 튜플로 변환
scala> val itPostsRDD = itPostsSplit.map(x => (x(0),x(1),x(2),x(3),x(4),x(5),x(6),x(7),x(8),x(9),x(10),x(11),x(12)))
itPostsRDD: org.apache.spark.rdd.RDD[(String, String, ...
// 2-1. toDF 호출해 DataFrame 생성
scala> val itPostsDFrame = itPostsRDD.toDF()
itPostsDFrame: org.apache.spark.sql.DataFrame = [_1: string, ...
// 2-2. toDF에 컬럼명 입력해서 DataFrame 생성
scala> val itPostsDF = itPostsRDD.toDF("commentCount", "lastActivityDate", "ownerUserId", "body", "score", "creationDate", "viewCount", "title", "tags", "answerCount", "acceptedAnswerId", "postTypeId", "id")
itPostsDFrame: org.apache.spark.sql.DataFrame = [commentCount: string, ...
// 3. show 호출하여 상위 10개 로우 확인
scala> itPostsDF.show(10)
// printSchema를 이용해 스키마를 살펴볼 수 있음
scala> itPostsDF.printSchema
```

`printSchema`로 칼럼 정보를 출력해보면 칼럼 이름은 잘 들어갔지만, 모든 칼럼의 타입이 `String`이며 `nullable`인 것을 알 수 있다. 하지만 예제 데이터는 각자 칼럼 타입이 있으므로 우리는 RDD를 DataFrame으로 변환할 때 _1. 칼럼 이름_ 과 _2. 데이터 타입_ 을 지정할 수 있어야 한다.

#### 5.1.1.4 케이스 클래스를 사용해 RDD를 DataFrame으로 변환

RDD의 각 로우를 _케이스 클래스_ 로 매핑한 후 `toDF` 메서드 호출

```scala
import java.sql.Timestamp
// nullable 필드는 Option[T]
case class Post(
    commentCount:Option[Int],
    lastActivityDate:Option[java.sql.Timestamp],
    ownerUserId:Option[Long],
    body:String,
    score:Option[Int],
    creationDate:Option[java.sql.Timestamp],
    viewCount:Option[Int],
    title:String,
    tags:String,
    answerCount:Option[Int],
    acceptedAnswerId:Option[Long],
    postTypeId:Option[Long],
    id:Long)

// 아래 같은 암시적 클래스를 정의해 itPostsRDD를 Post객체로 깔끔하게 매핑할 수 있다
// catching 함수는 scala.util.control.Exception.Catch 타입의 객체를 반환하는데, 이 객체의 opt 메서드는 사용자가 지정한 함수(ex: s.toInt) 결과를 Option 객체로 매핑한다. catching에 지정한 예외 발생 -> None, 그 외 -> Some
object StringImplicits {
    implicit class StringImprovements(val s: String) {
        import scala.util.control.Exception.catching
        def toIntSafe = catching(classOf[NumberFormatException]) opt s.toInt
        def toLongSafe = catching(classOf[NumberFormatException]) opt s.toLong
        def toTimestampSafe = catching(classOf[IllegalArgumentException]) opt Timestamp.valueOf(s)
    }
}

// RDD 로우 파싱 메서드 작성
import StringImplicits._
def stringToPost(row: String): Post = {
    val r = row.split("~")
    Post(r(0).toIntSafe,
        r(1).toTimestampSafe,
        r(2).toLongSafe,
        r(3),
        r(4).toIntSafe,
        r(5).toTimestampSafe,
        r(6).toIntSafe,
        r(7),
        r(8),
        r(9).toIntSafe,
        r(10).toLongSafe,
        r(11).toLongSafe,
        r(12).toLong
    )
}

// 로우 파싱 메서드를 사용해 RDD를 DF로 변환
val itPostsDFCase = itPostsRows.map(x => stringToPost(x)).toDF()

// schema 확인
itPostsDFCase.printSchema
```

#### 5.1.1.5 스키마를 지정해 RDD를 DataFrame으로 변환

`SparkSession`의 `createDataFrame` 메서드에 _1. `Row` 타입의 객체를 포함하는 RDD_ 와 _2. `StructType` 객체_ 를 인자로 전달해 DataFrame 생성

`StructType`은 스파크 SQL의 테이블 스키마를 표현하는 클래스이며, 테이블 칼럼을 표현하는 `StructField` 객체를 한 개 또는 여러 개 가질 수 있다.

```scala
import org.apache.spark.sql.types._
// schema 정의
val postSchema = StructType(Seq(
    StructField("commentCount", IntegerType, true),
    StructField("lastActivityDate", TimestampType, true),
    StructField("ownerUserId", LongType, true),
    StructField("body", StringType, true),
    StructField("score", IntegerType, true),
    StructField("creationDate", TimestampType, true),
    StructField("viewCount", IntegerType, true),
    StructField("title", StringType, true),
    StructField("tags", StringType, true),
    StructField("answerCount", IntegerType, true),
    StructField("acceptedAnswerId", LongType, true),
    StructField("postTypeId", LongType, true),
    StructField("id", LongType, true))
)

import org.apache.spark.sql.Row
// 각 요소를 직접 지정하거나, Seq, Tuple 객체를 전달해 Row를 구성해야 함
// 스파크 버전 2.0이 스칼라의 Option과 잘 호환되지 않아 자바의 null 사용
def stringToRow(row: String): Row = {
    val r = row.split("~")
    Row(r(0).toIntSafe.getOrElse(null),
        r(1).toTimestampSafe.getOrElse(null),
        r(2).toLongSafe.getOrElse(null),
        r(3),
        r(4).toIntSafe.getOrElse(null),
        r(5).toTimestampSafe.getOrElse(null),
        r(6).toIntSafe.getOrElse(null),
        r(7),
        r(8),
        r(9).toIntSafe.getOrElse(null),
        r(10).toLongSafe.getOrElse(null),
        r(11).toLongSafe.getOrElse(null),
        r(12).toLong)
}

// Row로 변환
val rowRDD = itPostsRows.map(row => stringToRow(row))
// Row RDD와 schema로 DataFrame 생성
val itPostsDFStruct = spark.createDataFrame(rowRDD, postSchema)
```

DataFrame은 대부분 RDB에서 일반적으로 지원하는 `String`, `integer`, `float`, `double`, `byte`, `Date`, `Timestamp`, `Binary`\(`BLOB`\) 타입뿐 아니라 배열, 맵, 구조체 등 복합 데이터 타입도 지원

#### 5.1.1.6 스키마 정보 가져오기

* `printSchema`: 스키마 정보 출력
* `schema`: `StructType`으로 정의된 스키마 정보
* `columns`: 칼럼 이름 목록
* `dtypes`: 칼럼 이름과 타입으로 구성된 튜플 목록

### 5.1.2 기본 DataFrame API

DataFrame의 DSL은 RDB의 SQL과 유사한 기능을 제공. DataFrame은 RDD와 마찬가지로 _불변성_ 과 _지연 실행_ 하는 특징이 있다.

#### 5.1.2.1 칼럼 선택

대부분의 DataFrame DSL 함수는 `Column` 객체를 입력받을 수 있다.

```scala
val postsDf = itPostsDFStruct

/* SELECT 칼럼들을 입력받아 이 칼럼들만으로 구성된 새로운 DataFrame 반환 */

// 칼럼 명
val postsIdBody = postsDf.select("id", "body")

// DataFrame 객체의 col 함수
val postsIdBody = postsDf.select(postsDf.col("id"), postsDf.col("body"))

// SparkSession.implicits 객체의 암시적 메서드 사용
// SymbolToColumn 메서드는 스칼라의 Symbol 클래스를 Column으로 변환
val postsIdBody = postsDf.select(Symbol("id"), Symbol("body"))
val postsIdBody = postsDf.select('id, 'body)

// implicits의 또 다른 암시적 메서드 $ 사용해 문자열을 ColumnName(Column을 상속한 클래스) 객체로 변환해 사용 
val postsIdBody = postsDf.select($"id", $"body")

/* DROP 칼럼을 하나 입력받아 칼럼 한 개를 제외한 새로운 DataFrame 반환 */
// 칼럼 명
val postsIds = postsIdBody.drop("body")

// ...
```

#### 5.1.2.2 데이터 필터링

`where`와 `filter`가 있으며 두 함수는 동일하게 동작한다. 이 두 함수는 _1. `Column` 객체_ 또는 _2. SQL구문으로 이루어진 문자열 표현식_ 을 인수로 받는다.

`Column` 클래스는 칼럼 이름을 지정하는 기능 외에 _다양한 유사 SQL 연산자를 제공_ 하며, _이 연산자를 사용해 구성한 표현식의 타입 또한 `Column`_ 클래스다.

```scala
'body
// Symbol = 'body

'body contains "Italiano"
// org.apache.spark.sql.Column = contains(body, Italiano)

postsIdBody.filter('body contains "Italiano").count
// Long = 46

('postTypeId === 1) and ('acceptedAnswerId isNull)
// org.apache.spark.sql.Column = ((postTypeId = 1) AND (acceptedAnswerId IS NULL))

postsDf.filter(('postTypeId === 1) and ('acceptedAnswerId isNull))
// 채택된 답변이 없는 질문만 필터링
```

[전체 `Column` 연산자](http://spark.apache.org/docs/2.0.0/api/scala/index.html#org.apache.spark.sql.Column)

`limit` 함수로 _DataFrame 상위 n개 로우를 선택_ 할 수 있다.

```scala
val firstTenQs = postsDf.filter('postTypeId === 1).limit(10)
```

#### 5.1.2.3 칼럼을 추가하거나 칼럼 이름 변경

* `withColumnRenamed`: 칼럼 이름 변경
* `withColumn`: 칼럼 이름과 `Column` 표현식으로 신규 칼럼 추가

```scala
// withColumnRenamed
val firstTenQsRn = firstTenQs.withColumnRenamed("ownerUserId", "owner")

// withColumn
// 포스트 유형이 질문인 것 중 점수당 조회수를 게산하여 ratio라는 컬럼으로 추가하고, 특정 점수보다 낮은 질문 걸러내기
postsDf.filter('postTypeId === 1).withColumn("ratio", 'viewCount / 'score).where('ratio < 35).show()
```

#### 5.1.2.4 데이터 정렬

`orderBy`와 `sort`는 한 개 이상의 칼럼 이름 또는 `Column` 표현식을 받고 이를 기준으로 데이터 정렬한다. `Column` 클래스에 `asc`나 `desc` 설정할 수 있으며, 기본적으로 `asc`

### 5.1.3 SQL 함수로 데이터 연산 수행

스파크 SQL은 데이터에 연산을 수행할 수 있는 함수를 지원하며 _DataFrame API_ 나 _SQL 표현식_ 으로 사용할 수 있다.

스파크 SQL 함수의 네 가지 카테고리

* 스칼라 함수: 각 로우의 단일 칼럼 또는 여러 칼럼 값을 게산해 단일 값을 반환
* 집계 함수: 로우의 그룹에서 단일 값을 계산
* 윈도 함수: 로우의 그룹에서 여러 결과 값을 계산
* 사용자 정의 함수: 커스텀 스칼라 함수 또는 커스텀 집계 함수

#### 5.1.3.1 내장 스칼라 함수와 내장 집계 함수

스칼라 함수와 집계 함수는 모두 `org.apache.spark.sql.functions` 객체에 위치하므로 아래 코드로 모든 함수를 임포트할 수 있다. \(스파크 셸에서는 자동으로 해줌\)

```scala
import org.apache.spark.sql.functions._
```

스칼라 SQL 함수 예시

* 수학 계산: `abs`, `exp`, `hypot`, `log`, `cbrt` ...
* 문자열 연산: `substring`, `length`, `trim`, `concat` ...
* 날짜 및 시간 연산: `year`, `date_add` ...

집계 함수는 _보통 `groupBy`와 함께_ 쓰지만 `select`나 `withcolumn` 메서드에 사용하면 _전체 데이터셋을 대상으로 값을 집계_ 할 수 있다.

스파크 집계 함수 예시

* `min`, `max`, `count`, `avg`, `sum` ...

가장 오랜 시간 논의한 질문을 찾아보는 예제

```scala
postsDf.filter('postTypeId === 1)
    .withColumn("activePeriod", datediff('lastActivityDate, 'creationDate))
    .orderBy('activePeriod desc)
    .head // 첫 번째 Row
    .getString(3) // body column
    .replace("&lt;", "<")
    .replace("&gt;", ">")
```

모든 질문을 대상으로 점수의 평균값, 최댓값, 질문의 총수 구하기

```scala
postsDf.select(avg('score), max('score), count('score)).show
```

#### 5.1.3.2 윈도 함수

집계 함수와 유사하지만, _로우들을 단일 결과로만 그루핑하지 않는다._ 윈도 함수에는 _움직이는 그룹,_ 즉 프레임\(frame\)을 정의한다. 프레임은 윈도 함수가 _현재 처리하는 로우와 관련된 다른 로우 집합_ 으로 정의하며, 이 집합을 현재 로우 계산에 활용할 수 있다.

윈도 함수를 사용하려면 집계 함수나 랭킹 함수, 분석 함수 등을 사용해 `Column` 정의를 구성한 후, 윈도 사양\(`WindowSpec` 객체\)을 생성하고, 이를 `Column`의 `over` 함수에 인수로 전달한다. `over` 함수는 이 윈도 사양을 사용하는 _윈도 칼럼_ 을 정의해 반환한다.

윈도 사양을 만드는 세 가지 방법

* `partitionBy` 메서드를 사용해 단일 칼럼 또는 복수 칼럼을 파티션의 분할 기준으로 지정
* `orderBy` 메서드로 파티션의 로우를 정렬할 기준을 지정 \(데이터셋 전체 대상\)
* `partitionBy`와 `orderBy` 둘 다 사용

혹은 `rowBetween(from, to)` 함수나 `rangeBetween(from, to)` 함수를 사용해 프레임에 포함될 로우를 제한할 수 있다.

```scala
import org.apache.spark.sql.expressions.Window // 윈도우 함수를 사용하기 위한 임포트

// 사용자가 올린 질문 중 최고 점수를 계산하고, 해당 사용자가 게시한 다른 질문의 점수와 최고 점수 간 차이를 출력하는 예제
postsDf.filter('postTypeId === 1)
    .select('ownerUserId, 'acceptedAnswerId, 'score
        , max('score).over(Window.partionBy('ownerUserId)) as "maxPerUser")
    .withColumn("toMax", 'maxPerUser - 'score).show(10)

// 질문 생성 날짜를 기준으로 질문자가 한 바로 전 질문과 바로 다음 질문의 ID를 질문별로 출력하는 예제
postsDf.filter('postTypeId === 1)
    .select('ownerUserId, 'id, 'creationDate,
        lag('id, 1).over(Window.partionBy('owerUserId).orderBy('creationDate)) as "prev",
        lead('id, 1).over(Window.partionBy('owerUserId).orderBy('creationDate)) as "next")
    .orderBy('owerUserId, 'id).show(10)
```

#### 5.1.3.3 사용자 정의 함수

스파크 SQL에서 지원하지 않는 특정 기능이 필요할 때, _사용자 정의 함수\(UDF\)_ 를 사용할 수 있다. 사용자 정의 함수는 `functions` 객체의 `udf` 함수로 생성한다. `udf`는 칼럼 여러 개를 인수로 받는데, 0~10개까지 받을 수 있다.

```scala
// 질문에 달린 태그 개수 세는 예제
// 태그 개수 세는 udf 정의
val countTags = udf((tags: String) => "&lt;".r.findAllMatchIn(tags).length)

// udf 함수 대신 아래와 같이 사용할 수도 있음
// 사용자 정의 함수를 원하는 이름으로 등록하고, SQL 표현식에서도 이 함수를 사용할 수 있다.
val countTags = spark.udf.register("countTags", (tags: String) => "&lt;".r.findAllMatchIn(tags).length)

// UDF로 Column 정의를 생성해 select 문에 전달
postsDf.filter('postTypeId === 1)
    .select('tags, countTags('tags) as "tagCnt")
    .show(10, false) // false는 show 메서드가 출력된 문자열을 중략하지 않도록 설정
```

### 5.1.4 결측 값 다루기

`null`이거나 비어 있거나, 이에 준하는 문자열 상수\(ex: N/A 또는 unknwon\)를 포함하는 데이터의 경우 사용하기 전에 정체해야 할 때도 있다. 이럴 때 DataFrame의 `na` 필드로 제공되는 `DataFrameNaFunctions`를 활용해 처리할 수 있다.

이런 결측값 처리 방법엔 크게 세 가지가 있다. 1. 결측 값을 가진 로우를 제외 2. 결측 값 대신 다른 상수를 채워 넣기 3. 결측 값에 준하는 특정 칼럼 값을 다른 상수로 치환

`drop`을 이용해 특정 로우를 제외하는 예시

```scala
// 칼럼 하나 이상에 null이나 NaN 값을 가진 모든 로우 제외
val cleanPosts = postsDf.na.drop()
// 위와 동일한 결과. any는 어떤 칼럼에서든 null값을 발견하면 해당 로우를 제외
val cleanPosts = postsDf.na.drop("any")
// 모든 칼럼이 null인 로우만 제외
val cleanPosts = postsDf.na.drop("all")
// 특정 칼럼을 지정할 수도 있음
val cleanPosts = postsDf.na.drop(Array("acceptedAnserId"))
```

`fill`을 이용해 `null` 또는 `NaN` 값을 `double` 상수 또는 문자열 상수로 채우는 예시

```scala
// 숫자 타입 컬럼의 결측 값을 0으로 대체
postsDf.na.fill(0)
// viewCount 컬럼의 결측 값을 0으로 대체
postsDf.na.fill(0, Seq("viewCount"))
// 칼럼 이름과 대체 값을 Map 형태로 전달
postsDf.na.fill(Map("viewCount" -> 0))
```

`replace`를 이용해 특정 칼럼의 특정 값을 다른 값으로 치환하는 예시

```scala
// post id가 1177인 데이터를 3000번으로 정정
val postsDfCorrected = postsDf.na.replace(Array("id", "acceptedAnswerId"), Map(1177 -> 3000))
```

### 5.1.5 DataFrame을 RDD로 변환

DataFrame은 RDD기반이므로 DataFrame을 RDD로 변환하는 것은 간단하다. DataFrame의 `rdd` 필드로 하부 RDD에 접근할 수 있으며, 이 `rdd` 필드의 평가 또한 지연\(lazily evaluated\)된다. 결과는 `Row`요소로 구성된 RDD이며 `Row` 클래스는 칼럼 번호로 칼럼 값을 가져올 수 있는 다양한 함수 `getString(index)`, `getInt(index)`, `getMap(index)` 등을 제공한다.

```scala
val postsRdd = postsDf.rdd
// postsRDD: org.apache.spark.rdd.RDD[org.apache.spark.sql.Row]
```

DataFrame 데이터와 파티션을 `map`이나 `flatMap`, `mapPartitions` 등의 변환 연산자로 매핑하면 _실제 매핑 작업은 하부 RDD에서 실행_ 되므로, 연산의 결과는 새로운 DataFrame이 아니라 **새로운 RDD**가 된다.

DataFrame 변환 연산자는 스키마를 변경할 수 있으며, RDD의 `Row` 객체를 다른 타입으로 변환할 수 있다. 하지만 스키마가 변경되지 않았을 경우 RDD를 다시 DataFrame으로 자동으로 변환할 수 있지만, 스키마가 변경된 경우는 자동으로 변환할 수 없다.

### 5.1.6 데이터 그루핑

DataFrame의 그루핑은 `groupBy` 함수로 시작한다. 이 함수는 _칼럼 이름 또는 `Column` 객체의 목록을 받고 `GroupedData` 객체를 반환_ 한다. `GroupedData`는 지정된 칼럼 값이 모두 동일한 로우 그룹들을 표현한 객체이며, 이 로우 그룹에 대한 집계 함수 \(`count`, `sum`, `max`, `min`, `avg`\)를 제공한다.

```scala
postsDf.groupBy('ownerUserId, 'tags, 'postTypeId).count.orderBy('ownerUserId desc).show(10)
// agg 함수를 사용해 서로 다른 칼럼의 여러 집계 연산을 한꺼번에 수행
// 사용자별로 가장 마지막 포스트 수정 날짜와 최고 점수 산출
postsDf.groupBy('ownerUserId).agg(max('lastActivityDate), max('score)).show(10)
// 아래처럼 Map을 이용해 칼럼 이름과 집계 함수 이름을 매핑해 전달할 수도 있음
postsDf.groupBy('ownerUserId).agg(Map("lastActivityDate" -> "max", "score" -> "max")).show(10)
// 칼럼 표현식을 사용하면 여러 칼럼의 표현식을 연결할 수 있음
postsDf.groupBy('ownerUserId).agg(max('lastActivityDate), max('score).gt(5)).show(10)
```

#### 5.1.6.1 사용자 정의 집계 함수

커스텀 집계 함수를 직접 정의해 사용할 수도 있으며, 일반적으로 `org.apache.spark.sql.expressions.UserDefinedAggregateFunction` 추상 클래스를 상속해 사용한다.

* [공식 API 문서](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.expressions.UserDefinedAggregateFunction)
* [자바 구현 예제](https://github.com/apache/spark/blob/master/sql/core/src/test/java/test/org/apache/spark/sql/MyDoubleAvg.java)

#### 5.1.6.2 rollup 및 cube

`groupBy`는 _지정된 칼럼들이 가질 수 있는 모든 조합별_ 로 집계 연산을 수행하지만, `rollup`과 `cube`는 _지정된 칼럼의 부분 집합을 추가로 사용_ 해 집계 연산을 수행

* `cube`: 칼럼의 모든 조합\(Combination\)을 대상
* `rullup`: 지정된 칼럼 순서를 고려한 순열\(Permutation\)

```scala
val smplDf = postsDf.where('ownerUserId >= 13 and 'ownerUserId <= 15)
smplDf.groupBy('ownerUserId, 'tags, 'postTypeId).count.show()
smplDf.rollup('ownerUserId, 'tags, 'postTypeId).count.show()
smplDf.cube('ownerUserId, 'tags, 'postTypeId).count.show()
```

스파크의 주요 환경 설정과 설정 변수들은 런타임에 변경할 수 없지만, 스파크 SQL의 환경 설정 변수는 런타임에 변경 가능. SQL의 `SET` 명령을 사용하거나 `SparkSession`의 `conf`필드로 제공되는 `RuntimeConfig`객체의 `set` 메서드를 호출

```scala
spark.sql("SET spark.sql.caseSensitive=true")
spark.conf.set("spark.sql.caseSensitive", "true")
```

### 5.1.7 데이터 조인

RDD 조인 방법과 크게 다르지 않다. 1. 한 DataFrame의 `join` 함수에 다른 DataFrame 전달 2. 하나 이상의 칼럼 이름 또는 `Column` 정의를 조인 기준으로 지정

* 이름을 사용할 경우 각 DataFrame에 동일한 이름의 칼럼이 있어야 함
* 칼럼 이름이 다를 경우 `Column` 정의 사용해야 하며 이 경우 세 번째 인수에 조인 유형\(`inner`, `outer`, `left_outer`, `right_outer`, `leftsemi`\) 지정 가능

```scala
val itVotesRaw = sc.textFile("first-edition/ch05/italianVotes.csv").map(x => x.split("~"))
val itVotesRows = itVotesRaw.map(row => Row(row(0).toLong, row(1).toLong, row(2).toInt, Timestamp.valueOf(row(3))))
val votesSchema = StructType(Seq(
    StructField("id", LongType, false),
    StructField("postId", LongType, false),
    StructField("voteTypeId", IntegerType, false),
    StructField("creationDate", TimestampType, false)))
val votesDf = spark.createDataFrame(itVotesRows, votesSchema)

// join!
// 두 DataFrame 모두 id를 포함하므로 Column 객체를 생성에 명시적으로 어느 쪽의 id 칼럼인지 알려줘야 하지만, postId는 votesDf에만 있으므로 Symbol을 이용해 간단하게 작성 가능
val postsVotes = postsDf.join(votesDf, postsDf("id") === 'postId) // inner join
val postsVotesOuter = postsDf.join(votesDf, postsDf("id") === 'postId, "outer") // outer join
```

## 5.2 DataFrame을 넘거 Dataset으로

`Dataset`의 핵심 아이디어

> 사용자에게 도메인 객체\(domain object\)에 대한 변환 연산을 손쉽게 표현할 수 있는 API를 지원함과 동시에, 스파크 SQL 실행 엔진의 빠른 성능과 높은 안정성을 제공하는 것

다시 말해 일반 자바 객체를 `Dataset`에 저장할 수 있고, 스파크 SQL의 _텅스텐 엔진_ 과 _카탈리스트 최적화_ 를 활용할 수 있다는 의미

스파크 2.0에서는 DataFrame을 `Row` 객체로 구성된 `Dataset` 즉, `Dataset[Row]`로 구현했다. 또 DataFrame의 `as` 메서드를 사용해 DataFrame을 `Dataset`으로 변환할 수 있다.

_**as 메서드 시그니처**_

```scala
def as[U : Encoder]: Dataset[U]
```

## 5.3 SQL 명령

DataFrame DSL 외에 SQL 명령을 사용할 수도 있다. 스파크 SQL은 SQL 명령을 DataFrame 연산으로 변환한다. 혹은 스파크 쓰리프트 서버로 연결해 일반 SQL 애플리케이션에서도 스파크에 접속할 수 있다. 스파크는 스파크 전용 SQL과 하이브 쿼리 언어\(Hive Query Language, HQL\)를 지원한다.

### 5.3.1 테이블 카탈로그와 하이브 메타스토어

DataFrame을 테이블로 등록해 두면 스파크 SQL 쿼리에서 테이블을 참조할 수 있다. 스파크는 테이블 정보를 **테이블 카탈로그\(table catalog\)** 에 저장한다.

하이브 지원 기능이 없는 스파크를 사용하면 테이블 정보를 인-메모리 Map에 저장해서 스파크 세션이 종료되면 사라지지만, 하이브를 지원하는 스파크에서는 하이브 메타스토어\(영구적인 데이터베이스\)에 저장하므로 _스파크를 재시작해도 테이블 정보를 유지_ 한다.

#### 5.3.1.1 테이블을 임시로 등록

`createOrReplaceTempView` 메서드를 사용해 하이브 지원 스파크이든 미지원 스파크이든 _임시로 테이블 정의를 저장_ 할 수 있다.

```scala
postsDf.createOrReplaceTempView("posts_temp") // posts_temp 이름으로 질의 수행 가능
```

#### 5.3.1.2 테이블을 영구적으로 등록

`HiveContext`는 _metastore\_db_ 폴더 아래 로컬 작업 디렉터리에 Derby 데이터베이스를 생성\(존재하면 재사용\)한다. 작업 디렉터리 위치를 변경하려면 _hive-site.xml_ 파일의 `hive.metastore.warehouse.dir`에 원하는 경로를 지정한다.

```scala
// DataFrame을 영구 테이블로 등록
postsDf.write.saveAsTable("posts")
votesDf.write.saveAsTable("votes")

// 기존에 테이블이 존재한다면 덮어쓰기
postsDf.write.mode("overwrite").saveAsTable("posts")
votesDf.write.mode("overwrite").saveAsTable("votes")
```

#### 5.3.1.3 스파크 테이블 카탈로그

스파크 2.0부터 `SparkSession`의 `catalog` 필드로 제공되는 `Catalog` 클래스를 이용해 _테이블 카탈로그 관리 기능_ 을 사용할 수 있다.

```scala
scala> spark.catalog.listTables().show()
+----------+--------+-----------+---------+-----------+
|      name|database|description|tableType|isTemporary|
+----------+--------+-----------+---------+-----------+
|     posts| default|       null|  MANAGED|      false|
|     votes| default|       null|  MANAGED|      false|
|posts_temp|    null|       null|TEMPORARY|       true|
+----------+--------+-----------+---------+-----------+
```

`listTables` 메서드가 `Table` 객체의 `Dataset`을 반환하므로 `show` 메서드 사용 가능

* `isTemporary`: 영구 테이블/임시 테이블
* `tableType`
  * `MANAGED`: 스파크가 테이블의 데이터까지 관리. 사용자의 홈 디렉터리 아래 _spark\_warehouse_ 폴더에 저장된다. `spark.sql.warehouse.dir`를 변경하면 다른 위치에 저장 가능
  * `EXTERNAL`: 다른 시스템\(예: RDBMS\)으로 관리
* `database`: 기본 데이터베이스 이름은 `default`

```scala
scala> spark.catalog.listColumns("votes").show() // 특정 테이블의 칼럼 정보 조회
+------------+-----------+---------+--------+-----------+--------+
|        name|description| dataType|nullable|isPartition|isBucket|
+------------+-----------+---------+--------+-----------+--------+
|          id|       null|   bigint|    true|      false|   false|
|      postid|       null|   bigint|    true|      false|   false|
|  votetypeid|       null|      int|    true|      false|   false|
|creationdate|       null|timestamp|    true|      false|   false|
+------------+-----------+---------+--------+-----------+--------+
scala> spark.catalog.listFunctions.show() // SQL 함수 목록 조회
```

#### 5.3.1.4 원격 하이브 메타스토어 설정

스파크는 기존에 설치된 하이브 메타스토어 데이터베이스나 스파크가 사용할 새로운 데이터베이스를 구축해 이를 _원격 메타스토어 데이터베이스_ 로 지정할 수 있다. 이는 스파크 _conf_ 디렉터리 아래에 있는 _hive-site.xml_ 파일에 설정한다.

원격 하이브 메타스토어를 사용하기 위한 설정

* `javax.jdo.option.ConnectionURL`: JDBC 접속 URL
* `javax.jdo.option.ConnectionDriverName`: JDBC 드라이버의 클래스 이름
* `javax.jdo.option.ConnectionUserName`: 데이터베이스 사용자 이름
* `javax.jdo.option.ConnectionPassword`: 데이터베이스 사용자 암호

설정된 JDBC 드라이버는 스파크 드라이버와 모든 실행자의 클래스패스에 추가되어야 한다.

### 5.3.2 SQL 쿼리 실행

DataFrame을 테이블로 등록하면 `SparkSession`의 `sql` 함수로 SQL 표현식을 사용해 데이터에 질의를 실행할 수 있다. 이 _질의의 결과는 DataFrame_ 이다.

```scala
import spark.sql // spark shell은 임포트되어 있음
val resultDf = sql("select * from posts")
```

#### 5.3.2.1 스파크 SQL 셸 사용

스파크는 `spark-sql` 명령으로 실행되는 별도의 SQL 셸을 제공한다. 실행 시에는 `spark-shell`이나 `spark-submit`의 인수들을 똑같이 전달할 수 있으며, 이 외에도 SQL 셸 고유의 인수를 전달할 수 있다. 아무 인수를 전달하지 않으면 로컬 모드로 시작한다.

스파크 셸이 Derby 메타스토어에 잠금을 걸기 때문에 `spark-sql` 셸을 실행하고 싶다면 이미 실행 중인 스파크 셸은 종료해야 한다. SQL 셸에서는 스파크 셸에서 작성하는 것돠 동일하게 SQL 명령을 입력할 수 있지만, 표현식 마지막에 세미콜론\(;\)을 붙여야 한다.

```sql
spark-sql> select substring(title, 0, 70) from posts where postTypeId = 1 order by creationDate desc limit 3;
```

혹은 `spark-sql`의 `-e` 인수를 이용해 셸을 실행하지 않고 SQL 쿼리를 실행할 수 있다.

```bash
$ spark-sql -e "select substring(title, 0, 70) from posts where postTypeId = 1 order by creationDate desc limit 3"
```

* `-f` 인수: 파일에 저장된 SQL 명령 실행
* `-i` 인수: 초기화 SQL 파일 지정

### 5.3.3 쓰리프트 서버로 스파크 SQL 접속

스파크 쓰리프트라는 JDBC\(or ODBC\) 서버를 이용해 _원격에서 사용자의 쿼리를 받아 SQL 명령을 실행_ 할 수 있다. 스파크 쓰리프트는 스파크 클러스터에서 구동되어 전달된 SQL 쿼리를 DataFrame으로 변환한 후, RDD 연산으로 변환해 실행하며, 결과를 JDBC 프로토콜로 반환한다.

#### 5.3.3.1 쓰리프트 서버 시작

스파크 _sbin_ 디렉터리의 `start-thriftserver.sh` 명령으로 시작하며 `spark-sql` 명령과 마찬가지로 `spark-shell` 및 `spark-submit` 명령의 인수를 동일하게 사용할 수 있다.

PostgreSQL을 원격 메타스토어 데이터베이스로 설정한 경우 스파크 쓰리프트 서버를 시작하는 방법

```bash
$ sbin/start-thriftserver.sh --jars /usr/share/java/postgresql-jdbc4.jar
```

#### 5.3.3.2 비라인으로 쓰리프트 서버 접속

스파크 _bin_ 디렉터리의 `beeline`은 _쓰리프트 서버로 접속할 수 있는 하이브의 명령줄 셸_ 이다. 서버 접속이 성공하면 비라인 명령 프롬프트에 HQL 명령을 입력해 쿼리를 실행할 수 있다.

#### 5.3.3.3 써드-파티 JDBC 클라이언트로 접속

오픈소스 자바 SQL 클라이언트인 Squirrel SQL을 사용해 스파크 쓰리프트 서버로 접속하는 예제

### 5.4 DataFrame을 저장하고 불러오기

스파크는 다양한 데이터 소스\(다양한 파일 포맷 및 데이터베이스\)를 지원하며, 관계형 데이터베이스와 스파크를 연동하는 `Dialect` 클래스\(JDBC의 `JdbcType` 클래스와 스파크 SQL의 `DataType` 클래스를 매핑하는 클래스\)를 제공한다.

스파크는 메타스토어에 데이터 저장 위치와 저장 방법을 보관하며 _실제 데이터는 데이터 소스에 저장_ 된다.

#### 5.4.1 기본 데이터 소스

스파크가 기본으로 지원하는 데이터 포맷들

#### 5.4.1.1 JSON

XML을 대체하는 경량\(lightweight\) 데이터 포맷. 스파크는 _JSON 스키마를 자동으로 유추_ 할 수 있으므로 외부 시스템과 데이터를 주고받는 포맷으로 매우 적합하지만 영구적으로 저장하는 데 사용하기에는 저장 효율이 떨어진다.

#### 5.4.1.2 ORC

Optimized Row Columnar. 하둡의 표준 데이터 저장 포맷인 RCFile보다 효율적으로 하이브의 데이터를 저장할 수 있는 파일 포맷. 칼럼형 포맷이며 로우 데이터를 스트라이프 여러 개로 묶어서 저장한다.

#### 5.4.1.1 Parquet

불필요한 의존 관계를 가지지 않으며 특정 프레임워크에 종속되지 않도록 설계되었다. ORC와 마찬가지로 칼럼형 파일 포맷이며 칼럼별로 압축 방식을 지정해서 데이터 압축 가능. 스파크가 기본으로 사용하는 포맷.

### 5.4.2 데이터 저장

DataFrame은 `write` 필드로 제공되는 `DataFrameWriter` 객체를 사용해 저장한다.

```scala
postsDf.write.saveAsTable("posts")
```

* `saveAsTable`, `insertInto`: 데이터를 하이브 테이블로 저장하며 이 과정에서 메타스토어 활용. 하이브를 지원하지 않는 스파크인 경우 하이브 테이블 대신 임시 테이블에 저장
* `save`: 데이터를 파일에 저장

#### 5.4.2.1 Writer 설정

* `format`: 데이터를 저장할 파일 포맷, 즉 데이터 소스 이름 지정.
  * `spark.sql.sources.default`: 기본으로 사용할 데이터 소스 지정\(기본: _parquet_\)
* `mode`: 지정된 테이블이나 파일이 이미 존재할 때 대응 방식
  * `overwrite`: 기존 데이터 덮어쓰기
  * `append`: 기존 데이터에 추가
  * `ignore`: 저장 요청을 무시. 아무 것도 하지 않음
  * `error`: 예외를 던짐 \(기본 설정\)
* `option`, `options`: 데이터 소스를 설정할 매개변수 이름과 변수 값 추가\(`options`는 매개변수 이름-값 쌍을 `Map`으로\)
* `partitionBy`: 복수의 파티션 칼럼 지정

```scala
// DataFrameWriter는 Builder pattern으로 구현되었음
postsDf.write.format("orc").mode("overwrite").option(...)
```

#### 5.4.2.2 saveAsTable 메서드로 데이터 저장

데이터를 하이브 테이블에 저장하고, 테이블을 하이브 메타스토어에 등록\(하이브 미지원 버전인 경우 임시 테이블에 등록\)

데이터를 하이브 SerDe\(Serialization/Deserialization 클래스\)가 존재하는 포맷으로 저장하면 스파크에 내장된 하이브 라이브러리가 저장 작업을 수행한다. 하지만 SerDe가 없는 포맷으로 저장하면 스파크는 저장할 포맷을 따로 선택한다.

#### 5.4.2.3 insertInto 메서드로 데이터 저장

하이브 메타스토어에 존재하는 테이블을 지정해야 하며, 이 테이블과 새로 저장할 DataFrame 스키마가 서로 같아야 한다. 그렇지 않으면 예외가 발생한다. 또 테이블이 이미 존재하고 포맷도 결정했기 때문에 `format`과 `option` 매개변수는 사용하지 않는다. 만약 `mode`에 `overwrite`를 지정하면 기존 테이블 내용을 삭제하고 DataFrame으로 테이블을 대체한다.

#### 5.4.2.4 save 메서드로 데이터 저장

데이터를 파일 시스템에 저장하며 저장할 경로\(HDFS, S2, 로컬 파일 시스템 등 URL 경로\)를 전달해야 한다. 로컬 경로인 경우 스파크 실행자가 구동 중인 모든 머신의 로컬 디스크에 파일을 저장한다.

#### 5.4.2.5 단축 메서드로 데이터 저장

`DataFrameWriter`는 데이터를 기본 데이터 소스\(json, orc, parquet\)로 저장하는 단축 메서드 제공. 내부적으로 데이터 소스 이름으로 `format`을 호출한 후, `save` 메서드를 호출한다. 즉 하이브 메타스토어를 활용하지 않는다.

#### 5.4.2.6 jdbc 메서드로 관계형 데이터베이스에 데이터 저장

`DataFrameWriter`의 `jdbc` 메서드를 사용해 DataFrame을 관계형 데이터베이스에 저장할 수 있다.

```scala
val props = new java.util.Properties()
props.setProperty("user", "user")
props.setProperty("password", "password")
postsDf.write.jdbc("jdbc:postgresql://postgresrv/mydb", "posts", props) // URL, 테이블 이름, 접속에 필요한 정보를 담은 props
```

### 5.4.3 데이터 불러오기

`SparkSession`의 `read` 필드로 제공되는 `org.apache.spark.sql.DataFrmaeReader` 객체로 데이터를 DataFrame으로 읽어올 수 있다. `format`, `option`, `options` 그리고 `schema` 함수를 사용할 수 있다.

`DataFrameReader`에도 단축 메서드가 있는데 `DataFrameWriter`와 유사하게 `format`을 호출한 후 `load` 메서드를 호출한다.

혹은 `table` 함수로 하이브 메타스토어에 등록된 테이블을 읽을 수 있다.

#### 5.4.3.1 jdbc 메서드로 관계형 데이터베이스에서 데이터 불러오기

`DataFrameWriter`의 `jdbc` 메서드와 유사하지만, 여러 조건\(WHERE 절에 쓸 표현식\)을 사용할 수 있다.

#### 5.4.3.2 sql 메서드로 등록한 데이터 소스에서 데이터 불러오기

임시 테이블을 등록하면 SQL 쿼리에서 기존 데이터 소스 참조 가능

## 5.5 카탈리스트 최적화 엔진

Catalyst optimizer는 DSL과 SQL 표현식을 RDD 연산으로 변환하며, 사용자는 카탈리스트를 확장해 다양한 최적화를 추가 적용할 수 있다.

카탈리스트 엔진 최적화 과정 1. 파싱 완료된 논리 실행 계획 생성 2. 쿼리가 참조하는 테이블 이름, 칼럼 이름, 클래스 고유 이름 등 릴레이션 존재 여부 검사 3. 분석 완료된 논리 실행 계획 생성 4. 하위 레벨 연산을 재배치하거나 결합하는 등 최적화 시도 5. 최적화된 논리 실행 계획 생성 6. 실제 물리 실행 계획 작성

#### 5.5.1.1 실행 계획 검토

DataFrame의 `explain` 메서드로 최적화 결과를 확인하고 실행 계획을 검토할 수 있다. 출력된 결과는 아래쪽에서 위쪽으로 읽으면 된다.

혹은 스파크 드라이버가 실행 중인 머신의 4040 포트로 웹 UI에 접속해 확인할 수도 있다.

```scala
val postsFiltered = postsDf.filter('postTypeId === 1).withColumn("ratio", 'viewCount / 'score).where('ratio < 35)

postsFiltered.explain() // 물리 실행 계획만 조회
postsFiltered.explain(true) // 논리 및 물리 실행 계획 조회
```

#### 5.5.1.2 파티션 통계 활용

카탈리스트는 각 칼럼의 통계를 활용해 필터링 작업 중 일부 파티션을 건너뛰고 작업 성능을 추가로 최적화한다. 칼럼 통계는 DataFrame을 메모리에 캐시하면 자동으로 계산된다.

## 5.6 텅스텐 프로젝트의 스파크 성능 향상

객체를 이진수로 인코딩하고 메모리에서 직접 참조하는 방식으로 메모리 관리 성능을 대폭 개선했다.

두 가지 메모리 관리 모드

* 오프-힙 할당: `sum.misc.Unsafe` 클래스를 사용해 C 언어처럼 주소로 직접 할당/해제. long 배열에 저장하긴 하지만 JVM이 관여하지 않고\(GC에서도 제외\) 스파크가 직접 관리
* 온-힙 할당: 이진수로 인코딩한 객체를 JVM이 관리하는 대규모 long 배열에 저장

