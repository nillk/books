## Chapter 2. 전환
새로운 프로그래밍 언어를 배우기는 쉽다. 하지만 새 패러다임을 익히기는 어렵다. 이미 친숙한 문제에 대해 다른 해답을 떠올릴 능력을 배워야 하기 때문이다. 함수형 코드를 작성하기 위해서는 문제에 접근하는 방식의 전환이 필요하다.

### 2.1 일반적인 예제
함수형 프로그래밍은, 복잡한 최적화는 런타임에게 맡기고 개발자가 좀 더 추상화된 수준에서 코드를 작성할 수 있게 함으로써, 알고리즘 측면에서 가비지 컬렉션과 동일한 역할을 수행할 것이다. 개발자들은 가비지 컬렉션에서 얻었던 것같이 복잡성이 낮아지고 성능이 높아지는 혜택을 받게 될 것이다.

#### 2.1.1 명령형 처리
**명령형 프로그래밍**은 *상태를 변형하는 일련의 명령들로 구성된 프로그래밍 방식*이다. 전형적인 for 루프가 명령형 프로그래밍의 훌륭한 예이다. 초기 상태를 설정하고, 되풀이할 때마다 일련의 명령을 실행한다.

```java
public String cleanNames(List<String> listOfNames) {
  StringBuilder result = new StringBuilder();
  for (int i = 0; i < listOfNames.size(); i++) {
    if (listOfNames.get(i).length() > 1) { // 1. 한 글자짜리 이름 필터
      result.append(capitalizeString(listOfNames.get(i))).append(","); // 2. 이름들을 대문자로 변형 3. 하나의 문자열로 변환
    }
  }
  return result.substring(0, result.length() - 1).toString();
}

public String capitalizeString(String s) { // 2. 문자열의 가장 앞 문자를 대문자로 변형
  return s.substring(0, 1).toUpperCase() + s.substring(1, s.length());
}
```

명령형 프로그래밍은 *반복문 내에서 연산하기를 권장*한다. 위 예제에서는 세 가지를 실행했다.
1. 한 글자짜리 이름 **필터**
2. 남아 있는 이름들을 대문자로 **변형**
3. 목록을 하나의 문자열로 **변환**

이 세 가지 작업을 이름 목록에 적용할 **유용한 작업들**이라고 정의하자. 명령형 언어에서는 세 가지 작업에 모두 *저수준의 메커니즘*을 목록 내에서 반복해서 사용해야 한다.

#### 2.1.2 함수형 처리
**함수형 프로그래밍**은 프로그램을 *수학 공식을 모델링하는 표현과 변형으로 기술하며, 가변 상태를 지양*한다.

앞에서 언급한 **필터, 변형, 변환** 등의 논리적 분류도 *저수준의 변형을 구현하는 함수들*이었다. 개발자는 *고계함수에 매개변수로 주어지는 함수를 이용하여 저수준의 작업을 커스터마이즈*할 수 있다. 예를 들어 `map(function)`에서 `map`은 고계함수이고 `function` 은 매개변수로 주어지는 함수에 해당한다. 후자를 사용하여 각 요소에 변형을 적용한다는 뜻이다. 따라서 우리는 이 문제를 의사코드를 사용하여 개념화할 수 있으며, 함수형 언어는 이런 개념화된 해법을 세부사항에 구애받지 않고 모델링할 수 있게 해준다.

##### Scala
```scala
val result = listOfNames
  .filter(_.length() > 1) // 1. 한 글자짜리 이름 필터
  .map(_.capitalize) // 2. 이름들을 대문자로 변형
  .reduce(_ + "," + _) // 3. 하나의 문자열로 변환
```

##### Java 8
```java
public String cleanNames(List<String> names) {
  if (names == null) return "";
  return names
    .stream()
    .filter(name -> name.length() > 1) // 1. 한 글자짜리 이름 필터
    .map(name -> capitalize(name)) // 2. 이름들을 대문자로 변형
    .collect(Collections.joining(",")); // 3. 하나의 문자열로 변환
}

private String capitalize(String e) {
  return e.substring(0, 1).toUpperCase() + e.substring(1, e.length());
}
```

