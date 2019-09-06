# Chapter 8. 속성 기반 검사

개발자는 프로그램의 행동 방식, 제약만을 지정하고, 테스트를 하기 위한 _실제 입력값은 테스팅 도구가 임의로 자동 생성해서 프로그램이 명시된 대로 작동하는지 확인_ 하는 것을 속성 기반 테스팅이라고 부른다. 다시 말해 프로그램 행동 방식의 서술과 테스트 케이스의 생성을 분리한다. 따라서 개발자는 프로그램의 행동 방식과 테스트 케이스에 대한 고수준의 제약을 작성하는 데에만 집중하면 된다.

## 8.1 속성 기반 검사의 간략한 소개

```scala
// 0에서 100사이의 정수들의 목록을 생성
val intList: Gen[List[Int]] = Gen.listOf(Gen.choose(0, 100))

// 만족해야만 하는 속성으로 && 으로 연결된 두 속성 모두 만족해야 한다
val prop =
    forAll(intList)(ns => ns.reverse.reverse == ns) &&
    forAll(intList)(ns => ns.headOption == ns.reverse.lastOption)

// 명백히 거짓인 속성 ? 모두 같은 숫자로 이루어진 List이면...
val failingProp = forAll(intList)(ns => ns.reverse == ns)
```

여기서 `Gen[List[Int]]`는 `List[Int]`형태의 _테스트 데이터를 생성하는 방법을 아는 어떤 것_ 이다. 함수 `forAll`은 `Gen[A]`형식의 생성기와 `A => Boolean` 형식의 predicate를 조합해서 _하나의 속성_ 을 만들어 낸다. 이 속성은 생성기가 생성한 모든 값이 predicate를 만족해야 함을 나타낸다.

```scala
prop.check
// OK, passes 100 tests.

failingProp.check
// ! Falsified after 6 passes tests.
> ARG_0: List(0, 1)
```

속성을 점검한 예시로 100개의 테스트 케이스가 `prop`를 만족했으며, `failingProp`는 만족하지 못했음을 알 수 있다. 예제에서 보이다시피 실패한 테스트 데이터도 출력되었다.

속성 기반 검사 라이브러리들이 갖춘 유용한 기능들

* Test case minimization

  어떤 속성이 크기가 10인 목록에 대해 실패한다면, 프레임워크는 그 검사에 실패하는 가장 작은 목록에 도달할 때까지 점점 더 작은 목록으로 검사를 수행하고, 찾아낸 최소의 목록을 보고한다.

* Exhaustive test case generation

  `Gen[A]`가 생성할 수 있는 모든 값의 집합을 정의역이라고 부르는데, 이 정의역이 충분히 작다면 표본 값을 생성하는 대신 전수 검사를 할 수 있다.

## 8.2 자료 형식과 함수의 선택

속성 기반 검사를 위한 라이브러리를 설계해보자

### 8.2.1 API의 초기 버전

이전 예제를 보자. `Gen.choose(0, 100)`은 `Gen[Int]`를 리턴하고, `Gen.listOf`는 `Gen[Int]`를 받아 `Gen[List[Int]]`를 돌려주는 함수임을 알 수 있다. 그러므로 `listOf`는 아래와 같이 정의할 수 있음을 알 수 있다.

혹은 테스트 데이터의 길이를 명시적으로 지정하는 함수를 만들 수도 있다. 하지만 길이를 명시적으로 지정한 테스트 케이스만을 사용해야 한다면 대부분의 경우 검사 실행 함수가 유연하게 테스트를 수행할 수 없을 것이다.

```scala
def listOf[A](a: Gen[A]): Gen[List[A]]
// 혹은 테스트 데이터의 길이를 명시적으로 지정할 수 있는 API
def listOfN[A](n: Int, a: Gen[A]): Gen[List[A]]
```

`forAll` 함수는 `Gen[List[Int]]`와 그에 대한 predicate 역할을 하는 `List[Int] => Boolean` 함수를 받는다.

```scala
def forAll[A](a: Gen[A])(f: A => Boolean): Prop
```

그리고 `Prop`는 `&&` 연산자를 지원한다.

```scala
trait Prop { def &&(p: Prop): Prop }
```

### 8.2.2 속성의 의미와 API

이제 API의 형식과 _의미_ 를 생각해보자

