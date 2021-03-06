# Chapter 5. 진화하라

함수형 언어의 코드 재사용은 객체지향과는 다르다.

* 객체지향 언어: 수많은 자료구조와 거기에 딸린 수많은 연산을 포함. 클래스에 종속된 메서드를 만드는 것을 권장해 _반복되는 패턴을 재사용_
* 함수형 언어: 적은 수의 자료구조와 많은 연산을 포함. 자료구조에 대해 _공통된 변형 연산을 적용_

## 5.1 적은 수의 자료구조, 많은 연산자

객체지향적인 명령형 프로그래밍 언어에서 재사용의 단위는 클래스와 그것들이 주고 받는 메시지이다. OOP 세상에서는 특정한 메서드가 장착된 특정한 자료구조를 개발자가 만들기를 권장한다. 함수형에서는 몇몇 주요 자료구조\(list, set, map\)와 거기에 따른 최적화된 연산들을 선호한다. 함수 수준에서 캡슐화하면 커스텀 클래스 구조를 만드는 것보다 좀 더 _세밀하고 기본적인 수준에서 재사용_ 이 가능해진다.

## 5.2 문제를 향하여 언어를 구부리기

유연한 언어를 접하면 문제를 언어에 맞게 바꾸는 대신 언어를 문제에 어울리게 구부릴 수 있다. 특별히 함수형 언어만의 기능은 아니지만, _언어를 우아하게 문제 도메인으로 바꾸는 기능_ 은 함수형, 선언형 방식의 현대 언어에서 흔히 볼 수 있다.

스칼라는 가단성을 위해 설계되었기 때문에 연산자 오버로딩이나 묵시적 자료형 같은 확장이 가능하다.

> 가단성: 금속 가공에서 나온 개념으로, 고체가 외부에서 작용하는 힘에 의해 외형이 변하는 성질. 소프트웨어에서는 언어나 프레임워크가 사용하는 코드에 의해 용도나 반응이 변하는 성질을 의미한다.

## 5.3 디스패치 다시 생각하기

여기서 디스패치란 넓은 의미로 언어가 작동 방식을 동적으로 선택하는 것을 말한다.

### 5.3.1 그루비로 디스패치 개선하기

자바의 `switch`와 유사하지만, 여러가지 동적 자료형을 받을 수 있다. 또한 범위, 열린 범위, 정규식, 디폴트 조건을 모두 사용할 수 있어서 자바처럼 `if`를 연속적으로 사용하거나 팩토리 패턴을 사용할 필요 없이 간편하고 간결한 코드를 작성할 수 있다.

### 5.3.2 클로저 언어 구부리기

자바나 자바 계열 언어들ㄹ에는 _키워드_ 가 있다. 키워드들은 문법의 기반을 이루며, 개발자들은 언어 내에서 키워드를 만들 수 없다. 자바에서는 함수나 클래스를 만들 수는 있지만 기초적인 빌딩 블록을 만드는 것은 불가능하다. 따라서 개발자는 문제를 프로그래밍 언어로 번역해야 한다. 클로저 같은 리스프 계열의 언어에서는 개발자가 언어를 문제에 맞게 변형할 수 있다.

```text
(ns lettergrades)

(defn in [score low high]
  (and (number? score) (<= low score high)))

(defn letter-grade [score]
  (cond
  (in score 90 100) "A"
  (in score 80 90) "B"
  (in score 70 80) "C"
  (in score 60 70) "D"
  (in score 0 60) "F"
  (re-find #"[ABCDFabcdf]" score) (.toUpperCase score)))
```

### 5.3.3 클로저의 멀티메서드와 맞춤식 다형성

자바의 경우 클래스를 사용한 다형성을 사용한다. 클로저도 다형성을 지원하지만 클래스를 평가해서 디스패치를 결정하는 것에 국한되어 있지는 않다. 클로저는 개발자가 원하는 대로 디스패치가 결정되는 다형성 멀티메서드를 지원한다. 다형성을 상속과 분리하면 강력하고 상황에 맞는 디스패치 방식이 가능해진다.

## 5.4 연산자 오버로딩

함수형 언어의 공통적인 기능은 연산자 오버로딩이다.

### 5.4.1 그루비

