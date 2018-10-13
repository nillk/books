## Chapter 7. 실용적 사고
### 7.1 자바 8
자바 8은 언어에 고계함수를 그냥 덧붙이지 않고, 교묘하게 기존의 인터페이스들이 함수형 기능을 사용할 수 있도록 만들었다.

자바 8의 스트림을 사용하면, `collect()`나 `forEach()`처럼 출력을 발생하는 함수(종결 작업 terminal operation 이라고 부른다)를 호출할 때까지 다른 함수들을 연결해서 구성할 수 있다. 자바는 이미 언어가 가지고 있는 클래스와 컬렉션에 *맵*과 *리듀스*와 같은 함수형 구조를 더하여, 컬렉션을 효율적으로 업데이트하는 문제를 처리한다.

#### 7.1.1 함수형 인터페이스
Runnable 이나 Callable같이 메서드를 하나만 가지는 인터페이스를 흔히 단일 추상 메서드 (Single Abstract Method) 인터페이스라고 부른다. 함수형 인터페이스는 람다와 SAM이 상호작용할 수 있게 해준다. 하나의 함수형 인터페이스는 하나의 SAM을 포함하며, 여러 개의 디폴트 메서드도 함께 포함할 수 있다. 함수형 인터페이스는 기존 SAM 인터페이스가 전통적인 익명 클래스 인스턴스를 람다 블록으로 대체할 수 있게 개선해준다.

자바 8에서는 인터페이스에 *디폴트 메서드*를 지정할 수 있다. 디폴트 메서드는 인터페이스 안에 `default`로 표시되는 공개(public), 비추상(nonabstract), 비정적(nonstatic) - 본문이 있는 - 메서드이다. 디폴트 메서드를 덧붙일 수 있는 기능은 흔히 사용되는 mixin과 유사하고, 자바에 더해진 좋은 기능이다.

mixin은 다른 클래스에서 사용될 메서드를 정의하지만, 그 클래스의 상속 체계에 포함되지 않은 클래스를 지칭한다.

#### 7.1.2 옵셔널
자바 8에서 `min()`과 같은 내장 메서드는 값 대신 `Optional`을 리턴한다. `Optional`은 오류로서의 `null`과 리턴 값으로서의 `null`을 혼용하는 것을 방지한다.

#### 7.1.3 자바 8 스트림
스트림은 여러모로 컬렉션과 비슷하지만 다음과 같은 중요한 차이가 있다.
* 스트림은 값을 저장하지 않으며, 종결 작업을 통해 입력에서 종착점까지 흐르는 파이프라인처럼 사용된다.
* 스트림은 상태를 유지하지 않는 함수형으로 설계되었다. 일례로 `filter()` 작업은 밑에 깔린 컬렉션을 바꾸지 않고 필터된 값의 스트림을 리턴한다.
* 스트림 작업은 최대한 게으르게 한다.
* 무한한 스트림이 가능하다.
* `Iterator` 인스턴스처럼 스트림은 사용과 동시에 소멸되고, 재사용 전에 다시 생성해야 한다.

스트림 작업은 *중간 작업* 또는 *종결 작업*이다. 중간 작업은 새 스트림을 리턴하고, 게으르다. 종결 작업이 요청되었을 때에야 필요한 값만을 리턴하는 스트림을 만든다. 종결 작업은 스트림을 순회하여 값이나 부수효과를 낳는다.

### 7.2 함수형 인프라스트럭처
#### 7.2.1 아키텍처
함수형 아키텍처는 불변성이 그 중심에 있다. 함수형 사고로의 전환의 이점은 코드에서 생기는 변화가 제대로 이루어졌는지 확인할 테스트가 있다는 사실을 인지하게 되는 것이다. 여기서의 테스트 목적은 변이를 확인하는 것이다. 불변 클래스들은 생성 시에만 변화가 있기 때문에 테스트가 간단하다.

불변 객체는 자동적으로 스레드 안전하기 때문에 동기화의 문제도 없다. 모든 초기화가 생성 시에 일어나기 때문에 그 상태를 알 수 없거나 예외 때문에 잘못된 상태를 가질 가능성도 없다. 조슈아 블록은 이를 실패의 원자성이라고 부른다. 가변성에 의존하는 성패가 일단 객체가 생성되는 시점에 이미 해결된다는 뜻이다.

자바 클래스를 불변형으로 만들려면 아래와 같이 해야 한다.