`Prop`을 위한 함수 세 가지

1. 속성을 생성하는 `forAll`
2. 속성들의 합성을 위한 `&&`
3. 속성의 점검을 위한 `check` -&gt; ScalaCheck에서는 콘솔 출력이라는 부수 효과 존재

`check`는 부수 효과가 있으므로 함수 합성의 기저로 사용할 수는 없다. 이 상태에서는 `Prop`을 위한 `&&`을 구현할 수 없다. `check`에 부수 효과가 있으므로, `&&`을 구현하려면 두 `check` 메서드를 모두 실행해야 한다. 하지만 그렇게 된다면, `check` 메서드는 속성 각각의 성공/실패 여부를 출력할 것이다. 여기서 `check`의 부수효과는 큰 문제가 아니며, _정보를 폐기_ 한다는 것이 문제다.

여기서 우리가 `&&`을 이용해 속성을 조합하려면 `check`로부터 뭔가 의미 있는 값\(속성 검사의 성공, 실패 `Boolean`\)을 알 수 있어야 한다. 그렇다면 우리는 AND, OR, NOT, XOR 등의 함수도 `Prop`에 정의할 수 있을 것이다. 그런데 속성이 실패한 경우, 성공한 테스트 케이스는 몇 개이고 실패를 유발한 테스트 케이스는 뭔지 알고 싶지 않을까? 성공했을 때도 몇 개의 테스트 케이스를 통과했는지 알고 싶을 수 있다. 그러므로 `Either`를 사용하기로 하자. 실패의 경우에는 그냥 사람에게 테스트 데이터를 출력하고 검사를 끝내면 된다.

```scala
object Prop {
    type FailedCase = String
    type SuccessCount = Int
}

trait Prop { def check: Either[(FailedCase, SuccessCount) SuccessCount] }
// Test failed: Left((s, n)) = s는 속성이 실패하게 만든 값, n은 실패 이전에 성공한 테스트 수
// Test success: Right(n)
```

### 8.2.3 생성기의 의미와 API

`Gen[A]`는 `A` 형태의 테스트 데이터를 생성하는 방법을 아는 _어떤 것_ 이라고 했다. 여기서 우리는 값을 무작위로 생성하는 방법을 택할 수 있다. 혹은 난수 발생기\(`RNG`\)에 대한 `State` 전이를 감싸는 하나의 형식으로 만들 수도 있다.

```scala
case class Gen[A](sample: State[RNG, A])
```

우리는 어떤 연산들이 **기본 수단**이고 어떤 연산들이 그로부터 **파생된 연산**들인지 이해하고, _작지만 표현력 있는 기본 수단들의 집합을 구하는 데_ 관심이 있다. 주어진 기본 수단들의 집합으로 표현할 수 있는 대상들을 찾는 가장 좋은 방법 하나는, 표현하고자 하는 구체적인 예제들을 선택하고, 그에 필요한 기능성을 조합할 수 있는지를 시험해보는 것이다. 그 과정에서 패턴들을 인식하고, 그 패턴들을 추출해서 조합기를 만들고, 기본 수단들의 집합을 정련해 나간다.

### 8.2.4 생성된 값들에 의존하는 생성기

생성된 값들에 의존하는 생성기를 만들려면 _한 생성기가 다른 생성기에 의존하게 만드는 `flatMap`이 필요_ 하다.

```scala
def flatMap[B](f: A => Gen[B]): Gen[B]
```

### 8.2.5 `Prop` 자료 형식의 정련

현재 `Prop`은 아래와 같은 모양이다.

```scala
trait Prop {
    def check: Either[(FailedCase, SuccessCount), SuccessCount]
}
```

여기엔 _속성이 검사를 통과했다는 결론을 내리는 데 충분한 테스트 케이스의 개수_ 가 빠져 있다. 이 의존성을 추상화하면 아래와 같이 될 것이다.

```scala
type TestCases = Int
type Result = Either[(FailedCase, SuccessCount), SuccessCount]
case class Prop(run: TestCases => Result)
```

여기서 봤을 때 `SuccessCount`가 `Either` 양변에 기록되는데, 사실 어떤 속성이 검사를 통과했다면, 이는 `run`의 `TestCases`의 수\(전체 테스트 수\)와 동일할 것이다. 그러므로 `Either`의 `Right`의 경우 따로 저장할 필요 없는 정보이다.

