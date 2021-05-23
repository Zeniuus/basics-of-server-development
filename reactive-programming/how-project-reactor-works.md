# Project Reactor 동작 원리 & 쓰레드 관리

## 동작 원리

### 구성 요소

- `Publisher`: 구독의 대상이 되는 인터페이스. 이벤트를 방출하는 책임을 가지고 있다.
    - `subscribe(Subscriber)` - `Subscriber`가 `Publisher`를 구독하는 메소드.
- `Subscriber`: `Publisher`을 구독하는 인터페이스. `Publisher`가 방출한 이벤트를 수신하여 원하는 작업을 진행하는 책임을 가지고 있다.
    - `onSubscribe()` - `Subscriber`가 `Publisher`을 구독 완료했을 때 호출되는 이벤트 callback 메소드. 아래의 상호작용 부분에서 더 자세히 다룬다.
    - `onNext()`, `onError()`, `onComplete()` - `Publisher`가 `Subscriber`에게 이벤트를 전파할 때 부르는 callback 메소드.  이벤트가 정상적으로 전파되면 `onNext()`, 에러가 발생하면 `onError()`,  이벤트를 모두 전파하면 `onComplete()`가 호출된다.
- `Subscription`: `Publisher`와 `Subscriber` 의 구독 관계를 표현하는 객체.
    - `request()`, `cancel()` - 아래의 상호작용 부분에서 더 자세히 다룬다.

### 상호작용

위 세 가지 컴포넌트의 상호작용은 크게 구독 단계와 이벤트 전파 단계라는 두 가지 단계로 이루어진다. (구독 단계 / 이벤트 전파 단계는 필자가 임의로 명명한 것으로, 일반적으로 사용되는 이름이 아닐 수 있다)

1. 구독 단계
    1. `Subscriber`가 `Publisher`을 subscribe한다. (`Publisher.subscribe(Subscriber)`)
    2. `Publisher`가 `Subscriber`와의 subscription을 만든다.
    3. `Publisher`가 `Subscriber`에게 subscribe가 완료되었다고 이벤트를 전파한다. (`Subscriber.onSubscribe(Subscription)`)
2. 이벤트 전파 단계
    1. `Subscriber`가 `Subscription`에서 데이터를 요청한다. (`Subscription.request()`)
    2. `Subscription`은 값이 준비되면 `Subscriber`에게 데이터를 전파한다. (`Subscriber.onNext()`)
    3. `Subscription`은 모든 데이터를 전파했을 경우 `Subscriber`에게 데이터 전파가 끝났다고 알린다. (`Subscriber.onComplete()`)

`Mono.map`과 같이 `Mono`나 `Flux`가 제공하는 여러가지 유틸 함수를 사용하더라도 큰 흐름은 달라지지 않는다. 단지 `Publisher.subscribe()`, `Subscriber.onSubscribe()`, `Subscription.request()`, `Subscriber.onNext()`가 연쇄적으로 불릴 뿐이다.

![Spring Reactor 동작 원리](/reactive-programming/images/how-spring-reactor-works-1.png)

여기서 한 가지 강조하고 넘어가고 싶은 것이 있는데, 바로 `Mono.fromCallable()`이나 `Mono.map()`과 같은 operator의 로직이 실행되는 타이밍이다. 위의 상호작용을 보면 operator의 로직이 구독 단계가 아닌 이벤트 전파 단계에서 실행되는 것을 알 수 있다. 구독 단계에서는 단지 `Subscription`을 `Subscriber`에게 전달하는 일만 일어나고, 실제 operator의 로직은 이벤트 전파 단계, 좀 더 구체적으로는 위 그림의 4번째 단계인 `onNext()` chain에서 일어난다. 이 이야기는 아래 쓰레드 관리 항목에서 중요하게 다룰 예정이다.

### `Subscription` 인터페이스는 왜 필요한가?

필자가 Spring Reactor 내부 코드를 뜯어보기 시작했을 때 가장 이해가 안 됐던 부분 중 하나는 `Subscription` 인터페이스의 존재였다. `Publisher`와 `Subscriber`만 있으면 reactive programming이 가능할 것 같은데, 왜 굳이 `Subscription`이라는 인터페이스를 따로 만들었을까?

이는 한 `Publisher`를 여러 `Subscriber`가 구독할 수 있기 때문이다. 이것이 가능하려면 각 "구독"은 독립적인 상태와 컨텍스트를 가져야 한다. 1부터 10까지의 정수를 순서대로 방출하는 `Publisher`를 예시로 들어보자. 두 개의 `Subscriber`가 이 `Publisher`를 구독하면, 각 `Subscriber`는 서로 다른 페이스로 이벤트를 수신할 수 있다. 예를 들어 첫 번째 `Subscriber`는 3까지 받았고, 두 번째 `Subscriber`는 6까지 받았을 수 있다. 이 때 각 `Subscriber`가 다음 데이터를 `Publisher`에 요청하면 첫 번째 `Subscriber`는 4를 받고 두 번째 `Subscriber`는 7을 받아야 한다. 이것이 가능하려면 각각의 구독의 상태가 어디선가 관리되고 있어야 한다.

