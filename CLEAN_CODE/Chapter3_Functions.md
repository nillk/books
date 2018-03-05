## Chapter 3. 함수
***함수는 프로그램의 가장 기본적인 단위***
* 읽기 쉽고 이해하기 쉬운 함수
* 의도를 분명히 표현하는 함수

### 3.1 작게 만들어라

명백한 함수. 각 함수가 이야기 하나를 표현
* if/else, while 등에 들어가는 블록은 한 줄로
* 들여쓰기는 2단을 넘어가지 않게

### 3.2 한 가지만 해라
> 함수는 한가지를 해야 한다.  
한 가지를 잘해야 한다.  
한 가지만 해야 한다.  

문제! 그 **한 가지** 가 뭘까
* 지정된 함수 이름 아래 **추상화 수준**이 하나
* 우리가 함수를 만드는 이유를 생각해봐!  
큰 개념(다시 말해, 함수 이름)을 다음 추상화 수준에서 여러 단계로 나눠 수행하기 위함 아님?

단순히 다른 표현이 아니라 *의미 있는 이름으로 다른 함수를 추출할 수 있다면* 그건 이미 여러 가지 작업을 하고 있다는 의미!

### 3.3 함수 당 추상화 수준은 하나로
추상화 수준을 섞으면 읽는 사람이 헛갈림 (깨진 창문의 법칙. 걷잡을 수 없어진다!)

**위에서 아래로 코드 읽기**  
*함수는 위에서 아래로 이야기처럼 읽히도록. 즉, 추상화 수준이 한 단계씩 낮아지도록.*

***짧으면서 한 가지만 하는 함수***

### 3.4 switch
switch는 작게 만들기 힘듦 (if/else도 물론 T_T)  
*본질적으로 N가지를 처리하기 때문에 **한 가지**만 처리하는 것도 불가능*

> 저차원 클래스에 숨기고 반복하지 않도록 (Polymorphism)

### 3.5 서술적인 이름을 사용하라
길어도 O.K. 짧고 어려운 것보다야! 서술적인 이름을 사용하면 개발자의 머리 속에서도 설계가 뚜렷해짐. 일관성 잊지 말기!

### 3.6 함수 인수
이상적인 인수 개수는 **0**. 인수는 개념을 이해하기 어렵게 만듦.

`includeSetupPageInto(newPageContent);` 보다는 `includeSetupPage();` 으로

* 함수 이름과 인수가 추상화 수준이 다름
* 중요하지 않은 세부 사항(StringBuffer)를 알아야 함

테스트 관점에서 보면 인수는 더 어려워!

#### 많이 쓰는 단항 형식
인수 1개인 두 가지 경우
* 인수에 질문을 던짐  
```java
boolean fileExists("MyFile")
```
* 인수를 뭔가로 변환해 결과 반환  
```java
InputStream fileOpen("MyFile")
```

#### 플래그 인수
***추하다...***  
함수가 한꺼번에 여러가지를 처리한다고 대놓고 공표하는 셈이잖아!!!

#### 이항 함수
인수 1개인 함수보다 이해하기 어렵다.  
`writeField(name)` <-> `writeField(outputStream, name)`

우리가 `outputStream`을 무시해야 한다는 사실을 깨닫는 데는 시간이 필요하다. 그리고 그 사실은 문제를 일으킨다. *무시한 코드에는 오류가 숨어들기 때문에.*

#### 삼항 함수
어려웡!

#### 인수 객체
인수가 2~3개 필요하다면 일부를 독자적인 클래스 변수로 선언할 가능성을 따져보자. 위의 코드를 아래의 코드로 바꾸는 것이 더 낫다.

```java
Circle makeCircle(double x, double y, double radius)
```

```java
Circle makeCircle(Point center, double radius)
```

x와 y를 묶는 건 결국 개념(center)을 포함한다.

#### 인수 목록
인수 개수가 가변적인 함수도 필요
```java
String format(String format, Object... args)
```
실제로 이항 함수

#### 동사와 키워드
함수의 의도와 인수의 순서와 의도를 제대로 표현하려면? ***좋은 함수 이름***
* 단항 함수는 함수와 인수가 동사/명사 쌍을 이뤄야 함  
```java
writeField(name)
```
* 함수 이름에 인수 이름을 넣어보자!  
```java
assertExpectedEqualsActual(expected, actual)
```

### 3.7 부수 효과를 일으키지 마라
부수 효과는 거짓말! 한 가지를 하겠다고 약속하고는 다른 것도 한다! (예상치 못하게 클래스 변수나 전역 변수를 수정하는 일도 존재) 그리고 이는 대부분 temporal coupling, order dependency 등 초래

***함수 명에 나와있지 않은 일 하지마!***
* `checkPassword`함수 내부에서 `session.initialize()` 한다면?  
함수 이름만 보고 호출했다가 세션 정보 bye~
* 일시적 결합 초래  
특정 상황 즉, 세션을 초기화해도 괜찮은 경우에만 호출 가능

최소한 이름이라도 바꾸자... (물론, 한 가지만 한다는 규칙에는 위배되지만)