목록 내에 null이 있을 경우에 대비해서 조건 하나를 추가하는 것도 쉽게 할 수 있다. Java 런타임은 null 체크와 길이 필터를 하나의 연산으로 묶어주기 때문에, 개발자는 아이디어를 간단명료하게 표현하면서도 성능이 좋은 코드를 작성할 수 있다.
```java
  return names
    .stream()
    .filter(name -> name != null) // null check
    .filter(name -> name.length() > 1) // 1. 한 글자짜리 이름 필터
    .map(name -> capitalize(name)) // 2. 이름들을 대문자로 변형
    .collect(Collections.joining(",")); // 3. 하나의 문자열로 변환
```

##### Groovy
```groovy
public static String cleanUpNames(listOfNames) {
  listOfNames
    .fildAll { it.length() > 1 } // 1. 한 글자짜리 이름 필터
    .collect { it.capitalize() } // 2. 이름들을 대문자로 변형
    .join ',' // 3. 하나의 문자열로 변환
}
```

##### Clojure
```clojure
(ns trans.core
  (:require [clojure.string :as s]))

(defn process [list-of-emps]
  reduce str (interpose ","
    (map s/capitalize (filter #(< 1 (count %)) list-of-emps))))
```

Clojure와 같은 리스프 계열 언어는 *안에서 밖으로* 실행됨

```clojure
(defn process2 [list-of-emps]
  (->> list-of-emps
    (filter #(< 1 (count %)))
    (map s/capitalize)
    (interpose ",")
    (reduce str)))
```

하지만 클로저에는 스레드-라스트(thread-last)(`->>`) 매크로가 있고, 컬렉션에 적용되는 다양한 변형 연산의 일반적인 순서를 리스프의 전형적인 순서와 반대로 왼쪽에서 오른쪽으로 바꿔 가독성을 높여준다. 리스프의 장점 중 하나가 이런 문법상의 유동성이다.

함수형 사고로의 전환은, 세부적인 구현에 대한 애기가 아니라 이런 *고수준 추상 개념*을 적용할지를 배우는 것이다. 그렇다면 고수준의 추상적 사고로 얻는 이점은 무엇일까?
1. 문제의 공통점을 고려하여 다른 방식으로 분류하기를 권장
2. 런타임이 최적화를 잘할 수 있도록 해준다. 어떤 경우에는 작업 순서를 바꾸면 더 능률적이 된다 (더 적은 아이템을 처리 한다든가...)
3. 개발자가 엔진 세부사항에 깊이 파묻힐 경우 불가능한 해답을 가능하게 함

자바의 경우 위 예제와 같은 문제를 여러 스레드에 나눠서 처리 하게 하려면, 개발자가 저수준의 반복 과정을 제어해야 하기 때문에, 스레드 관련 코드가 문제 해결 코드에 섞여 들어가게 된다. 하지만 스칼라에서는 스트림에 `par`만 붙이면 된다.

```scala
val parallelResult = employees
  .par
  .filter(_.length() > 1)
  .map(_.capitalize)
  .reduce(_ + "," + _)
```

물론 Java 8에서도 가능!
```java
public String cleanNamesP(List<String> names) {
  if (names == null) return "";
  return names
    .parallelStream()
    .filter(n -> n.length() > 1)
    .map(e -> capitalize(e))
    .collect(Collectors.joining(","))
}

```

### 2.2 사례 연구: 자연수의 분류
과잉수, 완전수, 부족수 : 진약수의 합(aliquot sum)

#### 2.2.1 명령형 자연수 분류
Java에서는 진약수의 합을 구하기 위해, *여러가지 상태를 유지*한다. OOP 언어는 캡슐화를 이점으로 사용하기 때문에 객체지향적인 세계에서는 내부 상태의 사용이 보편적이며 권장된다. 상태를 분리 해놓으면 값을 삽입할 수 있기 때문에 단위 테스트 같은 엔지니어링이 쉬워진다.