```scala
type Result = Option[(FailedCase, SuccessCount)]
case class Prop(run: TestCases => result)
```

여기서 `None`은 속성이 검사를 통과했음을 나타낸다. 다시 말해 실패의 부재를 나타내는 용도이다. 하지만 언뜻 보았을 때 의도가 아주 명확하지는 않으므로, 의도를 명확히 나타낼 수 있는 새로운 자료 형식을 만들어보자.

```scala
sealed trait Result {
    def isFalsified: Boolean
}

case object Passes extends Result {
    def isFalsified = false
}

case object Falsified(failure: FailedCase, successes: SuccessCount) extends Result {
    def isFalsified = true
}
```

그럼 이제 `forAll`을 구현할 수 있을까?

```scala
def forAll[A](a: Gen[A])(f: A => Boolean): Prop
```

하지만 우리는 시도할 테스트 케이스의 개수나 테스트 케이스를 생성하는 데 필요한 모든 정보가 필요하다. 그러니 아래와 같이 고치자.

```scala
case class Prop(run: (TestCases, RNG) => Result)
```

그러면 `forAll`는 아래와 같이 구현할 수 있다.

```scala
def forAll[A](as: Gen[A])(f: A => Boolean): Prop = Prop {
    (n, rng) => randomStream(as)(rng).zip(Stream.from(0)).take(n).map {
        case (a, i) => try {
            if (f(a)) Passed else Falsified(a.toString, i)
        } catch { case e: Exception => Falsified(buildMsg(a, e), i) }
    }.find(_.isFalsified).getOrElse(Passed)
}

def randomStream[A](g: Gen[A])(rng: RNG): Stream[A] =
    Stream.unfold(rng)(rng => Some(g.sample.run(rng)))

def buildMsg[A](s: A, e: Exception): String =
    s"test case: $s\n" +
    s"generated an exception: ${e.getMessage}\n" +
    s"stack trace:\n${e.getStackTrace.mkString("\n")}"
```

## 8.3 검례 최소화

검사 실패 시 프레임워크가 해당 검사를 실패하는 _가장 작은 또는 가장 간단한 테스트 데이터_ 를 찾아내는 기법

검례 최소화를 위한 접근 방식 두 가지

* 수축\(shrinking\) - 테스트가 실패했다면, 그 테스트 케이스의 _크기_ 를 점점 줄여가면서 검사를 반복하되, 테스트가 더 이상 실패하지 않으면 멈춘다. 각 자료 형식마다 _개별적인 코드를 작성_ 해야 한다.
* 크기별 생성\(sized generation\) - 처음부터 크기와 복잡도를 점차 늘려가면서 테스트 케이스를 생성한다.

ScalaCheck는 수축을 사용하는데, 우리는 크기별 생성을 사용해보자. 조금 더 간단하고, 어떤 의미로는 더 모듈성이 좋다.

```scala
case class SGen[+A](forSize: Int => Gen[A])
```

하지만 `forAll` 함수에 `SGen`을 그대로 대입할 수는 없다. `SGen`은 크기를 알아야 하지만, `Prop`에는 크기 정보가 주어지지 안는다. 이 정보를 그냥 `Prop`에 추가할 수 있지만, _다양한 크기의 생성기를 실행하는 책임을 `Prop`에 부여_ 하고 싶기 때문에 _최대 크기_ 를 지정하면 된다.

```scala
type MaxSize = Int
case class Prop(run: (MaxSize, TestCases, RNG) => Result)

def forAll[A](g: SGen[A])(f: A => Boolean): Prop = forAll(g(_))(f)

def forAll[A](g: Int => Gen[A])(f: A => Boolean): Prop = Prop {
    (max, n, rng) =>
        val casesPerSize = (n + (max - 1)) / max // 각 크기마다 이만큼의 무작위 테스트 케이스 생성
        val props: Stream[Prop] =
            Stream.from(0).take((n min max) + 1).map(i => forAll(g(i))(f)) // 각 크기마다 하나의 속성을 만든다. 단, 전체 속성 개수가 n을 넘지는 않게 한다.
        val prop: Prop =
            props.map(p => Prop { (max, _, rng) =>
                p.run(max, casesPerSize, rng)
            }).toList.reduce(_ && _) // 모든 속성을 하나로 결합
        prop.run(max, r, rng)
}
```

