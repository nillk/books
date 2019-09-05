# Chapter 3. 양도하라

저수준 세부사항의 조작을 _런타임에 양도함_ 으로써 찾아야 할 수많은 오류를 방지해주는 능력이야말로 함수형 사고의 가치라고 하겠다.

## 3.1 반복 처리에서 고계함수로

이전 장에서 고계함수에 _어떤 연산을 할 것인지를 표현_ 하기만 하면, 분산 처리를 포함하여 언어가 그 동작을 능률적으로 처리하는 것을 보았다. 그렇다고 개발자가 저수준 추상 단계에서 코드가 어떻게 동작하는지 이해하는 것까지 전부 떠넘겨도 된다는 것은 아니다. 우리는 한 단계 아래를 이해해야 한다.

## 3.2 클로저

> 그 내부에서 참조되는 모든 인수에 대한 묵시적 바인딩을 지닌 함수. 다시 말해 이 함수\(또는 메서드\)는 자신이 참조하는 것들의 **문맥\(context\)** 을 포함

모든 함수형 언어는 클로저를 포함한다.

```groovy
class Employee {
  def name, salary
}

def paidMore(amount) {
  return {Employee e -> e.salary > amount}
}

isHighPaid = paidMore(100000)
```

위 예제에서 `paidMore`라는 함수의 리턴 값은 **클로저**라는 코드 블록이다. 여기서 100,000이라는 값은 코드 블록에 **바인딩**되었으며, 추후에 `Employee` 객체의 인스턴스를 받아 직원의 연봉과 바인딩된 값\(100,000\)을 비교한다. 클로저가 생성될 때에 코드 블록의 스코프에 포함된 인수들을 둘러싼 상자가 같이 만들어져서 이름이 클로저가 되었다. 클로저는 함수형 언어나 프레임워크에서 코드 블록을 다양한 상황에서 실행하게 해주는 메커니즘으로 많이 쓰인다. 클로저를 `map()`과 같은 고계 함수에 전달하는 게 대표적인 예

**Closure**란 단어의 어원은 _문맥을 포괄함\(Enclosing context\)_ 이다. 클로저 인스턴스는 _생성될 때 스코프 내에 있던 모든 것을 캡슐화하여 유지_ 한다. 언어에서 클로저를 지원하지 않는 경우에도 여러 변형된 \(익명 클래스나 제네릭을 사용한\) 구현이 가능하지만, 개발자가 _직접 내부 상태를 관리_ 해야 한다. 클로저는 내부 상태의 관리를 언어나 프레임워크에 맡겨 버린다. 여기서 왜 클로저의 사용이 함수적 사고를 예시하는 지가 분명해진다.

클로저는 _지연 실행\(deferred execution\)_ 의 좋은 예이다. 클로저 블록에 코드를 바인딩함으로써 그 블록의 실행을 나중으로 연기할 수 있다.

_명령형 언어_ 는 **상태**로 프로그래밍 모델을 만든다. _클로저_ 는 코드와 문맥을 한 구조로 캡슐화해서 **행위**의 모델을 만들 수 있게 해준다.

## 3.3 커링과 부분 적용

커링이나 부분 적용은 함수나 메서드의 인수의 개수를 조작할 수 있게 해준다. 주로 인수 일부에 기본값을 주는 방법을 사용해 인수 고정이라고도 부른다.

## 3.3.1 정의와 차이점

### 커링

다인수 함수를 일인수 함수들의 체인으로 바꿔주는 방법. _변형 과정_ 을 지칭하는 것이지, 변형된 함수의 실행을 지칭하는 것은 아니다. 함수의 호출자가 몇 개의 인수를 고정할지를 결정

### 부분적용

주어진 다인수 함수에서 생략될 인수의 값을 미리 정해서 몇몇 인수에 값을 미리 적용하고 나머지 인수만 받는 함수를 리턴

커링은 체인의 다음 함수를 리턴하는 반면에, 부분 적용은 주어진 값을 인수에 바인딩시켜서 인수가 더 적은 함수를 만든다. 예를 들어, `process(x,y,z)`의 완전히 커링된 버전은 `process(x)(y)(z)`이다. 각각의 함수가 일인수 함수를 리턴하는 일인수 함수이다. `process(x,y,z)`의 인수 하나를 부분 적용하면 인수 두 개짜리의 `process(y,z)`가 된다.

### 그루비

그루비의 커링은 `Closure` 클래스의 `curry()` 함수를 사용하여 구현한다.

```groovy
def product = { x, y -> x * y }

def quadrate = product.curry(4)
def octate = product.curry(8)

println "4x4: ${quadrate.call(4)}"
println "8x8: ${octate(8)}"
```

위에서 `curry()`가 매개변수 하나를 고정하고 일인수 함수를 리턴한다. 명시적으로 `call()`을 사용할 수도 있지만, 그냥 코드 블록 이후에 매개 변수를 적을 수도 있다.