#### 2.2.2 조금 더 함수적인 자연수 분류기
그렇다면 공유 상태를 최소화하고 싶을 때는 어떻게 할까? *멤버 변수를 없애고 필요한 값들을 매개 변수로 넘긴다.*

조금 더 함수적인 분류기에서는 모든 메서드가 `public static` 스코프를 가지게 되었고, 자급적인 **순수함수** (부수효과가 없는 함수)이다. 또한 내부 상태가 없기 때문에 메서드를 숨길 이유가 없다. 일반적으로 객체지향 시스템에서 재사용할 수 있는 가장 세밀한 요소는 *클래스*이다. 하지만 상태를 제거하고 함수적으로 구현하게 되면 *함수 수준에서 재사용*이 가능하다.

#### 2.2.3 자바 8을 사용한 자연수 분류기
자바 8에서 제공하는 **람다 블록**과 **고계함수**을 통해 개발자들은 높은 수준의 추상화에 접근할 수 있다. 람다와 고계함수, 스트림을 이용해 개발자는 더이상 저수준의 코드 제어를 할 필요가 없다. 자바 8 이전에는 모든 컬렉션이 *운동에너지*와 같다. 컬렉션은 중간 상태 없이 값을 바로 준다. 함수형 언어에서의 스트림은 나중에 사용하기 위해 저장해두는 *위치에너지*와 같다. 스트림은 값을 요구할 때까지는 위치에너지를 운동에너지로 변환하지 않는다. 이게 바로 [4장](Chapter4_Smarter_Not_Harder.md)에서 다룰 *게으른 평가*다.

#### 2.2.4 함수형 자바를 사용한 자연수 분류기
자바 1.5 이후 버전에 무리 없이 함수형 표현을 추가하려는 목적으로 만들어진 오픈소스 프레임워크

함수형 자바 버전과 자바 8 버전의 차이가 단순한 문법 설탕이라고 생각할 수도 있지만, 사실상 그 이상이다. 하지만 문법적 편리함은 중요하다. 한 언어로 아이디어를 표현하는 방식이 곧 **문법**이기 때문이다. 문법적인 장애가 끼어들면 추상적 개념이 사고 과정과 불필요하게 마찰을 빚게 된다.

### 2.3 공통된 빌딩블록
#### 2.3.1 필터
컬렉션의 각 요소를 *사용자가 정한 조건에 통과시켜* 주어진 조건에 맞는 요소들로 이루어진 더 작은 목록을 만든다.

#### 2.3.2 맵
컬렉션의 각 요소에 *같은 함수를 적용*하여 새로운 컬렉션을 만든다.

#### 2.3.3 폴드/리듀스
`foldLeft`나 `reduce`는 캐터모피즘(catamorphism)이라는 목록 조작 개념의 특별한 변형이다. 캐터모피즘은 카테고리 이론의 개념으로 목록을 접어서 다른 형태로 만드는 연산을 총칭한다. 이 두가지는 언어들 사이에서도 이름이 다양하고 약간씩 의미도 다르다.
1. 둘 다 *누산기(accumulator)* 를 사용해 값을 모은다.
2. `reduce`는 주로 *초기 값*을 주어야 할 때 사용하고, `fold`는 *누산기에 아무것도 없는 채*로 시작한다.
    > 아래 예제나 구글링 결과는 fold 가 초기값을 가진 채로 시작한다고 한다. 실수인듯...
3. 컬렉션에 대한 연산 순서는 메서드 이름(`foldLeft` 또는 `foldRight`)으로 지정한다.
4. 둘 다 컬렉션을 변경하지 않는다.

예를 들어 `foldLeft()`는 다음과 같은 의미이다.
* 이항 함수나 연산으로 목록의 첫째 요소와 누산기의 초기 값을 결합. (초기 값이 없는 경우도 있다.)
* 앞의 단계를 목록이 끝날 때까지 계속하면 *누산기가 연산의 결과를 갖게 된다.*