이 구독 상태의 관리를 `Publisher`가 직접 수행할 수도 있지만, 이러면 `Publisher`가 너무 많은 책임을 진다. 이는 `Publisher`와 `Subscriber`의 1대1 구독 관계를 표현하는 별도의 매핑 인터페이스를 만드는 것이 자연스럽다. 이것이 바로 `Subscription` 인터페이스이다.

### 왜 두 단계의 상호작용이 필요한가?

두 번째 의문은 왜 구독 단계와 이벤트 전파 단계를 분리했냐는 점이다. 결론부터 말하면 이는 backpressure를 구현하기 위함이다.

backpressure란, 이벤트를 전파하는 행위의 주도권을 `Publisher`가 아니라 `Subscriber`에게 쥐어주는 메커니즘이다. 즉, `Publisher`가 마음대로 이벤트를 전파하는 게 아니라 `Subscriber`가 요청했을 때만 이벤트를 전파하게 만드는 것이다. 이러한 메커니즘이 필요한 이유는 클라이언트가 스스로의 CPU/메모리 사용 상황에 따라 수신하는 이벤트의 양을 조절하게 해주기 위함이다. `Publisher`가 엄청나게 많은 이벤트를 한꺼번에 내려줄 경우 `Subscriber` 쪽에서는 OOM이 발생하거나 과도하게 많은 CPU를 사용하여 다른 작업에 문제가 발생할 수도 있다. 이러한 문제를 방지하기 위해 `Subscriber`가 요청했을 때, `Subscriber`가 요청한 만큼의 이벤트만 `Publisher`가 전파하는 메커니즘이 필요하다.

상호작용이 한 단계로 이루어지면 backpressure를 구현할 수 없다. `Subscriber`가 `Publisher.subscribe()`를 호출하는 순간부터 `Subscriber`는 아무것도 할 수 없다. 이벤트 방출에 대한 모든 제어권은 `Publisher`가 가져가게 된다. 하지만 상호작용이 구독 단계와 이벤트 전파 단계로 나뉘게 되면 위에서 본 것처럼 `Subscriber`가 이벤트를 `request()` 할 수 있게 되므로, `Subscriber`가 이벤트 방출을 조절할 수 있게 된다.

## 쓰레드 관리

### 동작 원리

Spring Reactor를 사용할 때 가장 중요한 것 중 하나는 쓰레드 관리이다. 왜냐하면 Spring Reactor가 주로 사용되는 상황이 네트워크 호출이나 무거운 연산 등 쓰레드 관리가 필요한 상황이기 때문이다.

Spring Reactor에서는 쓰레드 관리를 위해 크게 `subcribeOn()`과 `publishOn()`이라는 두 가지 메소드를 제공하는데, 처음에는 각각을 어떤 상황에서 사용해야 하는지 알기가 쉽지 않다. 하지만 Spring Reactor의 동작 원리를 알면 이 두 가지의 사용처를 쉽게 판단할 수 있다.

두 가지 메소드의 동작 방식을 간단히 알아보자.

- `subscribeOn()`-`subscribeOn()`은 동작 방식의 두 단계 중 구독 단계에서 쓰레드를 변경한다. 이를 그림으로 표현하면 아래와 같다.

    ![`subscribeOn()` 동작 원리](/reactive-programming/images/how-spring-reactor-works-2.png)

- `publishOn()` - `publishOn()`은 동작 방식의 두 단계 중 이벤트 전파 단계에서 쓰레드를 변경한다. 이를 그림으로 표현하면 아래와 같다.

    ![`publishOn()` 동작 원리](/reactive-programming/images/how-spring-reactor-works-3.png)

### `subscribeOn()` vs. `publishOn()`

아까 위에서 `Mono.map()`과 같은 operator의 로직은 전부 `onNext()` chain에서 실행된다고 했다. 이러한 특성과 앞에서 살펴본 각 메소드의 동작 원리를 합치면, 우리는 `subscribeOn()`과 `publishOn()`을 각각 어떤 상황에서 사용해야 하는지에 대한 원칙을 명확히 세울 수 있다.

1. `subscribeOn()` 은 자신이 구독한 `Publisher`와 operator chain의 모든 동작을 다른 쓰레드에서 처리하고 싶을 때 호출한다.
2. `publishOn()`은 `publishOn()`을 호출한 이후의 동작을 다른 쓰레드에서 처리하고 싶을 때 호출한다.
3. `subscribeOn()`은 operator chain의 어느 단계에서 호출하든 같은 효과를 얻는다.

일반적인 예시를 통해 이 원칙에 대한 이해도를 높여보자.