##### 모든 필드를 final로 선언
자바에서 `final`로 선언된 필드들은 선언 시나 생성자 내부에서 초기화해야 한다.

##### 클래스를 final로 선언해서 오버라이드를 방지하라
클래스가 오버라이드 되면 메서드들도 오버라이드 될 수 있다. 가장 안전한 건 하위 클래스를 금지하는 것!

##### 인수가 없는 생성자를 제공하지 말라
불변성 객체의 모든 상태는 생성자가 정해야 한다. 그런데 상태가 없다면 객체가 필요할까? 상태가 없는 클래스의 정적 메서드로 충분할 때도 있다. 물론 디폴트 생성자를 제공하지 않는 건 자바 빈스 표준을 위반하지만 `setXXX` 메서드 때문에 어쨌든 자바 빈스는 불변일 수가 없다.

##### 적어도 하나의 생성자를 제공하라
디폴트 생성자가 없으므로 객체에 상태를 더할 수 있는 마지막 기회를 제공해야 한다!

##### 생성자 외에는 변이 메서드를 제공하지 말라
`setXXX` 를 제공하지 않아야 하는 것은 물론 가변 객체의 참조를 리턴하지 않아야 한다.

##### CQRS (명령-쿼리 간 책임 분리)
[그레그 영](http://codebetter.com/gregyoung)  
[마틴 파울러](http://bit.ly/fowler-cqrs)

전통적인 애플리케이션 아키텍처에서는 모델 부분이 비즈니스 규칙이나 검증을 담당한다. 개발자는 **읽기**와 **쓰기**의 혼합된 의미를 모델 전역에서 관리해야 한다. CORS는 *데이터베이스 업데이트를 담당하는 부분*과 *자료 표현과 보고를 처리하는 부분*을 분리함으로써 아키텍처의 일부를 단순화한다. 분리된 모델은 곧 분리된 논리적 프로세스를 의미한다.

*읽기*를 *변이*로부터 분리하면 논리적으로 단순해진다. *읽기 쪽에서는 모든 것을 불변형으로 처리할 수 있다.*

#### 7.2.2 웹 프레임워크
웹 프로그래밍은 웬 전반을 *요구를 응답으로 바꾸는 일련의 변형*으로 볼 수 있기 때문에 함수형 프로그래밍에 잘 어울린다.

##### 경로 설정 프레임워크
함수형을 포함한 현대 웹 프레임워크들은 경로 설정 (routing) 라이브러리를 사용하여 경로 설정을 애플리케이션 기능으로부터 분리시킨다.

##### 함수를 목적지로 사용
웹상의 요구를 `Request`로 받아서 `Response`를 리턴하는 함수로 생각

##### 도메인 특화 언어
마틴 파울러는 DSL을 *좁은 문제 도메인에 적용되는 표현으로 제한된 프로그래밍 언어*로 정의한다. DSL은 다양한 언어에서 자주 사용되는 접근법이기도 하지만, 함수형 언어들은 특히 선언적 코드를 선호하며, 이것은 종종 DSL의 목적 그 자체이기도 하다.

##### 빌드 도구와의 밀접한 연동
대부분의 함수형 언어는 IDE에서의 사용에 국한되지 않고 커맨드라인 빌드 도구와 밀접하게 연결되어 있다.

#### 7.2.3 데이터베이스
클로저 커뮤니티에서 상업적 NoSQL 데이터베이스에 처음으로 내놓은 [데이토믹](http://www.datomic.com)은 들어오는 모든 사실들에 시간을 붙여서 저장하는 *불변성 데이터베이스*이다. **데이터** 대신에 **값**을 저장함으로써, 데이토믹은 저장소를 상당히 효율적으로 사용한다. 데이토믹은 정보에 시간의 개념을 첨부한다.

##### 모든 스키마와 데이터의 변화를 영원히 기록하기
스키마 조작을 포함한 모든 것이 저장 유지되어 데이터베이스의 이전 버전으로 돌아가는 것이 간단해졌다.

##### 읽기와 쓰기의 분리
데이토믹은 자연스럽게 **읽기**와 **쓰기** 작업을 분리한다. 즉 내부적으로 *CORS 아키텍처*이다.

##### 이벤트 주도 아키텍처를 위한 불변성과 타임스탬프
이벤트 주도 아키텍처는 애플리케이션 상태 변화를 이벤트 스트림으로 저장한다. 모든 정보를 타임스탬프와 함께 저장하는 데이터베이스가 여기에 적합하다.