그루비는 연산자들을 메서드 이름에 자동으로 매핑하는 연산자 오버로딩을 허용한다. 일례로 정수 클래스에서 `+` 연산자를 오버로딩하려면 `plus()` 메서드를 오버라이딩하면 된다. 이런 연산자 오버로딩은 특정한 분야에서 애플리케이션을 만드는 개발자들에게, 각자의 문제 도메인에 적합한 연산자를 만들어서 쓸 수 있게 함으로써 큰 도움이 된다.

### 5.4.2 스칼라

스칼라는 연산자와 메서드릐 차이점을 없애는 방법으로 연산자 오버로딩을 허용한다. 즉 연산자는 _특별한 이름을 가진 메서드_ 에 불과하다. 따라서 곱셈 연산자를 스칼라에서 오버라이드 하려면 `*` 메서드를 오버라이드하면 된다.

## 5.5 함수형 자료구조

대부분의 함수형 언어들은 예외 패러다임을 지원하지 않는다. 예외는 함수형 언어가 준수하는 전제 몇 가지를 깨뜨린다.

1. 함수형 언어는 부수효과가 없는 _순수함수_ 를 선호한다. 예외를 발생시키는 것은 예외적인 프로그램 흐름을 야기하는 부수효과다. 함수형 언어들은 주로 _값_ 을 처리하기 때문에 예외적인 프로그램 흐름보다는 _오류를 나타내는 리턴값_ 에 반응하는 것을 선호한다.
2. 또 하나의 특성은 _참조 투명성_ 이다. 함수형에서는 단순한 값 하나를 사용하든, 값을 리턴하는 함수를 사용하든 다를 바가 없어야 한다. 만약 함수에서 예외가 발생할 수 있다면, 호출하는 입장에서는 값을 함수로 대체할 수 없다.

### 5.5.1 함수형 오류 처리

자바에서 예외를 사용하지 않고 오류를 처리하기 위해 해결해야 할 근본적인 문제는 메서드가 하나의 값만 리턴할 수 있다는 제약이다. 물론 메서드를 여러 개의 값을 지닌 하나의 `Object`나 그 서브클래스의 참조를 리턴할 수 있다. 따라서 `Map`을 사용하여 다수의 리턴 값을 지원하게 할 수 있다. 하지만 이 접근 방법에는 문제점이 있다.

1. `Map`에 들어가는 값은 타입 세이프하지 않기 때문에 컴파일러가 오류를 잡아낼 수 없다.
2. 메서드 호출자는 리턴 값을 가능한 결과들과 비교해보기 전에는 성패를 알 수 없다.
3. 여러 가지 결과가 모두 리턴 `Map`에 존재할 수가 있으므로, 그 경우에는 결과가 모호해진다.

따라서 타입 세이프하게 둘 또는 더 많은 값을 리턴할 수 있게 해주는 메커니즘이 필요해진다.

### 5.5.2 Either 클래스

`Either`는 왼쪽 또는 오른쪽 값 중 하나만 가질 수 있게 설계되었다. 이런 자료구조를 _분리합집합 disjoint union_ 이라고 한다. 자바에도 내장되지는 않았지만, 제네릭을 사용해 간단한 `Either` 클래스를 만들 수 있다. `Either`를 사용하면 타입 세이프티를 유지하면서 예외 또는 제대로 된 결과 값을 리턴하는 코드를 만들 수 있다. `Either`는 두 부분을 가진 값을 간편하게 리턴할 수 있는 개념이다. 이는 트리 모양 자료구조와 같은 일반적인 자료구조를 만드는 데 유용하다.

#### 로마숫자 파싱

`Either`를 사용한 리턴은 예외를 얼마든지 세부화할 수 있으면서도 타입 세이프티를 지킬 수 있다. 또한 제네릭을 통한 메서드 선언으로 오류를 분명하게 알 수 있다.

#### 게으른 파싱과 함수형 자바

함수형 자바의 `P1` 클래스를 조합해서 사용하면 게으른 오류 평가를 구현할 수 있다. `P1`은 함수형 자바에서 코드 블록을 실행하지 않고 여기저기 전달해주어 원하는 컨텍스트에서 실행하게 해주는 일종의 고계함수다. 아래 예제에서 보이는 바와 같이 `_1()`을 호출하기 전까지는 결과가 생성되지 않는다.

```java
public static P1<Either<Exception, Integer>> parseNumberLazy(final String s) {
  if (!s.matches("[IVXLXCDM]+"))
    return new P1<Either<Exception, Integer>>() {
      public Either<Exception, Integer> _1() {
        return Either.left(new Exception("Invalid Roman numeral"));
      }
    };
  else
    return new P1<Either<Exception, Integer>>() {
      public Either<Exception, Integer> _1() {
        return Either.right(new RomanNumeral(s).toInt());
      }
    };
}
```

