## Chapter 2. 전환
새로운 프로그래밍 언어를 배우기는 쉽다. 하지만 새 패러다임을 익히기는 어렵다. 이미 친숙한 문제에 대해 다른 해답을 떠올릴 능력을 배워야 하기 때문이다. 함수형 코드를 작성하기 위해서는 문제에 접근하는 방식의 전환이 필요하다.

### 2.1 일반적인 예제
함수형 프로그래밍은, 복잡한 최적화는 런타임에게 맡기고 개발자가 좀 더 추상화된 수준에서 코드를 작성할 수 있게 함으로써, 알고리즘 측면에서 가비지 컬렉션과 동일한 역할을 수행할 것이다. 개발자들은 가비지 컬렉션에서 얻었던 것같이 복잡성이 낮아지고 성능이 높아지는 혜택을 받게 될 것이다.

#### 2.1.1 명령형 처리
**명령형 프로그래밍**은 상태를 변형하는 일련의 명령들로 구성된 프로그래밍 방식이다. 전형적인 for 루프가 명령형 프로그래밍의 훌륭한 예이다. 초기 상태를 설정하고, 되풀이할 때마다 일련의 명령을 실행한다.

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
**함수형 프로그래밍**은 프로그램을 수학 공식을 모델링하는 표현과 변형으로 기술하며, 가변 상태를 지양한다.

앞에서 언급한 **필터, 변형, 변환** 등의 논리적 분류도 저수준의 변형을 구현하는 함수들이었다. 개발자는 고계함수에 매개변수로 주어지는 함수를 이용하여 저수준의 작업을 커스터마이즈할 수 있다. 예를 들어 `map(function)`에서 `map`은 고계함수이고 `function` 은 매개변수로 주어지는 함수에 해당한다. 후자를 사용하여 각 요소에 변형을 적용한다는 뜻이다. 따라서 우리는 이 문제를 의사코드를 사용하여 개념화할 수 있으며, 함수형 언어는 이런 개념화된 해법을 세부사항에 구애받지 않고 모델링할 수 있게 해준다.

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
