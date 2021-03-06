# 출시 전략 고려하기

[기획서와 친해지기](https://github.com/Zeniuus/basics-of-server-development/blob/main/project/working-with-proposals.md) 문서에서 이야기한 것처럼, 개발자는 기획서를 보고 요구사항을 뽑아내고 이를 코드로 옮기는 작업에 능숙해야 한다. 이 과정에서 놓치기 쉬운 요구사항 중 하나는 바로 출시 전략이다. 출시 전략이란 기능을 언제, 어떤 방식으로 출시할 것인지에 대한 계획이다. 출시 전략은 종종 기능 외의 또 하나의 요구사항이 되곤 하는데, 출시 전략에 따라 개발해야 하는 요소들이 급격하게 늘어날 수도 있다. 따라서 설계 단계부터 출시 전략을 파악하고 이를 고려한 설계를 하는 것이 중요하다.

## 일반적인 출시 전략

일반적으로 사용하는 출시 전략은 크게 3가지 정도이다.

- 서버가 배포되자마자 출시하기
- 클라이언트가 배포되자마자 출시하기
- 특정 시점 이후에 출시하기

### 서버가 배포되자마자 출시하기

서버 내부적인 로직의 수정이나 기존 API로 수용할 수 있는 수준의 기능 추가/변경의 경우, 출시 일정을 빡빡하게 관리할 필요가 없으면 보통 서버가 출시되자마자 변경사항이 적용되도록 출시 전략을 짠다.

이 경우는 고려해야 할 사항이 딱히 없다. 서버 배포가 되자마자 기능이 출시되어도 되기 때문에, 그냥 기능을 잘 개발하고 너무 늦어지지 않게 서버를 배포하기만 하면 된다.

### 클라이언트가 배포되자마자 출시하기

클라이언트 UX 변경, 기존 API로 수용할 수 없는 수준의 기능 추가/변경 등의 경우, 보통 클라이언트가 출시되는 순간부터 기능이 적용되도록 출시 전략을 짠다.

새 기능은 서버를 필요로 할 수도 있고 아닐 수도 있다. 클라이언트 UX 변경과 같은 경우에는 클라이언트 작업만 필요로 하지만, 신규 기능의 출시는 서버에서 개발한 새로운 API를 필요로 할 것이다. 만약 새 기능이 서버를 필요로 한다면, 반드시 클라이언트보다 서버를 먼저 배포해놓아야 한다. 안 그러면 새로운 클라이언트 버전이 배포되어 새 API를 호출하려고 하면 404 에러가 뜰 것이다.

클라이언트보다 서버를 먼저 배포해야 하기 때문에, 서버를 먼저 배포해도 문제가 없게끔 기능을 개발하는 것이 중요하다. 변경사항이 기존 클라이언트에 영향을 주면 안 되는 경우 클라이언트의 버전을 보고 버전별로 서로 다른 로직을 적용하는 게 필요할 수도 있다.

### 특정 시점 이후에 출시하기

특정 시점부터 새로운/변경된 기능을 도입하고 싶은 경우, 추가적인 개발을 통해 출시 전략을 구현해야 한다.

여기서의 추가적인 개발은 보통 [feature flag](https://en.wikipedia.org/wiki/Feature_toggle)를 의미한다. feature flag란 코드의 실행 흐름을 제어할 수 있는 flag를 둠으로써 빠르게 실행 흐름을 변경할 수 있는 개발 기법이다. feature flag를 사용하면 미리 개발해둔 두 개 이상의 로직 중에 어떤 로직을 적용할지를 빠르게 변경할 수 있다. 대략 아래와 같은 구조이다.

```kotlin
fun doSomething() {
    if (featureFlag.isEnabled()) {
        runNewLogic()
    } else {
        runLegacyLogic()
    }
}
```

시간에 따라 변경되는 feature flag를 만들고 싶으면 아래와 같이 구현하면 된다.

```kotlin
fun doSomething() {
    if (featureFlag.isEnabled(now)) {
        runNewLogic()
    } else {
        runLegacyLogic()
    }
}

// featureFlag의 구현
fun isEnabled(now: Instant): Boolean {
    return newLogicStartAt <= now
}
```

언제나 위와 같이 깔끔한 if문으로 feature flag를 구현할 수 있는 것은 아니다. 로직을 분기타야 하는 부분이 여러군데 흩어져 있을 수도 있고, 현재 시각이 아닌 데이터의 생성 시각으로 분기를 타야할 수도 있다. 중요한 것은 feature flag를 나중에 걷어내기 쉽도록 응집도 높게 구현하는 것이다.

클라이언트의 UI 컴포넌트에 대해서도 feature flag를 구현해야 할 수도 있다. 예를 들어 특정 버튼을 특정 시각까지 숨겼다가 그 이후부터 노출해주고 싶을 수 있다. 이런 경우 클라이언트에서 feature flag를 구현해야 하는데, 이 때 사용할 flag를 클라이언트에서 직접 구현할 수도 있겠지만 서버에서 flag를 내려주는 게 보다 좋은 선택이다. 왜냐하면 출시 일정은 상황에 따라 변할 수 있는데, 클라이언트에 flag를 넣어서 출시하면 출시 일정을 변경하는 게 불가능하기 때문이다.

## 고려사항

출시 계획과 관련하여 몇 가지 도움이 될 만한 고려사항이 있다.

### 출시 계획 협상하기

출시 계획이 개발을 어렵게 만드는 경우, 다른 팀과의 협상을 시도해볼 수 있다. 예를 들어 현재 기획 상 feature flag를 구현해야 하는데, 이게 너무 어려운 작업이라면 서버 출시 직후에 적용하면 안 되겠냐고 물어볼 수 있다. 출시 계획도 기획의 일부임을 명심하자.

### 기능을 쪼개서 출시 전략을 고려하기

한 프로젝트 내에서도 각 기능마다 출시 전략이 다를 수 있다. 어떤 건 클라이언트가 출시되자마자 적용되어도 괜찮고, 어떤 건 반드시 시간 분기를 타야 할 수도 있다. 예를 들어 새로운 상품을 판매한다고 해보자. 구매 페이지는 미리 노출하되, 특정 시점부터 구매가 가능하게 막고 판매 시작 전에는 구매 버튼을 누르면 커밍순 팝업을 보여주려고 한다. 이 때 출시 계획은 다음과 같이 되어야 할 것이다.

- 구매 페이지로 접근하는 버튼 - 클라이언트가 배포되자마자 출시되어야 함
- 구매 페이지 - 웹서버가 먼저 배포되어 있어야 함
- 구매 API - 시간 분기를 태워야 함

### 팀의 배포 일정을 고려하기

PR을 메인스트림에 통합할 때는 개발하는 기능의 출시 전략과 팀의 출시 일정을 잘 고려해야 한다. 클라이언트가 배포되자마자 출시되는 기능이나 feature flag를 구현한 경우에는 보통 아무때나 메인스트림에 PR을 통합해도 괜찮다. 하지만 서버가 배포되자마자 기능이 풀리는 경우, 예정된 서버 배포일이 언제인지 유의깊게 살펴서 메인 스트림에 머지해야 한다. 잘못 통합한 경우 출시를 위해 롤백해야 하는 불편함이 생길 수 있다.

### 하위호환성 지키기

새로운 기능을 출시할 때는 반드시 하위호환성이 지켜져야 한다. 이에 대한 자세한 내용은 [상위호환성과 하위호환성](https://github.com/Zeniuus/basics-of-server-development/blob/main/deployment/forward-and-backward-compatibility.md#api%EC%9D%98-%ED%98%B8%ED%99%98%EC%84%B1) 문서를 참고하자.