`fold`나 `reduce`의 경우 컬렉션의 각 요소에 *매번 새로운 함수(같은 이항 함수이지만 첫 매개변수가 달라짐)를 적용*하게 된다. 모든 요소에 같은 일항 함수를 적용하는 `map`과는 구별된다.

### 2.4 골치 아프게 비슷비슷한 이름들
함수형 언어들은 공통된 부류의 몇 가지 함수들을 가지고 있다.

#### 2.4.1 필터
##### 스칼라
필터의 형태가 여러가지

1. filter: 입력 조건에 따라 컬렉션을 필터한 결과를 리턴
```scala
val numbers = List.range(1, 11)
numbers filter (_ % 3 == 0) // List(3, 6, 9)
```
2. partition: 컬렉션을 여러 조각으로 분리한 결과를 리턴. 분리 조건을 정하는 함수를 전달한다.
```scala
numbers partition (_ % 3 == 0) // (List(3, 6, 9), List(1, 2, 4, 5, 7, 8, 10))
```
3. find: 컬렉션에서 조건을 만족시키는 첫 번째 요소만 리턴
```scala
numbers find (_ % 3 == 0) // Some(3)
numbers find (_ < 0) // None
```
하지만 값 자체는 아니고, `Option` 클래스로 래핑된 값이다. `Option`은 `Some` 또는 `None` 두 가지 값을 가질 수 있다. (다른 함수형 언어들과 마찬가지로 `null`을 리턴하는 것을 피하기 위해 관례적으로 사용) 값이 없는 경우 `None`을 리턴한다.  
4. takeWhile: 컬렉션의 앞에서부터 술어 함수를 만족시키는 값들의 최대 집합을 리턴
```scala
List(1, 2, 3, -4, 5, 6, 7, 8, 9, 10) takeWhile (_ > 0) // List(1, 2, 3)
```
5. dropWhile: 술어 함수를 만족시키는 최다수의 요소를 건너뛴다.
```scala
val words = List("the", "quick", "brown", "fox", "jumped", "over", "the", "lazy", "dog")
words dropWhile (_ startsWith "t") // List(quick, brown, fox, jumped, over, the, lazy, dog)
```

##### 그루비
그루비는 함수형 언어라고 할 수는 없지만 스크립팅 언어에서 파생된 이름을 가진 함수형 패러다임을 다수 포함하고 있다.
1. findAll: 함수형 언어에서 전통적으로 `filter()`라고 불리는 함수
```groovy
(1..10).findAll {it % 3 == 0} // [3, 6, 9]
```
2. split: `partition()`과 유사한 함수
```groovy
(1..10).split {it % 3} // [ [1, 2, 4, 5, 7, 8, 10], [3, 6, 9] ]
```
3. find: 컬렉션에서 조건을 만족시키는 첫 번째 요소만 리턴. 하지만 스칼라와는 달리, 그루비는 자바의 관례를 따라 `find()`의 결과가 없을 때는 `null`을 리턴
```groovy
(1..10).find {it % 3 == 0} // 3
(1..10).find {it < 0} // null
```
4. takeWhile
```groovy
[1, 2, 3, -4, 5, 6, 7, 8, 9, 10].takeWhile {it > 0} // [1, 2, 3]
```
5. dropWhie
```groovy
def moreWords = ["the", "two", "ton"] + words
moreWords.dropWhile {it.startWith("t")} // [quick, brown, fox, jumped, over, the, lazy, dog]
```

##### 클로저
클로저에는 컬렉션을 조작하는 루틴들이 놀라운 만큼 많이 있다. 클로저는 전통적인 함수형 프로그래밍 이름을 사용한다.
1. filter
```clojure
(def numbers (range 1 11))
(filter (fn [x] (=0 (rem x 3))) numbers) ; (3, 6, 9)

(filter #(zero? (rem % 3)) numbers) ; (3, 6, 9)
```

클로저에서 `(filter )`의 리턴 자료형은 ()로 표기되는 `Seq`이다.

