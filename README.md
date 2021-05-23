# Basics of Server Development

## 개요

이 레포지토리는 내가 시니어의 직접적인 케어 없이 홀로 개발을 할 수 있게 되는 데에 가장 중요했던 요소를 모아놓은 저장소이다. 굳이 "요소"라는 애매한 워딩을 사용한 이유는 이 문서에 작성된 글이 다양한 분야의 다양한 주제를 다루기 때문이다. 각 문서의 주제는 개발할 때의 원칙/태도일 수도, 개발 전략일 수도, 혹은 특정 지식일 수도 있다. 이는 개발자로서 1인분을 하기 위해서는 특정 분야에 치우치지 않고 다방면으로 학습하는 게 중요하다는 반증이기도 하다.

이 문서는 레이 달리오의 \<Principles\>를 읽은 것이 계기가 되어 작성하게 되었다. 많은 생각을 하게 만들어준 책이었는데, 그 중 하나는 개발을 함에 있어서도 일반적으로 지켜야 하는 원칙이 꽤 많다는 점이었다. 레이 달리오가 인생과 투자의 원칙을 세운 것처럼, 나도 개발의 원칙을 세워 나보다 내가 세운 원칙을 기준으로 개발을 하고, 이런 원칙들을 다른 사람들과 공유하면 좋겠다는 생각이 들었다.

## 목차

아래 목차 순서는 중요도 순이 아니며, 프로젝트의 라이프 사이클을 기반으로 하여 내 임의로 결정한 순서이다.

최대한 특정 기술 스택에 관련되지 않은 내용만을 포함하려고 했으나, 일부 예외가 있다. Reactive Programming이나 모바일 클라이언트 구현 등의 내용이 대표적인데, 이 기술을 사용하지 않더라도 알고 있으면 좋은 지식이라 생각하여 포함하였다.

이미 좋은 글이 있거나 내가 이전에 작성한 글이 있어서 내가 별도로 작성할 필요가 없는 문서는 링크를 대신 걸어두었다.

* 일반론
  * [어떻게 성장할 것인가?](/general/how-to-grow.md)
  * [서버 개발자의 역할은 무엇인가?](/general/roles-of-server-development.md)
  * [기획서와 친해지기](/general/working-with-proposals.md)
  * 유용한 사고방식 및 사고 전략
* 코드와 설계
  * [코드 작성 원칙](/code-and-architecture/principles-of-writing-code.md)
  * [소프트웨어 설계](https://suhwan.dev/2020/04/11/backend-application-design-202004/)
* DB
  * DB index
  * [Lock으로 이해하는 트랜잭션 isolation level](https://suhwan.dev/2019/06/09/transaction-isolation-level-and-lock/)
  * [Foreign key S lock issue](http://www.chriscalender.com/advanced-innodb-deadlock-troubleshooting-what-show-innodb-status-doesnt-tell-you-and-what-diagnostics-you-should-be-looking-at/)
* 쓰레드 관리
  * multi-thread vs. event loop
  * multi-thread 사용 예시 - Spring MVC 기반 어플리케이션의 threading 전략
* Reactive Programming (Spring Reactor 기준)
  * [Project Reactor 동작 원리 & 쓰레드 관리 (`publishOn()` vs. `subscribeOn()`)](/reactive-programming/how-project-reactor-works.md)
  * Reactive Programming에서 일반적으로 맞닥뜨리는 까다로운 상황들
* 분산 시스템 개발
  * [분산 시스템 개발 시 고려해야 할 사항](/distributed-system/distributed-system-concerns.md)
  * [Eventual consistency의 달성](/distributed-system/eventual-consistency.md)
  * 멱등성의 실제 구현 - state machine의 활용
* 다양한 개발 상황과 유의사항
  * [Batch job 작성 시 유의사항](/situations-and-patterns/batch-job.md)
  * 모바일 네이티브 클라이언트 관련 개발 시 유의사항
* 테스팅 
  * TBA
* 배포 전략
  * [상위호환성과 하위호환성](/deployment/forward-and-backward-compatibility.md)
  * 배포할 때 데이터 휘발을 방지하기
* 기타 유용한 개념들
  * [캐릭터 인코딩이란?](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/)