```groovy
def volume = {h, w, l -> h * w * l}
def area = volume.curry(1)
def lengthPA = volume.curry(1, 1) // 부분적용
def lengthC = volume.curry(1).curry(1) // 커링
```

위의 `lengthPA`는 부분적용이고, `lengthC`는 커링을 두 번 적용하여 같은 결과를 얻는다. 미묘한 차이가 있지만 결과는 마찬가지다. 그루비는 밀접한 이 두 개념을 하나로 사용한다.

### 클로저

```text
(def subtract-from-hundred (partial - 100))

(subtract-from-hundred 10) ; (- 100 10)과 동일 ; 90
(subtract-from-hundred 10 20) ; (- 100 10 20)과 동일 ; 70
```

클로저에는 `(partial f a1 a2 ...)` 함수가 있다. 함수`f`와 인수 목록을 받아 부분 적용 함수를 리턴해준다. 클로저에서는 연산자와 함수의 구분이 없기 때문에 - 를 부분 적용 함수로 받았다.

클로저는 동적 자료형 언어이고 가변 인수를 지원하기 때문에 커링은 구현되지 않았다.

### 스칼라

#### 커링

스칼라에서는 여러 개의 인수 목록을 여러 개의 괄호로 정의할 수 있다.

```scala
object CurryTest extends App {
  def filter(xsL List[Int], p: Int => Boolean): List[Int] =
    if (xs.isEmpty) xs
    else if (p(xs.head)) xs.head :: filter(xs.tail, p)
    else filter(xs.tail, p)

  def modN(n: Int)(x: Int) = ((x % n) == 0) // 커링 함수

  val nums = List(1, 2, 3, 4, 5, 6, 7, 8)
  println(filter(nums, modN(2)))
  println(filter(nums, modN(3)))
}
```

#### 부분 적용 함수

```scala
def price(product : String) : Double =
  product match {
    case "apples" => 140
    case "oranges" => 223
  }

def withTax(cost: Double, state: String) : Double =
  state match {
    case "NY" => cost * 2
    case "FL" => cost * 3
  }

val locallyTaxed = withTax(_: Double, "NY") // 부분 적용
val costOfApples = locallyTaxed(price("apples"))

assert(Math.round(costOfApples) == 280)
```

#### 부분 \(제약이 있는\) 함수

