# Portable, Efficient, and Practical Library-Level  Choreographic Programming

**Portable, Efficient, and Practical Library-Level  Choreographic Programming** (2023)

SHUN KASHIWA, GAN SHEN, SOROUSH ZARE, LINDSEY KUPER

- https://arxiv.org/pdf/2311.11472

<br>
<br>

---

**안무 프로그래밍(Choreographic Programming, CP)** 은 <br>
여러 노드에서 실행되는 분산 애플리케이션을 프로그래밍하기 위한 새로운 패러다임이다. 

CP에서는, 각 노드에 대해 개별 프로그램을 구현하는 대신에, 프로그래머가 하나의 통합된 프로그램인 안무(choreography) 를 작성하고, <br>
그 후 엔드포인트 프로젝션(endpoint projection, EPP) 이라는 컴파일 단계를 통해 각 노드에 대한 개별 프로그램으로 변환된다.

CP 언어는 10년 넘게 존재해왔지만, 라이브러리 수준 CP <br>
— 즉 안무를 기존 호스트 언어에서 프로그램으로 표현하고, 안무 언어 구성 요소들과 엔드포인트 프로젝션을 전적으로 호스트 언어의 라이브러리를 통해 제공하는 방식 —
는 아직 초기 단계에 있다. 

라이브러리 수준 CP는 큰 잠재력을 가지고 있지만, 기존에 존재하는 구현 접근 방식들은 <br>
이식성, 효율성, 실용성 면에서 단점들을 가지고 있어 채택을 방해하고 있다.

이 논문에서 우리는 라이브러리 수준 CP의 기술 수준을 <br>
두 가지 새로운 안무 라이브러리 설계 및 구현 기법을 통해 발전시키는 것을 목표로 한다: <br>
의존성 주입으로서의 엔드포인트 프로젝션(EPP-as-DI) 과 안무 엔클레이브(choreographic enclaves). 

EPP-as-DI는 라이브러리 수준에서 EPP를 구현하기 위한 언어 독립적인 기법이다. <br>
기존의 라이브러리 수준 접근들과는 달리, EPP-as-DI는 호스트 언어로부터 많은 것을 요구하지 않으며 <br>
— 고차 함수에 대한 지원만 있으면 된다 — 매우 다양한 호스트 언어에서 사용할 수 있다.

안무 엔클레이브는 프로그래머가 더 큰 안무 안에 하위 안무(sub-choreography)를 정의할 수 있도록 해주는 언어 기능이다. <br>
엔클레이브 안에서는 “선택에 대한 지식”이 오직 그 엔클레이브에 참여하는 참여자들 사이에서만 전파되며, <br>
이는 기존의 라이브러리 수준 안무 조건문의 구현에서 발생하는 효율성 제한을 해결하면서 호스트 언어의 조건문 구문을 원활히 사용할 수 있게 해준다.

우리는 Rust 프로그래밍 언어를 위한 최초의 CP 라이브러리인 ChoRus 안에 EPP-as-DI와 안무 엔클레이브를 구현하였다. <br>
우리의 사례 연구와 벤치마크는 ChoRus의 사용성과 성능이 전통적인, 안무가 아닌 방식의 Rust 분산 프로그래밍과 비교했을 때도 긍정적인 결과를 보임을 보여준다.


---
## 1. Introduction

분산 시스템에서는, 독립적인 노드들의 집합이 서로 메시지를 보내고 받음으로써 통신한다. 

프로그래머는 각 노드의 로컬 행동 — 메시지를 보내고 받는 일과 내부 동작을 수행하는 일 —  <br>
이 전체 시스템의 원하는 전역 행동(global behavior)을 이루도록 보장해야 한다. 

매우 간단한 예로, Alice와 Bob이라는 노드가 참여하는 분산 프로토콜을 생각해보자.  <br>
이 프로토콜에서 Alice는 Bob에게 인사를 보내고, Bob은 응답한다. 

전통적인 분산 프로그래밍에서는 (메시지 전송을 구현하는 send와 receive 함수가 존재한다고 가정할 때)  <br>
Alice는 로컬 노드 프로그램으로 send("Hello!", Bob); receive(Bob); 을 실행할 수 있다.  <br>
동시에 Bob은 자신만의 로컬 노드 프로그램 receive(Alice); send("Hi!", Alice); 를 실행할 것이다. 

Alice와 Bob은 서로가 프로토콜을 충실히 따를 것에 의존한다.  <br>
예를 들어, 둘 중 하나라도 send 호출을 잊는다면, 상대편은 메시지를 받기 위해 영원히 기다리게 되며  <br>
(혹은 타임아웃이 발생하고 오류를 보고할 수도 있다).
이러한 접근은 교착 상태(deadlock)를 포함한 버그에 취약하다.

