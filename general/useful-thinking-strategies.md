# 개발 시 유용한 개념과 사고 전략

## 하향식 접근법

어딘가 먼 장소로 이동하는 상황을 생각해보자. 예를 들어, 서울에 있는 집에서 제주도에 있는 호텔까지 이동한다고 해보자. 이 경우 보통 큼지막한 규모로 계획을 세울 것이다. 예를 들어 "김포공항에서 제주공항까지 비행기를 타고 가고, 집 → 김포공항과 제주공항 → 호텔은 택시를 타고 간다" 정도. 그리고 실제로 이동을 할 때는 각 계획의 단계를 더 작은 행동들로 분해해서 수행한다.

- "택시를 타고 간다" - 택시 호출 → 택시가 도착하면 탑승 → 택시를 타고 김포공항까지 이동 → 요금 결제 → 택시 하차 → 김포공항 내부까지 걸어서 이동
- "비행기를 타고 간다" - 탑승 수속 → 보안 검색대 통과 → 탑승구까지 이동 → 탑승 후 좌석 찾기 → 착석 → 비행기를 타고 제주공항까지 이동 → 내리기

개발도 마찬가지이다. 특정한 동작을 구현하려면 그 동작을 컴파일러가 이해할 수 있는 수준인 변수 할당, 연산, 조건문, 반복문의 수준까지 분해해야만 한다. 일반적으로 우리가 원하는 동작은 코드 수준과는 상당히 먼 추상적인 동작이다. 때문에 한 번에 코드 레벨까지 분해하기에는 쉽지 않다. 이를 하려면 원하는 동작을 조금 더 상세한 세부 동작의 합으로 표현하는 작업을 반복적으로 수행해야만 한다.

이렇게, 최상위의 추상적인 요구사항을 구체적인 동작으로 분해하여 시스템을 설계하는 것을 하향식 접근법이라고 한다. 하향식 접근법은 원하는 방식으로 동작하는 코드를 작성하는 가장 기본적인 전략이다. 이것을 제대로 할 수 없으면 코딩 속도가 현저히 느려지고, 요구사항에 맞는 올바른 코드를 작성하기가 어렵다.

한 가지 주의해야 할 것은, 하향식 접근법은 대규모 설계에는 적합하지 않은 방식이라는 점이다. 대규모 설계에는 시스템의 컴포넌트 사이의 협력과 컴포넌트의 책임을 더 중요시하는 객체지향식 접근법이 더 적합하고, 하향식 접근법은 각 컴포넌트의 동작을 실제로 구현할 때 더 적합하다.

아직 원하는 대로 동작하는 코드를 작성하는 데 어려움을 느낀다면 협력과 책임을 고민하기보다는 하향식 접근법에 익숙해지는 게 우선이다. 설계는 올바른 동작 다음이다.

## MECE하게 상태를 나누기

MECE는 Mutually Exclusive, Collectively Exhaustive의 약자이다. Mutually Exclusive는 서로 겹치지 않는 것들의 모임이라는 뜻이고, Collectively Exhaustive는 모두 모았을 때 전체가 된다는 뜻이다. 즉 MECE하게 상태를 나눈다는 건 전체를 겹치지 않은 부분들로 나눈다는 뜻이다.

![MECE](/general/images/mece.png)

MECE는 컨설팅 업체인 맥킨지가 가장 먼저 도입한 개념이라고 하는데, 개발을 할 때도 여러 상황에서 상당히 유용하게 사용할 수 있는 개념이다.

- 기획에서 누락된 시나리오를 챙기기에 적합하다. 기획과 화면이 받아주지 않고 있는 코너 케이스, 특정 상태에서 어떻게 동작할지에 대한 정의가 누락된 부분 등을 발견하기 좋다.
- 3가지 이상의 시나리오가 존재하는 복잡한 기획이 있는 경우, MECE하게 분리해서 조건문을 작성하면 의도치 않은 동작을 방지할 수 있다.

무언가 놓치고 있는 듯한 불안한 느낌이 들면 상태를 MECE하게 나눠보도록 하자. 불안함의 원인을 찾을 수 있을지도 모른다.

## State Machine

State machine, 혹은 finite state machine은 컴퓨터 과학 쪽에서 나오는 개념으로, 연산을 모델링하는 방법 중 하나이다. state machine은 연산을 다음과 같은 4가지 요소로 모델링한다.

- 시스템이 지닐 수 있는 상태의 목록
- 시스템이 가동됐을 때의 초기 상태
- 상태의 전이가 발생하는 규칙
- 전이됐을 때 시스템이 종료되는 상태

State machine은 복잡한 life cycle을 가진 도메인을 설계할 때 유용하다. 필자가 특히 유용하게 사용했던 영역은 다음과 같다.

- 상태를 가진 엔티티의 행동 정의 - 긴 life cycle을 가지고 상태 변화를 추적해야 하는 엔티티를 설계할 때 큰 도움이 된다. 어떤 상태에서 어떤 행동이 가능해야 하는지, 가능한 상태 정의는 무엇인지를 파악하면 역으로 각 상태에서 하면 안 되는 행동과 상태 전이를 알아낼 수 있다. 이는 엔티티가 잘못된 상태로 빠질 위험을 감소시켜, 보다 견고하고 안전한 도메인 설계를 이끌어낼 수 있도록 도와준다.
- 분산 시스템 설계 - 분산 시스템에서는 올바른 재시도 로직을 구현하는 것이 중요하다. 올바른 재시도 로직에는 재시도를 하는 조건, 재시도를 중지해야 하는 조건 등이 포함되는데, 원격 프로세스의 진행 상황을 처리 중 / 재시도 필요 / 재시도 불가 / 처리 완료의 4가지 상태로 나누어서 추적하면 깔끔하고 명확한 재시도 로직을 구현할 수 있다.

## Refs

- [https://en.wikipedia.org/wiki/Finite-state_machine](https://en.wikipedia.org/wiki/Finite-state_machine)