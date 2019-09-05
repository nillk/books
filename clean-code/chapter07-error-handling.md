# Chapter 7. 오류 처리

깨끗하고 튼튼한 코드에 한 걸음 더 다가가는 단계로 우아하고 고상하게 오류를 처리하는 기법과 고려 사항 몇 가지를 소개한다.

## 7.1 오류 코드보다 예외를 사용하라

예외를 지원하지 않는 언어는 오류 플래그를 설정하거나 호출자에게 오류 코드를 반환해야 했다. 이런 방법은 _함수를 호출하자마자 오류를 확인_ 해야 하기 때문에 호출자 코드가 복잡해진다. 또한 잊어버리기 쉽다. 따라서 예외를 던지는 편이 _논리 코드와 오류 처리 코드가 분리되어_ 더 깔끔하다.

## 7.2 Try-Catch-Finally 문부터 작성하라

어떤 면에서 `try` 블록은 트랜잭션과 비슷한데, `try` 블록 안에서 무슨 일이 생기든지 `catch` 블록은 프로그램 상태를 일관성있게 유지해야 한다. 그러므로 _예외가 발생할 코드를 짤 때는 try-catch-finally 문으로 시작_ 하는 편이 낫다. 그러면 자연적으로 `try` 블록의 트랜잭션 범위부터 구현하게 된다.

```java
public List<RecordedGrip> retrieveSection(String sectionName) {
    try {
        FileInputStream stream = new FileInputStream(sectionName);
        // write rest of the logic
        stream.close()
    } catch (FileNotFoundException e) {
        throw new StorageException("retrieval error", e);
    }
    return new ArrayList<RecordedGrip>();
}
```

## 7.3 미확인 예외를 사용하라

Java의 Checked Exception은 OCP\(Open Closed Principle\)을 위반한다. 단적인 예로 하위 메서드에서 checked exception을 추가/변경하면 그 메서드를 사용하는 최종 메서드까지 모든 메서드가 _1. `catch` 블록에서 새로운 예외를 처리하거나 2. 선언부에 `throws` 절을 추가해야 한다._ 즉, 하위 단계에서 코드를 변경하면 상위 단계 메서드를 전부 고쳐야 한다는 소리다. 모든 메서드가 최하위 메서드에서 던지는 예외를 알아야 하므로 캡슐화가 깨진다. 때로는 checked exception도 유용하지만, 일반적으로는 의존성이라는 비용이 이익보다 크다.

## 7.4 예외에 의미를 제공하라

전후 상황을 충분히 덧붙이자. 오류 메시지에 정보와 실패한 연산 이름과 유형도 언급하자. 로깅을 한다면 `catch` 블록에서 오류를 기록하도록 충분한 기록은 넘기자.

## 7.5 호출자를 고려해 예외 클래스를 정의하라

일반적으로 오류를 처리하는 방식은 아래와 같다. 1. 오류를 기록한다 2. 프로그램을 계속해서 수행해도 좋은지 확인한다

오류는 발생한 위치로 분류할 수도 있고, 유형으로도 분류할 수 있다. 응용 프로그램에서는 이런 분류를 잘 정리해 오류를 잡아내어야 한다. 외부 라이브러리에서 오류를 던지는 경우 외부 라이브러리를 호출하는 부분을 감싸 그 안에서 하나의 예외 유형을 반환하게 할 수도 있다. 어떤 예외들을 던지느냐, 어떻게 처리해야 하느냐에 따라 다르다.

## 7.6 정상 흐름을 정의하라

때로는 오류로 인한 _중단_ 이 적합하지 않을 때도 있다.

```java
try {
    MealExpenses expenses = expenseReportDAO.getMeals(employee.getID());
    m_total += expenses.getTotal();
} catch (MealExpensesNotFound e) {
    m_total += getMealPerDiem();
}

// 바꾸자!
public classPerDiemMealExpenses implements MealExpenses {
    public int getTotal() {
        // 기본 일일 식비 리턴
    }
}

MealExpenses expenses = expenseReportDAO.getMeals(employee.getID());
m_total += expenses.getTotal();
```

위와 같이 변경한 것을 특수 사례 패턴 _Special case pattern_ 이라고 한다. 클래스를 만들거나 객체를 조작해 특수 사례를 처리한다. 그러면 클래스나 객체가 예외적인 상황을 캡슐화해서 처리하므로 호출자가 예외를 처리할 필요가 없다.

## 7.7 `null`을 반환하지 마라

흔히 오류를 유발하는 행위가 `null`을 반환하는 습관이다. `null`을 반환하면 오히려 일거리가 늘어나며, 호출자에게 문제를 떠넘긴다. 한 번이라도 `null` check를 빼먹는다면? :scream:

`null` 대신 예외를 던지거나 특수 사례 객체를 반환하자. 외부 API가 `null`을 반환한다면 감싸기 메서드를 구현해 예외를 던지거나 특수 사례 객체를 반환하는 방법을 고려하자.

## 7.8 `null`을 전달하지 마라

`null`을 반환하는 것도 나쁘지만, `null`을 전달하는 방식은 더 나쁘다. 미리 `null` check를 하거나 `assert`를 해서 `NullPointerException`을 방지할 수는 있겠지만 이런 경우 예외를 어떻게 처리해야 할지 결정해야 한다. 어쨌든 오류는 발생한다. 대다수 프로그래밍 언어는 인수로 넘어오는 `null`을 적절히 처리하는 방법이 없다. 그러니 애초에 `null`을 넘기지 못하도록 금지하는 것이 합리적이다.

## 7.9 결론

클린 코드는 읽기도 좋아야 하지만 _안정성도 높아야_ 한다. 오류 처리를 프로그램 로직과 분리하는 정도에 따라서 독립적인 추론이 가능해지며 코드 유지보수성도 크게 높아진다.