## 8.4 라이브러리의 사용과 사용성 개선

이제 라이브러리를 실제로 사용해보자. 그러면 전체적인 사용성에서 뭔가 부족한 점을 발견할 수 있을 것이다. 사용성은 대체로 편리한 구문과 공통의 사용 패턴에 적합한 보조 함수들이 갖추어져 있다면 좋다고 할 수 있다.

### 8.4.1 간단한 예제 몇 가지

```scala
// List의 max 행동 방식 명시
val smallInt = Gen.choose(-10, 10)
val maxProp = forAll(listOf(smallInt)) { ns =>
    val max = ns.max
    !ns.exists(_ > max)
}
```

그런데 `Prop`에서 `run`을 호출하는 건 좀 번거롭다. 실행하고 결과를 출력해주는 보조 함수를 도입해보자.

```scala
def run(p: Prop, maxSize: Int = 100, testCases: Int = 100, rng: RNG = RNG.Simple(System.currentTimeMillis)): Unit =
    p.run(maxSize, testCases, rng) match {
        case Falsified(msg, n) =>
            println(s"! Falsified after $n passed tests:\n $msg")
        case Passes =>
            println(s"+ OK, passed $testCases tests.")
    }
```

속성 기반 검사는 코드에 숨어 있는 가정을 드러내는 한 방법이다. 결과적으로 프로그래머가 그런 가정들을 좀 더 _명시적으로 코드에 지정_ 하게 한다.

### 8.4.2 병렬 계산을 위한 검사 모음 작성

7장에서 밝혀낸 병렬 계산에 대해 반드시 성립해야 하는 법칙들을 이 검사 라이브러리도 표현할 수 있을까?

```scala
map(unit(1))(_ + 1) == unit(2)

val ES: ExecutorService = Executors.newCachedThreadPool
val p1 = Prop.forAll(Gen.unit(Par.unit(1)))(i =>
    Par.map(i)(_ + 1)(ES).get == Par.unit(2)(ES).get)
```

아주 깔끔하게는 되지 않는다.

#### 속성의 증명

이 테스트 케이스에 대해서는 `forAll`가 다소 과하게 일반적이다. 이 검사에서는 입력이 가변적이지 않고 구체적인 값이 코드에 박혀 있다. 이런 테스트는 전통적인 unit test만큼 간단해야 한다. 조합기를 하나 추가해보자.

```scala
def check(p: => Boolean): Prop = {
    lazy val result = p // memoization
    forAll(unit(()))(_ => result)
}
```

그런데 이 구현은 단위 생성기를 통해 값을 하나만 생성하는데, 값 자체는 쓰이지 않고 무시 된다. memoization 했다고 해도 검사 실행기는 여러 개의 테스트 케이스를 생성해서 여러 번 점검한다.

그렇다면 테스트 케이스 개수를 무시하는 `Prop`을 구축하는 `check`를 새로 만들어보자.

```scala
def check(p: => Boolean): Prop = Prop { (_, _, _) => // MaxSize, TestCases, RNG
    if (p) Passed else Falsified("()", 0)
}
```

여기선 속성을 한 번만 검사하지만 여전히 "OK, passes 100 tests."라는 출력이 나올 것이다. 이 속성은 반례가 발견되지 않아서 pass된 게 아니고 한 번의 검사에 의해서 증명된 것이다. `Proved`를 만들어보자.

```scala
case object Proved extends Result

def run(p: Prop, maxSize: Int = 100, testCases: Int = 100, rng: RNG = RNG.Simple(System.currentTimeMillis)): Unit =
    p.run(maxSize, testCases, rng) match {
        case Falsified((msg, n)) =>
            println(s"! Falsified after $n passed tests:\n $msg")
        case Passes =>
            println(s"+ OK, passed $testCases tests.")
        case Proved =>
            println(s"+ OK, proved property.")
    }
```

#### Par의 검사

```scala
val p2 = Prop.check {
    val p = Par.map(Par.unit(1))(_ + 1)
    val p2 = Par.unit(2)
    p(ES).get == p2(ES).get
}
```