최근 등장한 안무 프로그래밍(choreographic programming, CP) 패러다임은 이러한 종류의 버그를 제거할 수 있는 방법을 제공한다 <br>
[Carbone and Montesi 2013; Montesi 2013; Cruz-Filipe and Montesi 2020; Giallorenzo et al. 2020; Hirsch and Garg 2022; Shen et al. 2023; Montesi 2023]. 

개별 노드를 프로그래밍하는 대신, 안무 프로그래머는 안무(choreography) 라 불리는 하나의 프로그램을 작성하는데,  <br>
이는 전체 시스템의 동작을 객관적이고 제3자의 관점에서 표현한다. 

예를 들어 위의 프로토콜은 Alice("Hello!") ~> Bob; Bob("Hi!") ~> Alice; 와 같은 안무로 표현될 수 있다.  <br>
여기서 ~> 연산자는 송신자와 수신자 간의 통신을 나타낸다.

안무는 엔드포인트 프로젝션(endpoint projection, EPP) 이라는 컴파일 단계에 의해 개별 노드의 로컬 프로그램들로 변환된다  <br>
[Qiu et al. 2007; Carbone et al. 2007, 2012]. 

EPP가 올바르게 작동한다면, 결과로 생성된 모든 로컬 프로그램에서의 send 연산은  <br>
다른 로컬 프로그램에 상응하는 receive 연산을 반드시 가지게 되며,
이는 교착 상태의 부재(deadlock freedom)를 보장한다  <br>[Carbone and Montesi 2013].

지난 10년 동안, 여러 CP 언어들이 제안되었다  <br>[Carbone and Montesi 2013; Montesi 2013; Dalla Preda et al. 2014, 2017; Giallorenzo et al. 2020; Hirsch and Garg 2022].  <br>
그러나 라이브러리 수준 CP — 즉 안무가 기존의 호스트 언어 안의 프로그램으로 표현되고,  <br>
안무 연산자들과 EPP가 전적으로 호스트 언어의 라이브러리에 의해 제공되는 방식 — 는 이제 막 등장하기 시작한 단계이다.

라이브러리 수준 CP는 프로그래머들이 이미 사용하고 있는 언어와 그 생태계를 그대로 활용할 수 있도록 함으로써,  <br>
CP의 접근성과 실용성을 크게 향상시킬 수 있는 잠재력을 지닌다. 

특정 호스트 언어에서 안무 프로그래밍을 라이브러리 수준으로 구현하면,  <br>
이는 일반적인 내장 DSL(도메인 특화 언어)의 장점을 그대로 누릴 수 있다 [Hudak 1996].  <br>
즉, 그것은 일반적인 라이브러리처럼 설치 가능하고, 일반적인 프로그램처럼 컴파일 가능하며,  <br>
개발, 디버깅, 배포를 위한 호스트 언어의 도구들을 그대로 사용할 수 있다.

또한, 라이브러리 수준 CP는 안무 구성 요소들을 더 큰 시스템에 통합하는 데 도움을 줄 수 있으며,  <br>
프로그래머가 특정 구성 요소를 안무적으로 구현하기 위해 언어를 변경할 필요가 없도록 해준다. 

예를 들어, 하나의 머신에서 플레이어들이 번갈아 가며 게임을 진행하는 턴제 게임을,  <br>
네트워크를 통해 상호작용하는 분산 구현으로 옮기려는 경우에 특히 적합하다.  <br>
CP를 사용하면 처음부터 끝까지 하나의 프로그램으로 작업할 수 있으며,  <br>
라이브러리 수준 CP는 이 과정에서 언어를 바꿀 필요조차 없게 해준다.

이러한 장점을 고려할 때, 우리는 라이브러리 수준 CP를 어떻게 구현할 수 있을까?  <br>
지금까지 존재하는 유일한 라이브러리 수준 CP 구현은 최근 제안된 HasChor 프레임워크이다 [Shen et al. 2023].  <br>
이 프레임워크는 Haskell에 내장된 도메인 특화 언어를 통해 CP를 지원한다.  <br>
HasChor에서는 안무가 모나딕(monadic) 계산으로 표현되며, ~> 같은 안무 연산자들이 사용될 수 있다.  <br>
내부적으로, HasChor는 freer monad의 동적 해석에 기반한 정교한 구현 기법을 사용하여 EPP를 수행한다 [Kiselyov and Ishii 2015].

HasChor 프레임워크는 라이브러리 수준 CP의 현재 기술 수준을 대표하지만,  <br>
그 구현 방식은 이식성, 효율성, 실용성 측면에서 여러 제한점을 가지고 있다. 

첫째, HasChor의 구현은 Haskell 고유의 언어 기능에 의존하며, Haskell의 특정 추상화 도구가 없는 다른 언어들로 쉽게 이식되기 어렵다.  <br>
특히, EPP를 구현하는 데 사용된 freer monad는 다른 언어로의 포팅을 어렵게 만든다. 

