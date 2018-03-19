## Chapter 3. 양도하라
저수준 세부사항의 조작을 *런타임에 양도함*으로써 찾아야 할 수많은 오류를 방지해주는 능력이야말로 함수형 사고의 가치라고 하겠다.

### 3.1 반복 처리에서 고계함수로
이전 장에서 고계함수에 *어떤 연산을 할 것인지를 표현*하기만 하면, 분산 처리를 포함하여 언어가 그 동작을 능률적으로 처리하는 것을 보았다. 그렇다고 개발자가 저수준 추상 단계에서 코드가 어떻게 동작하는지 이해하는 것까지 전부 떠넘겨도 된다는 것은 아니다. 우리는 한 단계 아래를 이해해야 한다.

### 3.2 클로저
> 그 내부에서 참조되는 모든 인수에 대한 묵시적 바인딩을 지닌 함수. 다시 말해 이 함수(또는 메서드)는 자신이 참조하는 것들의 **문맥(context)** 을 포함

모든 함수형 언어는 클로저를 포함한다.

```clojure
class Employee {
  def name, salary
}

def paidMore(amount) {
  return {Employee e -> e.salary > amount}
}

isHighPaid = paidMore(100000)
```

위 예제에서 `paidMore`라는 함수의 리턴 값은 **클로저**라는 코드 블록이다. 여기서 100,000이라는 값은 코드 블록에 **바인딩**되었으며, 추우에 `Employee` 객체의 인스턴스를 받아 직원의 연봉과 바인딩된 값(100,000)을 비교한다. 클로저가 생성될 때에 코드 블록의 스코프에 포함된 인수들을 둘러싼 상자가 같이 만들어져서 이름이 클로저가 되었다. 클로저는 함수형 언어나 프레임워크에서 코드 블록을 다양한 상황에서 실행하게 해주는 메커니즘으로 많이 쓰인다. 클로저를 `map()`과 같은 고계 함수에 전달하는 게 대표적인 예

**Closure**란 단어의 어원은 **문맥을 포괄함(Enclosing context)**이다. 클로저 인스턴스는 *생성될 때 스코프 내에 있던 모든 것을 캡슐화하여 유지*한다. 클로저를 지원하지 않는 경우에도 여러 변형된 (익명 클래스나 제네릭을 사용한) 구현이 가능하지만, 개발자가 *직접 내부 상태를 관리*해야 한다. 클로저는 내부 상태의 관리를 언어나 프레임워크에 맡겨 버린다. 여기서 왜 클로저의 사용이 함수적 사고를 예시하는 지가 분명해진다.

클로저는 *지연 실행(deferred execution)*의 좋은 예이다. 클로저 블록에 코드를 바인딩함으로써 그 블록의 실행을 나중으로 연기할 수 있다.

*명령형 언어*는 **상태**로 프로그래밍 모델을 만든다. *클로저*는 코드와 문맥을 한 구조로 캡슐화해서 **행위**의 모델을 만들 수 있게 해준다.

### 3.3 커링과 부분 적용
커링이나 부분 적용은 함수나 메서드의 인수의 개수를 조작할 수 있게 해준다. 주로 인수 일부에 기본값을 주는 방법을 사용해 인수 고정이라고도 부른다.

### 3.3.1 정의와 차이점
#### 커링
다인수 함수를 일인수 함수들의 체인으로 바꿔주는 방법. *변형 과정*을 지칭하는 것이지, 변형된 함수의 실행을 지칭하는 것은 아니다. 함수의 호출자가 몇 개의 인수를 고정할지를 결정

#### 부분적용
주어진 다인수 함수에서 생략될 인수의 값을 미리 정해서 몇몇 인수에 값을 미리 적용하고 나머지 인수만 받는 함수를 리턴

커링은 체인의 다음 함수를 리턴하는 반면에, 부분 적용은 주어진 값을 인수에 바인딩시켜서 인수가 더 적은 함수를 만든다. 예를 들어, `process(x,y,z)`의 완전히 커링된 버전은 `process(x)(y)(z)`이다. 각각의 함수가 일인수 함수를 리턴하는 일인수 함수이다. `process(x,y,z)`의 인수 하나를 부분 적용하면 인수 두 개짜리의 `process(y,z)`가 된다.

#### 그루비
그루비는 커링은 Closure 클래스의 `curry()` 함수를 사용하여 구현한다.

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

#### 클로저
```clojure
(def subtract-from-hundred (partial - 100))

(subtract-from-hundred 10) ; (- 100 10)과 동일 ; 90
(subtract-from-hundred 10 20) ; (- 100 10 20)과 동일 ; 70
```

클로저에는 `(partial f a1 a2 ...)` 함수가 있다. 함수`f`와 인수 목록을 받아 부분 적용 함수를 리턴해준다. 클로저에서는 연산자와 함수의 구분이 없기 때문에 - 를 부분 적용 함수로 받았다.

클로저는 동적 자료형 언어이고 가변 인수를 지원하기 때문에 커링은 구현되지 않았다.

#### 스칼라
##### 커링
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

##### 부분 적용 함수
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

##### 부분 (제약이 있는) 함수

