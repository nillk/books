# Chapter 2. 의미 있는 이름

## 2.1 들어가면서

이름은 아주 오만데에 다 쓰이지... 그러니까 잘 지어보자 ^\_^

## 2.2 의도를 분명히 밝혀라

**좋은 이름을 짓는데 걸리는 시간 &lt; 좋은 이름으로 절약하는 시간**

> 변수\(함수 or 클래스\)의 존재 이유? 수행 기능? 사용 방법?

이름으로 답할 수 있어야 한다.

## 2.3 그릇된 정보를 피하라

> `AccountList`가 있다고 치자. 그런데 사실 이 객체가 List형태가 아니라면?

`Group`, `bunchOfAccounts`, `Accounts`등으로 쓰자. 습관적으로 List라고 이름짓지 말 것.

흡사한 이름도 지양할 것

일관성이 떨어지는 명명도 그릇된 것

## 2.4 의미 있게 구분하라

`tmp01` 이런 거 쓰지 마

**Noise word** 추가하지 마

* Product
* ProductInfo
* ProductData

무슨 의미? 무슨 정보를 알 수 있지? 개념을 구분해야 한다.

## 2.5 발음하기 쉬운 이름 써라

genymdhms......

## 2.6 검색하기 쉬운 이름 써라

* 긴 이름이 짧은 이름보다 낫다. \(검색을 염두에 둬. 좋은 IDE 뒀다 어디다 쓰니\)
* 검색하기 쉬운 이름이 상수보다 낫다.
* 이름길이는 범위 크기\(scope\)에 비례해야 한다.

  그냥 숫자 7? Search로 찾을 수 없음. 모래사장에서 바늘 찾기

## 2.7 인코딩을 피하라

현대의 IDE를 사용하는 Java 개발자에겐 필요 없는 얘기긴 함 ☞☜  
헝가리식 표현법 피할 것\(제일 앞에 타입 붙이는 방법\)

## 2.8 자신의 기억력을 자랑하지 마라

세 번 쓰기 _**명료함 명료함 명료함**_  
다른 사람이 코드를 볼 수 있다는 걸 염두에 두자

## 2.9 클래스 명

명사 or 명사구 쓰세요.  
Manager, Processor, Data, Info 같은 거 쓰지마 \(왜죠? 음. 명료하지 않아서인듯?\)

## 2.10 메소드 명

동사 or 동사구 쓰세요.  
Constructor를 overload할 때는 static Factory method 쓰는 것을 추천. method는 인수를 설명하는 이름으로.  
예를 들어 아래의 코드가 위의 코드보다 낫다

```java
Complex fulcrumPoint = new Complex(23.0);
Complex fulcrumPoint = Complex.FromRealNumber(23.0);
```

## 2.11 기발한 이름은 피하라

재미있는 것보단 명확한 것  
HolyHandgernade... 뭔데 이거...?

## 2.12 개념 하나에 단어 하나

**일관성 있는 어휘를 사용할 것**  
똑같은 메소드를 클래스마다 `fetch`, `retrieve`, `get`... 이렇게 쓰지 말 것. \(같은 행동을 하는 메소드인데 어떤 클래스는 `fetch`, 다른 클래스는 `retrieve` 이렇게 쓰는 것을 말함\) 마찬가지로 `DeviceManager`랑 `ProtocolController`는 뭐가 다를까? 같은 일을 하는 애면 같은 단어를 쓰자.

## 2.13 말장난 NO

**한 단어를 두가지 목적으로 쓰지 말 것**  
2.12 이야기\(개념 하나에 단어 하나\) 했더니 다들 메소드명을 몽땅 `addXXX`라고 했어 T\_T 기존 add와 맥락이 다르면 당연히 `insert`나 `append`를 쓰는 게 맞지!!!

## 2.14 해법 영역에서 사용하는 이름을 사용하라

결국 내 코드를 읽을 사람도 프로그래머!  
즉, 전산 용어, 알고리즘 명, 패턴 명, 수학 용어 등 OK

_기술적 개념은 기술적 이름으로 표현_ 하는 것이 가장 좋다

## 2.15 문제 영역과 관련 있는 이름을 사용하라

적절한 프로그래머 용어가 없다면 문제 용어에서 가져오자. 그렇다면 분야 전문가에게 물어볼 수 있음

### 해법 영역 vs 문제 영역

우수한 프로그래머라면 이 둘을 구분할 수 있어야 한다. 더 관련이 깊은 것을 사용하자!

## 2.16 의미 있는 맥락을 추가하라

> `firstName`, `lastName`, `street`, `houseNumber`, `city`, `state`....

주소인 걸 알 수 있음  
그런데 만약 어떤 메소드가 그냥 `state`만 쓰면? 주소인 걸 알 수 있을까? _No!_  
정 안되면 `addFirstName`, `addLastName`, `addStreet`....로 쓰면 그나마 분명해진다. 물론, 그냥 `Address`라는 Class를 만드는 게 더 좋다. Address라는 큰 개념에 속한다는 걸 분명하게 알 수 있으니까.

## 2.17 불필요한 맥락 없앨 것

**짧은 이름이 긴 이름보다 낫다.**  
물론 의미가 분명한 경우에 한해서. 예를 들어 Gas Station Deluxe 프로그램을 만든다고 생각해보자. 모든 Class에 `GSDXXX`.... 이러지 말자. IDE를 방해하고 있어!

## 2.18 마치면서

좋은 이름을 선택하려면 설명하는 능력이 뛰어나야 하고\(...\) 문화적인 배경이 같아야 해 어려워 쥬금  
하지만 이름을 바꾸는 것을 두려워하지 말자. 더 나은 코드가 될 수 있다구!