둘째, HasChor의 EPP 구현 — 이는 현재 존재하는 유일한 라이브러리 수준 EPP 구현이다 — 은  <br>
독립적인 CP 언어들이 제공할 수 있는 것에 비해 비효율적인 런타임 동작을 야기한다.  <br>
특히, 안무에서의 조건문 처리는 불필요한 네트워크 트래픽을 초래하여, 네트워크 대역폭이 제한된 일반적인 분산 환경에서는 적합하지 않다.  <br>
또한, HasChor는 프로그래머가 조건문을 작성할 때 호스트 언어의 조건문이 아닌 HasChor 고유의 언어 구조를 사용해야만 한다.

마지막으로, HasChor는 안무 구성 요소를 더 크고 비안무적인 소프트웨어 시스템에 통합하는 데 유용한 기능들을 결여하고 있다.  <br>
이는 불행한 일인데, 왜냐하면 안무와 비안무 코드를 매끄럽게 통합할 수 있다는 점은 라이브러리 수준 CP의 장점이어야 하기 때문이다  <br>
— 독립적인 CP 언어들과는 대조적으로 말이다.

이 논문에서 우리는 위와 같은 제한점을 해결함으로써 라이브러리 수준 CP의 기술 수준을 발전시키고자 한다.  <br>
우리는 다음과 같은 구체적인 기여를 한다: <br>

우리는 의존성 주입으로서의 엔드포인트 프로젝션(EPP-as-DI) 을 제안한다.  <br>
이는 라이브러리 수준 CP를 위한 새롭고 언어에 독립적인 구현 기법이다 (3장).  <br>
HasChor의 접근 방식과 달리, EPP-as-DI는 호스트 언어로부터 거의 아무 것도 요구하지 않으며, 고차 함수 지원만 있으면 충분하다.  <br>
따라서, EPP-as-DI는 다양한 호스트 언어에서 간단히 사용할 수 있다.

우리는 라이브러리 수준 CP에서 효율적인 조건문을 구현하기 위한  <br>
새로운 설계 및 구현 기법인 안무 엔클레이브(choreographic enclaves) 를 제안한다 (4장).  <br>
엔클레이브를 사용하면, 프로그래머는 단순한 안무 조건문의 구현이 초래하는 대역폭 비효율성을 피하면서도  <br>
호스트 언어의 조건문 구문을 자연스럽게 사용할 수 있다. <br>

우리는 제안된 기법들을 사용하여 구현한 Rust용 안무 프로그래밍 라이브러리인 ChoRus 를 제시한다 (5장).  <br>
ChoRus는 EPP-as-DI와 안무 엔클레이브 외에도, 위치 지정 인자(located arguments)와 반환값을 지원하는 최초의 CP 라이브러리이다 (5.2.2절).
이 기능은 안무 구성 요소를 더 크고 비안무적인 소프트웨어 프로젝트에 통합하는 데 도움을 준다. 

우리는 Rust의 전통적인 분산 프로그래밍과 비교하여 ChoRus의 사용성과 성능을 실증적으로 평가한다 (6장).

ChoRus의 구현체, 사례 연구 및 벤치마킹 코드, 문서는 https://github.com/lsd-ucsc/ChoRus 에서 이용할 수 있다.

<br>

    안무 프로그래밍은 EPP 과정을 통해 각 노드의 로컬 프로그램으로 자동 변환되며, 
    모든 송신에는 반드시 대응되는 수신이 존재하도록 하여 교착 상태를 방지한다.

    [라이브러리 수준의 CP의 가능성과 한계]
    - 기존 CP 언어들은 새로운 언어를 배우고 사용해야 했지만, 
    라이브러리 수준의 CP는 기존 언어 내부에서 CP를 구현함으로써 접근성과 실용성을 크게 높인다. 
    
    - 현재까지 유일한 라이브러리 수준의 CP 구현은 HasChor 프레임워크
    - 휴대성 부족: Haskell에 특화된 기능에 의존하여 다른 언어로 이식하기 어렵다.
    - 비효율성: 조건문 처리 시 불필요한 네트워크 통신이 발생하여 일반적인 분산 환경에서 부적합하다.
    - 제한된 실용성: 조건문에 Haskell 전용 구문을 사용해야 하며, 기존 시스템과 통합이 어렵다.

    [본 논문의 기여]
    - EPP-as-DI (의존성 주입으로서의 EPP): 고차 함수만 지원하면 어떤 언어에서도 구현 가능한 범용적 기법.
    - 안무 엔클레이브: 호스트 언어의 조건문을 그대로 활용하면서, 효율적인 조건문 처리를 가능하게 함.
    - Rust 기반 CP 라이브러리인 ChoRus의 구현: 이 라이브러리는 EPP-as-DI와 엔클레이브를 포함하며, 
                                              위치 정보를 가진 인자/반환값을 지원.
    