이제 두 속성이 동듬함을 상당히 명확하게 표현할 수 있다. 하지만 단지 값을 비교하려 했을 뿐인데, `p(ES).get`이나 `p2(ES).get`같은 세부 사항이 드러났다. `map2`를 이용해 상등 비교를 `Par`로 승급시켜보자

```scala
def equal[A](p: Par[A], p2: Par[A]): Par[Boolean] =
    Par.map2(p, p2)(_ == _)

val p3 = check {
    equal(
        Par.map(Par.unit(1))(_ + 1),
        Par.unit(2)
    )(ES).get
}
```

여기서 다시 `Par`의 실행을 개별 함수 `forAllPar`로 옮기면 어떨까?

```scala
// 75% 의 고정 크기 스레드 풀 실행기, 25%의 크기가 정해지지 않은 스레드 풀 실행기
val S = weighted(
    choose(1, 4).map(Executors.newFixedThreadPool) -> .75,
    unit(Executors.newCachedThreadPool) -> .25)

def forAllPar[A](g: Gen[A])(f: A => Par[Boolean]): Prop =
    forAll(S.map2(g)((_,_))) { case (s, a) => f(a)(s).get }

// 하지만 여기서 S.map2(g)((_,_))는 tuple을 생성하기 위해 두 생성기를 조합한다는 의도에 비해 지저분하다
def **[B](g: Gen[B]): Gen[(A, B)] =
    (this map2 g)((_,_))

def forAllPar[A](g: Gen[A])(f: A => Par[Boolean]): Prop =
    forAll(S ** g) { case (s, a) => f(a)(s).get }
```

이 구문은 여러 생성기를 tuple로 엮을 때 잘 작동한다. `**`를 패턴으로 사용할 수도 있다 \(책 참조\)

이제 `S`는 스레드가 1~4개인 고정 크기 스레드 풀들을 가변적으로 사용하며 크기가 한정되지 않은 스레드 풀도 고려하는 `Gen[ExecutorService]`이다.

```scala
val p2 = chackPar {
    equal (
        Par.map(Par.unit(1))(_ + 1),
        Par.unit(2)
    )
}
```

7장에서 봤던 어떤 계산에 항등 함수를 적용하는 것은 아무런 효과도 없음을 뜻하는 법칙 `map(y)(x => x) == y`를 살펴보자.

```scala
val pint = Gen.choose(0, 10) map (Par.unit(_))
val p4 = forAllPar(pint)(n => equal(Par.map(n)(y => y), n))
```

이 속성을 표현하기는 쉽지 않다. 이 속성은 _임의의 형식의 모든 y_ 에 대해 성립함을 암묵적으로 명시한다. 하지만 여기선 특정한 y값들을 지정해야 한다. 하지만 map에는 속성의 타입\(`Double`, `String` 등\)이 영향을 미치지 않으므로 이 정도로도 확인할 수 있다.

## 8.5 고차 함수의 검사와 향후 개선 방향

지금까지 만든 라이브러리는 _자료_ 를 생성하는 수단은 꽤 갖췄지만, _함수_ 를 생성하는 수단은 없다. 고차 함수를 검사하기엔 무리가 있다.

예를 들어 `List(1, 2, 3).takeWhile(_ < 3)`의 결과는 `List(1, 2)`dㅣ다. 이 `takeWhile` 함수를 어떻게 검사할 수 있을까?

임의의 목록 `s: List[A]`와 임의의 함수 `f: A => Boolean`에 대해, 표현식 `s.takeWhile(f).forall(f)`는 `true`이다. 이렇게 고차 함수를 검사할 때 _특정 인수만 조사_ 하는 접근 방식을 취할 수 있다.

```scala
val isEven = (i: Int) => i % 2 == 0
val takeWhileProp =
    Prop.forAll(Gen.listOf(int))(ns => ns.takeWhile(isEven).forall(isEven))
```

하지만 `takeWhile`에 사용할 함수들을 검사 프레임워크가 자동으로 생성해준다면 더 좋겠지?

## 8.6 생성기의 법칙들

`Gen`에는 `Par`나 `Option`, `List`, `Stream` 등에 정의한 다른 함수들과 상당히 비슷해 보이는 것이 많다. 함수들 간에 서명이 비슷한 부분을 많이 찾을 수 있는데 이런 패턴은 3부에서 자세히 알아보겠다.