#### 디폴트 값을 제공하기

`Either`를 예외 처리에 사용하여 얻는 이점을 게으름만이 아니라 디폴트 값을 제공한다는 점도 있다.

```java
private static final int MIN = 0;
private static final int MAX = 0;

public static Either<Exception, Integer> parseNumberDefaults(final String s) {
  if (!s.matches("[IVXLXCDM]+"))
    return Either.left(new Exception("Invalid Roman numeral"));
  else {
    int number = new RomanNumeral(s).toInt();
    return Either.right(new RomanNumeral(number > MAX ? MAX : number).toInt());
  }
}
```

#### 에외 조건을 래핑하기

`Either`를 사용하면 예외를 래핑하여 구조적인 예외 처리를 함수형으로 변환할 수도 있다.

```java
public static Either<Exception, Integer> divide(int x, int y) {
  try {
    return Either.right(x / y);
  } catch (Exception e) {
    return Either.left(e);
  }
}
```

### 5.5.3 옵션 클래스

`Option` 클래스는 `Either`와 유사한 클래스로 적당한 값이 존재하지 않을 경우 `none`, 성공적인 리턴을 의미하는 `some`을 가지며 이를 사용하여 예외 조건을 더 쉽게 표현할 수 있다. `Either`와 유사하지만 적당한 리턴 값이 없을 수 있는 메서드를 위해 `none()`과 `some()`을 가지고 있다. `Option` 클래스는 `Either`의 간단한 부분집합이라고 볼 수 있다. `Either`는 어떤 값이든 저장할 수 있는 반면, `Option`은 주로 성공과 실패의 값을 저장하는 데 쓰인다.

```java
public static Option<Double> divide(double x, double y) {
  if (y == 0)
    return Option.none();
  return Option.some(x / y);
}
```

### 5.5.4 Either 트리와 패턴 매칭

#### 스칼라 패턴 매칭

스칼라는 디스패치에 패턴 매칭을 사용할 수 있다. 패턴 매칭은 _케이스 클래스 case class_ 와 함께 사용된다.

스칼라의 케이스 클래스

* 클래스 이름을 사용한 팩토리 클래스를 만들 수 있다.
* 클래스의 모든 변수들이 자동적으로 `val`이 된다.
* 컴파일러가 적당한 디폴트 `equals()`, `hashCode()`, `toString()` 메서드를 자동으로 생성해준다.
* 불변 객체를 지원하기 위하여 컴파일러가 새로운 복제 객체를 리턴하는 `copy()` 메서드를 자동으로 생성해준다.

#### Either 트리

* empty : 셀에 아무 값도 없음
* leaf : 셀에 특정 자료형의 값이 들어 있음
* node : 다른 leaf나 node를 가리킴

`Either`의 추상 개념은 원하는 개수만큼 슬롯을 확장할 수 있다. 일례로 `Either<Empty, Either<Leaf, Node>>` 같은 선언을 생각해보자. 이 자료구조의 각 _셀_ 은 가능한 세 자료형 중에서 한 번에 한 개만 가질 수 있는, 전통적인 컴퓨터과학에서의 `union`에 해당한다.

#### 패턴 매칭으로 트리 순회하기

패턴 매칭은 개발자가 경우들을 분리해 생각하게 한다. 아래 예제는 트리의 깊이를 알아내는 코드이며, 이를 변형해 값을 검색하거나, 값이 트리 안에서 몇 번 나타나는지 등의 코드를 작성할 수 있다. 이런 재귀적인 트리 탐색이 가능한 것은 트리가 `<Either, <Left, Node>>`란 특정한 자료구조로 만들어졌기 때문에 각각의 _슬롯_ 을 특정한 경우로 생각할 수 있어서이다. 트리의 내부 구조를 규격화한 덕분에, 트리를 따라가면서 각 요소의 자료형에 따른 경우에 대해서만 생각하며 분석할 수 있다.

```java
static public int depth(Tree t) {
  for (Empty e : t.toEither().left())
    return 0;
  for (Either<Leaf, Node> ln : t.toEither().right()) {
    for (Leaf leaf : ln.left())
      return 1;
    for (Node node : ln.right())
      return 1 + max(depth(node.left), depth(node.right));
  }
  throw new Exception("Inexhaustible pattern match on tree");
}
```