스칼라의 `PartialFunction` 트레이트는 [6장](https://github.com/nillk/TIL/tree/3c6424e4329f516067f44b3217c791ffbf08e908/Functional-Thinking/Chapter6_Advance.md)에서 자세히 다룰 패턴 매칭에 사용하려고 설계된 것이다. 이는 부분 적용 함수를 생성하지 않는다. _특정한 값이나 자료형에만 적용되는 함수_ 를 만드는데 사용할 수 있다.

`case` 블록은 부분 적용 함수가 아니라 부분함수를 정의한다. _부분 함수\(Partial function\)_ 는 허용되는 값을 제한하는 방법을 제공한다. 아래 예제 코드에서와 같이, `map`은 서로 다른 타입의 값을 가진 컬렉션에서는 사용할 수 없지만, `collect`는 정의되어 있는 값\(정수\)만을 collect 할 수 있다.

```scala
List(1, 3, 5, "seven") map { case i: Int => i + 1 } // scala.MatchError:seven (of class java.lang.String)
List(1, 3, 5, "seven") collect { case i: Int => i + 1 } // List(2, 4, 6)
```

부분 함수를 정의하려면 아래와 같이 `PartialFunction` 트레이트를 사용할 수도 있다. `PartialFunction` 이 아니라 `case` 블록과 방호블록\(Guard condition\)을 사용할 수도 있는데, 위의 코드 같은 경우 `answerUnits(0)` 를 호출할 경우 `ArithmeticException`이 나지만, `case` 블록을 사용한 경우 `MatchError`가 발생한다.

```scala
val answerUnits = new PartialFunction[Int, Int] {
  def apply(d: Int) = 42 / d
  def isDefinedAt(d: Int) = d != 0
}

// use case
val pAnswerUnits: PartialFunction[Int, Int] =
  { case d: Int if d != 0 => 42 / 0 }
```

`map()` 과 `collect()`가 다르게 작동하는 것은 이 부분함수의 작동 박식 떄문이다. `collect()`는 `isDefinedAt()`을 불러 맞지 않는 요소는 무시한다.

#### 보편적인 용례

**함수 팩토리**

커링\(또는 부분 적용\)은 전통적인 객체지향 언어에서 _팩토리 함수\(factory function\)_ 를 구현할 상황에서 사용하면 좋다.

```groovy
def adder = { x, y -> x + y }
def incrementer = adder.curry(1)

println "increment 7: ${incrementer(7)}" // 8
```

**템플릿 메서드 패턴**

이 패턴은 구현의 유연성을 보장하기 위해서 내부의 추상 메서드를 사용하는 겉껍질을 정의하는 데 있다. 부분 적용을 사용해 이미 알려진 기능을 제공하고 나머지 인수들은 추후에 구현하도록 남겨두는 것은 이 패턴과 흡사하다. [6장 참조](https://github.com/nillk/TIL/tree/3c6424e4329f516067f44b3217c791ffbf08e908/Functional-Thinking/Chapter6_Advance.md)

**묵시적인 값**

비슷한 인수 값들로 여러 함수를 연속적으로 불러올 때는 커링을 사용하여 _묵시적 인수 값_ 을 제공할 수 있다. 아래는 자료 저장 프레임워크를 사용할 때, 첫번째 인수인 데이터 소스를 묵시적으로 제공한다.

```text
(defn db-connect [data-source query params]
  ...)

(defn dbc (partial db-connect "db/some-data-source"))

(dbc "select * from %1" "cust")
```

### 3.3.2 재귀

재귀란 자신을 재참조하여 같은 프로세스를 반복하는 것을 말한다. 구체적으로 말해 같은 메서드를 반복해서 호출하여 컬렉션을 각 단계마다 줄여가면서 반복 처리하는 것을 말한다. \(종료 조건 명심!\) 재귀로 짠 코드가 이해가 쉬운 이유는 _문제의 핵심이 같은 작업을 반복_ 하며 목록을 줄여가는 것이기 때문이다.

#### 목록의 재조명

배경이 C나 C계열 언어\(자바 포함\)인 개발자들은 목록을 색인된 컬렉션으로 생각한다. 각각의 항목을 0번부터 매겨 정렬된 위치의 컬렉션으로 생각하는 것이다.

```groovy
def numbers = [6, 28, 4, 9, 12, 4, 8, 8, 11, 45, 99, 2]

def iterateList(listOfNums) {
  listOfNums.each { n ->
    println "${n}"
  }
}

iterateList(numbers)
```

많은 함수형 언어는 목록을 다른 각도에서 바라본다. 색인된 위치의 컬렉션으로 여기는 대신에, _첫 요소\(머리\)와 나머지\(꼬리\)의 조합_ 으로 생각해보자.

```groovy
def recurseList(listOfNums) {
  if (listOfNums.size == 0) return;

  println "${listOfNums.head()}" // 첫번째 요소
  recurseList(listOfNums.tail()) // 나머지 요소들
}

recurseList(numbers)
```

재귀는 종종 플랫폼마다 기술적 한계가 있기 때문에 만병통치약이 될 수는 없지만 길지 않은 목록을 처리하는 데에는 안전하다. 필터하는 예제를 보자. 아래 두 예제는 **누가 상태를 관리하는가?** 란 질문을 조명한다. 명령형 버전에서는 **개발자**가 관리한다. `new_list`를 생성하고, 필터된 항목을 추가하고, 마지막에 `new_list`를 리턴한다. 재귀 버전에서는 **언어**가 메서드 호출 시마다 리턴 값을 스택에 쌓아가면서 관리한다. 개발자는 `new_list`에 대한 책임을 양도하고 언어가 그것을 관리한다.

**명령형 필터**

```groovy
def filter(list, predicate) {
  def new_list = []
  list.each {
    if (predicate(it)) {
      new_list << it
    }
  }
  return new_list
}

modBy2 = { n -> n % 2 == 0 }

l = filter(1..20, modBy2)
```

**재귀적 필터**

```groovy
def filterR(list, predicate) {
  if (list.size() == 0) return list
  if (predicate(list.head()))
    [] + list.head() + filterR(list.tail(), predicate)
  else
    filterR(list.tail(), predicate)
}

modBy2 = { n -> n % 2 == 0 }

l = filter(1..20, modBy2)
```

_**꼬리 호출 최적화**_ 는 런타임에 도움을 줄 수 있다.

### 3.4 스트림과 작업 재정렬

명령형 사고로는 `filter` 작업을 먼저 한 후에 `map` 작업을 해야 한다. 그래야 맵 작업의 양이 줄어든다. 하지만 함수형 언어에는 `Stream`이란 추상 개념이 정의되어 있다. `Stream` 은 바탕 값\(backing value\)이 없다. `Stream` 은 그저 원천에서 목적지까지 값들이 흐르게 하는데, 그 사이에 있는 함수들은 **게으른** 함수 들이다. 목적지에서 값을 요구하지 않으면 결과를 내려고 시도하지도 않는다.

런타임은 이런 **게으른 작업**들을 효율적으로 재정렬할 수 있다. 그러기 위해서는, `filter`같은 함수에 주어진 람다 블록에 _부수 효과_ 가 없어야 한다.

