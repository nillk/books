## Chapter 8. 폴리글랏과 폴리패러다임
함수형 프로그래밍 패러다임은 문제과 그것을 푸는 데 사용되는 도구에 관한 사고의 틀이라고 할 수 있다. 많은 현대 언어들은 폴리패러다임 (호은 멀티패러다임)이다.

##### 직교
직교는 컴포넌트들이 서로 영향을 주지 않는다는 뜻이다. 예를 들어 함수형 프로그래밍과 메타프로그래밍은 그루비에서 서로 간섭하지 않기 때문에 직교한다고 할 수 있다. 메타프로그래밍을 사용해도 함수형 구조물에 아무런 영향이 없고, 반대의 경우도 마찬가지이다. 서로 직교한다고 공존할 수 없다는 것은 아니다. 단지 서로 간섭하지 않는다는 것 뿐이다.

### 8.1 함수형과 메타프로그래밍의 결합
그루비에서 `ExpandMetaClass`는 기존의 클래스 (언어가 제공하는 기본 클래스 포함) 메서드를 덧붙일 수 있게 한다. 각 메서드를 정의하는 코드 블록 내에서 미리 설정한 `delegate`에 접근할 수가 있다. `delegate`는 이 클래스의 메서드를 호출하는 객체의 값을 대리한다.

메타프로그래밍 메서드는 처음 호출하기 전에 반드시 정의해야 한다. 

```groovy
class IntegerClassifierTest {
  static {
    Integer.metaClass.isPerfect = {->
      NumberClassifier.isPerfect(delegate)
    }

    Integer.metaClass.isAbundant = {->
      NumberClassifier.isAbundant(delegate)
    }

    Integer.metaClass.isDeficient = {->
      NumberClassifier.isDeficient(delegate)
    }
  }
}
```

### 8.2 메타프로그래밍을 통한 자료형의 매핑
자료형들 사이에 메타프로그래밍 매핑을 가미하면 다른 라이브러리들을 더 깊이 녹아들게 할 수 있다. 
