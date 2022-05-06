## 03. 코드 구성하기

새로운 프로젝트를 들어가면서 패키지 구조를 잡는 것은 중요하다. <br> 
예를 들어 프로젝트 초반에 모두가 합의하여 레이어드 아키텍처를 사용한다 하더라도, import하는 것은 매우 쉽기 떄문에 일정에 따라 얼마든지 뒤섞일 수 있다. <br>

그러면 패키지 구조를 계층/기능/헥사고날로 구성했을 때의 장단점을 따져보자.

<br>

## 1. 계층으로 구성하기


<img width="311" alt="스크린샷 2022-05-07 오전 1 10 11" src="https://user-images.githubusercontent.com/35520314/167171131-c34594da-e37c-4ed4-84f1-de7a079a7683.png">

웹, 도메인, 영속성 계층으로 패키지 구조를 나누어보았다.

1. 3가지 구조만으로는 새로운 기능을 추가하려고 할 때, 연관되지 않는 기능끼리 섞여서 부수효과가 생길 수 있다.  <br>


2. AccountService 안에 어떤 메서드들이 어떤 유스케이스를 구현했는지 파악하기가 어렵다.
- 사실은 저 3개 구조에서 끝날 것이 아니라 application, domain, persistence, web(presentation) 으로 나누고 클래스 명을 더 명확하게 하면 유스케이스를 나타낼 수 있겠지만,
  이 또한 얼마든지 금방 엉망진창이 될 확률이 높았다.


3. 어떤 기능이 어느 어댑터에서 호출하고 어떤 기능을 제공하는지 알아보기가 어렵다.

<br>



## 2. 기능으로 나누면 괜찮아질까?
<br>

계좌 패키지에 관련된 모든 코드를 넣어보자. <br>

그리고 패키지 별로 package-private 접근 수준으로 경계를 강화하면, 각 기능 별로 불필요한 의존성을 방지할 수 있다.
(kotlin 기준 internal로 제한해보았다.)


<img width="313" alt="스크린샷 2022-05-07 오전 1 10 42" src="https://user-images.githubusercontent.com/35520314/167171209-391d3da0-b552-4a95-8b48-91eda5815bbe.png">

그러나 같은 패키지 안에 모든 코드들을 넣어두면, 패키지 수준에서 막는 방지효과를 기대할 수 없다. <br>
같은 패키지 안에 있기 때문에 AccountRepository랑 Impl을 나눠놨어도 Service에서는 Impl에 접근할 수 있기 때문이다. 

새로운 개발자가 와서 Service에 냅다 Impl을 호출해서 사용해도 코드는 잘 돌아간다. 그 코드를 수정해야하기 전까지는,,,

<br>



## 3. 패키지 구조로 아키텍처를 표현해보자!

<br>

- domain : Account, Activity 등 도메인 모델
- application
  - service: domain Service 패키지. 서비스는 유스케이스를 "구현"한다.
  - in 포트 -> 서비스 인터페이스 usecase
  - out 포트 -> 계좌를 가져오거나 상태를 업데이트 하는 등의 인터페이스
- adapter
  - in 어댑터: controller 같은 어댑터가 in 포트를 '호출'한다.
  - out 어댑터: persistence, out 포트를 "구현"한다.

<br>


<img width="446" alt="스크린샷 2022-05-07 오전 1 12 29" src="https://user-images.githubusercontent.com/35520314/167171501-aa64b470-3f9f-405f-b31b-08a2ec0a7a41.png">

이렇게 표현력 있게 나눠놓은 패키지 구조는 다음과 같은 장점이 있다.

- 어느 코드를 어떤 패키지에 넣을지 고민하게 되고, 결론이 명확하다.
- 다른 패키지에서 작업하기 때문에 동시에 작업해도 충돌이 나지 않는다.
- 패키지 간에 접근을 제어할 수 있다.
  - adapter 은 package-private
  - application의 in/out 포트랑 도메인은 public
  - 서비스 같은 경우는 public 일 필요는 없다. 
- 그럼에도 아키텍처에 완전히 핏하지 않은 패키지 구조가 들어올 수 있다 -> 적어도 표현력있는 구조는 모델-코드 갭을 줄여준다.


<br>



## 4. 의존성 주입의 역할


|  클린 아키텍처의 요건은 application 계층은 in,out 어댑터에 의존성을 갖지 않는 것이다.

의존성 역전 원칙을 이용하여, application 계층에 포트 인터페이스를 두고 이를 구현한 어댑터를 만들면 된다.


- adapter/in/web(👀) -> application/port/in <- application/service
  - 웹 어댑터가 in 포트를 호출하고,
  - in 포트를 구현한 서비스가 있다.

<br>

- application/service(👀) -> application/port/out <- adapter/out/persistence
  - 서비스는 out 포트를 호출한다.
  - out 포트를 구현한 영속성 어댑터가 있다.


<img width="647" alt="스크린샷 2022-05-07 오전 1 11 16" src="https://user-images.githubusercontent.com/35520314/167171296-17114cc1-c3be-4b81-ae80-942a206e200f.png">



<br>


---

참고문헌 <br>

https://github.com/thombergs/buckpal <br>
https://kotlinlang.org/docs/visibility-modifiers.html <br>
https://github.com/Meet-Coder-Study/Get-Your-Hands-Dirty-on-Clean-Architecture/blob/main/chapter3/jiaekim.md