1. 무거운 작업

    ```kotlin
    function someHeavyCalculation(): Int {
        // some heavy calculation
    }

    Mono.fromCallable { someHeavyCalculation() }
        // a - .subscribeOn(Schedulers.elastic())???
        // b - .publishOn(Schedulers.elastic())???
        .onNext { result ->
            notifyResult(result)
        }
        // c - .subscribeOn(Schedulers.elastic())???
        // d - .publishOn(Schedulers.elastic())???
        .subscribe()
    ```

    a, b, c, d 중 어떤 것을 사용해야 할까?

    어떤 것을 사용할지 판단하기 위해서는 우선 목적이 무엇인지 정해야 한다. 즉, 이 상황에서 어떻게 쓰레드를 관리해야 안전한지를 아는 것이 먼저이다.

    `someHeavyCalculation()`은 수 초 이상 걸리는 아주 무거운 연산 작업이다. 이러한 작업이 웹서버 쓰레드에서 실행되면 문제가 된다. 우리는 이 작업이 웹서버 쓰레드를 점유하지 않고 독자적인 쓰레드에서 진행되도록 하고 싶다.

    이 상황과 목표에 우리가 세운 세 가지 원칙을 적용해보자.

    - 2번 원칙에 따르면 b는 `.onNext {}` 부터 `Schedulers.elastic()` 쓰레드에서 실행시킨다. 즉, `someHeavyCalculation()`은 웹서버 쓰레드에서 실행된다는 뜻이다. 이는 d도 마찬가지이다.
    - 1번 원칙에 따르면 a는 `Mono.fromCallable {}`의 동작을 `Schedulers.elastic()` 쓰레드에서 실행시킨다. 이는 정확히 우리가 원하던 동작이다.
    - 3번 원칙에 따르면 c는 a와 같은 효과를 보인다.

    따라서 우리는 a와 c 중 하나를 선택해서 사용하면 된다. 둘 중 어느 것이 나은지는 취향의 영역이라고 생각한다.

2. 네트워크 통신

    이번에는 조금 더 어려운 예시이다.

    ```kotlin
    function doSomeNetworkCommunication(): Mono<Boolean> {
        // some api call
    }

    doSomeNetworkCommunication()
        // a - .subscribeOn(Schedulers.elastic())???
        // b - .publishOn(Schedulers.elastic())???
        .onNext { result ->
            if (result) {
                saveSuccessToDB()
            } else {
                saveFailureToDB()
            }
        }
        // c - .subscribeOn(Schedulers.elastic())???
        // d - .publishOn(Schedulers.elastic())???
        .subscribe()
    ```

    a, b, c, d 중 어떤 것을 사용해야 할까?

    이번에도 쓰레드 관리의 목적을 먼저 파악해보자.

    일반적으로 네트워크 통신을 처리하는 library는 독자적인 쓰레드 풀을 관리하고 있을 가능성이 높다. 즉, `doSomeNetworkCommunication()`의 리턴 값을 구독해서 네트워크 호출이 일어나는 순간 이 메소드를 호출한 쓰레드에서 네트워크 통신 library가 관리하는 쓰레드 풀의 쓰레드로 전환될 것이다. 만약 a, b, c, d 중 아무것도 하지 않으면 `.onNext {}` 안에서 DB를 다녀오는 무거운 작업이 네트워크 통신 library의 쓰레드를 점유하며 blocking하게 될 것이다. 이는 우리가 원하는 상황이 아니다.

    이제 우리는 이 상황에서의 쓰레드 관리 목표를 분명하게 정의할 수 있다. 바로 DB를 다녀오는 무거운 작업이 네트워크 통신 library의 쓰레드를 점유하지 않게 하는 것이다.

    이 상황과 목표에 우리가 세운 세 가지 원칙을 적용해보자.

    - 1번 원칙에 따르면 a는 네트워크 통신 library 동작을 `Schedulers.elastic()` 쓰레드 풀에서 실행시킨다. 이것이 목적을 달성하는 것이라고 생각하기 쉽지만, 여기에는 한 가지 함정이 있다. 바로 "네트워크 통신 library의 동작"이 무엇이냐에 대한 것이다. `onNext()` chain에서 네트워크 통신 library는 "자신의 쓰레드로 전환하여 네트워크 통신을 진행한다"라는 동작을 수행한다. 즉, a를 사용하면 쓰레드의 흐름이 다음과 같이 된다 :
        1. `Schedulers.elastic()` 쓰레드로 변경하여 네트워크 통신 library의 동작을 실행시킨다.
        2. 네트워크 통신 library가 자신이 관리하는 쓰레드로 변경하여 실제 네트워크 통신을 날린다.
        3. 통신이 완료되면 `.onNext {}`의 동작이 실행된다.

      따라서 `.onNext {}`는 네트워크 통신 library의 쓰레드에서 실행되게 된다.

    - 3번 원칙에 따르면 c는 a와 같은 효과를 보인다.
    - 2번 원칙에 따르면 d는 d 이후에 오는 operator를 `Schedulers.elastic()` 쓰레드 풀에서 실행시킨다. 그 말은 `.onNext {}` 까지는 네트워크 통신 library에서 실행된다는 뜻이다.
    - 2번 원칙에 따르면 b는 `.onNext {}`부터 `Schedulers.elastic()` 쓰레드 풀에서 실행시킨다.

    따라서 이 상황에서는 b를 사용하면 된다.