#### 2.4.2 맵
##### 스칼라
1. map: 코드 블록을 받아서 각 요소에 적용 후 변형된 컬렉션을 리턴
```scala
List(1, 2, 3, 4, 5) map (_ + 1) // List(2, 3, 4, 5, 6)
```
2. flatMap: 중첩 목록을 펼치는 연산을 지원. 흔히 flattening이라고 한다. `flatMap()` 함수는 중첩되지 않는 것처럼 보이는 자료구조에도 적용된다. 일례로 문자열의 목록은 중첩된 문자들의 배열로 볼 수 있다.
```scala
List(List(1, 2, 3), List(4, 5, 6), List(7, 8, 9)) flatMap (_.toList) // List(1, 2, 3, 4, 5, 6, 7, 8, 9)
words flatMap (_.toList) // List(t, h, e, q, u, i, c, k, b, r, o, w, n, f, o, x, ...)
```

##### 그루비
1. collect: `map`의 변형
```groovy
(1..5).collect {it += 1}
```
2. flatten: `flatMap`과 유사
```groovy
[ [1, 2, 3], [4, 5, 6], [7, 8, 9] ].flatten() // [1, 2, 3, 4, 5, 6, 7, 8, 9]
(words.collect {it.toList()}).flatten() // [t, h, e, q, u, i, c, k, b, r, o, w, n, f, o, x, ...]
```

##### 클로저
1. map
```clojure
(map inc numbers) ; 첫째 매개변수는 단일 매개변수를 하나만 받는 어떤 함수든 가능
; (2, 3, 4, 5, 6, 7, 8, 9, 10)
```
2. flatten
```clojure
(flatten [[1 2 3] [4 5 6] [7 8 9]]) ; (1, 2, 3, 4, 5, 6, 7, 8, 9)
```

#### 2.4.3 폴드/리듀스
##### 스칼라
동적 타이핑 언어인 그부리나 클로저에서는 다루지 않는 다양한 자료형 시나리오들을 해결해야 하기 때문에 폴드 연산 종류가 가장 다양하다.
1. reduce: 두 개의 매개변수를 받아서 하나의 결과를 리턴. `reductLeft()` 함수는 첫째 요소가 연산의 좌항이라고 간주한다. 덧셈은 피연산자의 위치에 상관 없지만, 순서가 중요한 연산을 할 때 연산자가 적용되는 순서를 바꾸려면 `reduceRight()`를 사용하라. 주의할 것은, 피연산자의 순서를 바꾸는 게 아니라, *연산의 방향*을 뒤바꾼다는 것이다.
```scala
List.range(1, 10) reduceLeft(_ + _) // 45
List.range(1, 10) reduceRight(_ - _) // 5 // 1-(2-(3-(4-(5-(6-(7-(8-9)))))))
// 8 - 9 = -1
// 7 - (-1) = 8
// 6 - 8 = - 2
// 5 - (-2) = 7
// 4 - 7 = -3
// 3 - (-3) = 6
// 2 - 6 = -4
// 1 - (-4) = 5
```
2. fold: `reduce`와 유사하지만, 초기 시드 값을 포함한다. 스칼라는 연산자 오버로딩을 지원한다. 자주 사용되는 두 폴드 연산 `foldLeft`와 `foldRight`는 상응하는 연산자 `/:`과 `:\`가 있다.
```scala
List.range(1, 10).foldLeft(0)(_ + _) // 45

(0 /: List.range(1, 10)) (_ + _) // 45
(List.range(1, 10) :\ 0) (_ - _) // 5
```

##### 그루비
그루비는 멀티 패러다임 언어이기 때문에 스칼라나 클로저에 비해 함수형 라이브러리가 적다.
1. inject: `reduce` 계열의 함수
```groovy
(1..9).inject {a, b -> a + b} // 45
(1..9).inject(0, {a, b -> a + b}) // 45
```

##### 클로저
`(reduce )`를 지원하며, 선택적으로 초기 값을 받아서 스칼라의 `reduce()`와 `foldLeft()` 두 경우를 해결한다.
1. reduce
```clojure
(reduce + (range 1 10))
; 45
```