#### 출력 인수
```java
appendFooter(s);
```
얘는 뭘까? 함수 선언부를 찾아봐야 알 수 있겠지?! 차라리 `report.appendFooter()` 같은 형태로 만들어.

### 3.8 명령과 조회를 분리하라
함수는
1. 무언가를 수행하거나 (개체 상태를 변경하거나)
2. 무언가에 답하거나 (개체 정보를 반환하거나)

하나만 해라.
```java
if(set("username", "unclebob"))
```
이거 뭔데. Set 되어 있는거야? 설정하는 거야? 뭐야? 분리해야 함. 아래의 코드가 낫다.

```java
if (attributeExists("username")) {
  setAttribute("username", "unclebob");
}
```

### 3.9 오류 코드보다 예외를 사용하라
명령 함수에서 오류코드를 반환하는 방식은 명령/조회 분리 규칙을 미묘하게 위반함. 자칫하면 if문에서 명령을 표현식으로 사용 & 코드 중첩
```java
if (deletePage(page) == E_OK)
```
이어지는 else 구문에 바로 오류처리 해줘야 함. 다시 말해 if-else의 향연이 펼쳐짐 (^_\^) try-catch 구문으로 묶는 게 깔끔

But!!! try-catch 구문은 본질적으로 추함. 코드 구조에 혼란을 일으키고, 정상/오류 동작을 뒤섞음. 그러니 분리해

```java
try {
  deletePage(page);
  registry.deleteReference(page.name);
  configKeys.deleteKey(page.name.makeKey());
} catch (Exception e) {
  logger.log(e.getMessage());
}
```

```java
public void delete(Page page) {
  try {
    deletePageAndAllReferences(page);
  } catch (Exception e) {
    logError(e);
  }
}

private void deletePageAndAllReferences(Page page) throew Exception {
  deletePage(page);
  registry.deleteReference(page.name);
  configKeys.deleteKey(page.name.makeKey());
}

private void logError(Exception e) {
  logger.log(e.getMessage());
}
```

`delete`는 예외 처리만하고 실제 동작은 `deletePageAndAllReferences`가!

**오류 처리도 한 가지 작업이다.**  
함수는 **한 가지**만! 오류 처리도 **한 가지**만!

#### Error.java 의존성 자석
오류 코드를 반환한다는 소리는 어디선가 오류 코드를 정의한다는 뜻. 만약 오류 코드가 변하면 이걸 쓰는 클래스들을 모두 컴파일하고 배치해야 함. 그리고 프로그래머는 그게 귀찮아서 새 오류코드 정의 안 함...

그냥 Exception써 *OCP (Open Closed Principle)*

### 3.10 반복하지 마라
중복은 악  
코드 길이가 늘어날 뿐 아니라 알고리즘이 변하면 모두 다 손봐야 함. 자, 그 중에 하나 빠뜨리면...?  
상속, 구조적 프로그래밍, AOP, COP ... 다 중복을 제거하려는 전략임!

### 3.11 구조적 프로그래밍
Edsger Dijkstra's rules of Structured programming
* 모든 함수와 함수 내 모든 블록에 입구와 출구가 하나만 존재해야 함
* 함수는 return 문이 하나여야 함
* Loop 안에서 break, continue 쓰지마. goto는 **NEVER!**

But, 함수가 작다면 되레 의도를 표현하기 쉬워질 수 있음. 큰 함수에서는 피하는 게 좋음. 알아서 잘 쓸 것 ^_\^

### 3.12 함수를 어떻게 짜죠?
> 소프트웨어를 짜는 행위는 여느 글짓기와 비슷하다. 먼저 생각을 기록한 후 읽기 좋게 다듬는다. 초안은 대개 서투르고 어수선하므로 원하는 대로 읽힐 때까지 말을 다듬고 문장을 고치고 문단을 정리한다.

> 함수를 짤 때도 마찬가지다. 처음에는 길고 복잡하다. 들여쓰기 단계도 많고 중복된 루프도 많다. 인수 목록도 아주 길다. 이름은 즉흥적이며 코드를 중복된다. 하지만 이 서투른 코드를 빠짐없이 테스트 하는 단위 테스트 케이스를 분명히 만든다.

> 그 후에 코드를 다듬고, 함수를 만들고, 이름을 바꾸고, 중복을 제거한다. 메소드를 줄이고, 순서를 바꾼다. 때로는 전체 클래스를 쪼개기도 한다. 그 도중에 코드는 항상 단위 테스트를 통과한다.

> 종내에는 이 장에서 말한 규칙을 따르는 함수를 얻는다. 처음부터 탁 짜내지 않는다. 그게 가능한 사람은 없으리라.

### 3.13 결론
함수는 그 언어에서 동사이며, 클래스는 명사다. 프로그래밍 기술은 언제나 언어 설계의 기술이다. 이 장은 함수를 잘 만드는 기교를 소개했다. 하지만 진짜 목표는 시스템이라는 이야기를 풀어가는 데 있다는 사실을 명심하라. 작성하는 함수가 분명하고 정확한 언어게 깔끔하게 맞아야 이야기를 풀어가기가 쉬워진다.
