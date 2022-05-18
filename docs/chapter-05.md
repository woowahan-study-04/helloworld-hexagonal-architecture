## Chapter 05. 웹 어댑터 구현하기

외부에서 API 를 호출할 때 필요한 웹 인터페이스를 어댑터로 구현해보자.

 > ⛳️ 유지보수 가능한 소프트웨어를 만드는 데 어떻게 도움이 될까?
웹 어댑터는 HTTP 요청을 → Usecase 메서드 호출로 변환하고 → 수행 결과를 → 다시 HTTP 응답모델로 변환하는 과정을 구현한다.
애플리케이션 계층은 HTTP와 관련된 작업을 하지 않는다. <br> <br> 웹 Controller 모델을 공유하지 않는 여러 클래스로 작게 나누자. 테스트하거나 동시 작업하기 쉽다. <br>
예: OrderController에 주문하기, 주문조회하기 등의 기능들을 넣는 것보다 별도의 PlaceOrderController, findOrderController 로 구분하는 것이 더 낫다.


## 1️⃣ 의존성 역전
웹 어댑터는 애플리케이션을 호출하기 때문에 주도하는 인커밍 어댑터이다.
외부 요청 → 애플리케이션 호출 → “애플리케이션 너는 이렇게 일하면 돼!” 라고 알려준다.


제어 흐름은 웹 어댑터 →  애플리케이션 서비스
애플리케이션 계층은 포트를 제공하여 어댑터와 통신할 수 있다.
웹 어댑터는 포트 인터페이스를 호출 & 서비스는 이 포트를 구현한다.



### 인터페이스가 필요한 이유
컨트롤러가 바로 서비스를 호출할 수 있지만, 간접 계층을 두는 이유는 다음과 같다.

애플리케이션이 외부와 어떤 통신을 하고 있는지 명세하고 있다.
포트를 생략하고 controller에서 sendMoneyService를 바로 호출한다면 [11장138pg]
애플리케이션에 접근하는 진입점을 정의하는 포트이기 때문에
“A 유스케이스를 구현하기 위해 어떤 메서드를 호출해야하는가?” 에 대해 어댑터가 더 알아야한다.
아키텍처를 쉽게 강제할 수 있다.
인커밍 어댑터가 포트만 호출하기 때문에 실수로 서비스를 호출할 일이 없다.
<br>
<br>

### 어댑터는 인과 아웃고잉 역할을 둘 다 할 수 있다.
맨 처음 웹 어댑터는 애플리케이션을 호출하는 입장이기 때문에 “인커밍" 어댑터라고 설명되어있다. <br>
외부에서 애플리케이션을 호출하면, 애플리케이션은 구현하기 때문이다.

하지만 웹 소켓처럼 애플리케이션이 웹 어댑터에게 알림을 줘야하는 경우에는
- 애플리케이션에서 포트를 호출하고
- 웹 어댑터가 이 포트를 구현한다.

그렇게 되면 웹 어댑터는 애플리케이션으로부터 호출당하는 입장이므로 “아웃고잉" 어댑터도 될 수 있다.

---
** 해당 챕터에서는 웹 어댑터는 인커밍 어댑터 역할만 한다고 가정.

## 2️⃣ 웹 어댑터의 책임
REST API 를 제공할 때 웹 어댑터의 책임은 어디까지일까?

1. HTTP 요청을 객체로 매핑한다.
2. 인증, 권한을 검사하고 (회원/비회원)
3. 객체의 입력 유효성을 검증한다.
4. 검증된 입력을 usecase의 입력모델로 매핑한다. (RequestDto → UsecaseModel)
5. 유스케이스를 호출한다.
6. 유스케이스의 출력을 HTTP로 매핑한다. (ResponseDto)
7. return HTTP 응답

<br>

[ 1~3 번 ]
웹 어댑터는 url, 경로, method, content type 과 같은 기준을 만족하는 HTTP 요청을 수신한다. <br>
POST 인데 GET으로 보낸다거나, url을 다르게 보낸다거나, json 으로 요청해야하는데 Stirng으로 보내는 등 제대로 수신된 요청을 받아서 파라미터와 콘텐츠를 객체로 역직렬화한다.


### 웹 어댑터의 입력모델과 유스케이스의 입력모델?
- 웹 어댑터 입력모델 유효성 검증 <br>
웹 어댑터 입력모델 → 유스케이스 입력모델로 변환이 가능한지를 검증한다. (예: orderItems는 not null인데 빈 배열로 요청한 경우 등을 체크한다.)
- 유스케이스 입력모델 유효성 검증 <br>
어플리케이션 수준에서 책임진다.
유효하지 않은 입력 값이 들어오지 않는지 검증한다. (예: 품절된 상품인지, 가격 정책이 잘못된 상품인지 등을 validator로 체크한다.)

[ 4 ~7 번 ]
웹 어댑터의 입력모델을 검증하고 나면 유스케이스 입력모델로 매핑하고, 유스케이스를 호출한다.
웹 어댑터는 다시 유스케이스의 출력을 리턴받아서 HTTP 응답으로 직렬화해서 전달한다.

---

## 3️⃣ 컨트롤러 나누기
단일 OrderController에서 모든 연산을 넣기보다 작은 단위의 클래스로 나누는 것이 좋다.

- 클래스 안의 코드가 많을수록 변경사항을 찾기 어렵다. 
- 컨트롤러에 코드가 많아질수록 테스트 코드도 많아지고, 특정 프로덕션 코드에 해당하는 테스트코드를 찾아서 파악하기가 어려워진다.
여러 연산이 하나의 큰 모델 클래스를 공유하면서 데이터 구조의 재활용을 촉진한다.
  - 모델 클래스에 들어있는 id 필드는 create에는 필요없지만 get에서는 필요하다.
  - 모델 클래스의 어느 필드를 어떤 연산에서 써야할지 헷갈리게 된다.
  - 모델 클래스에 새로운 필드를 하나 추가하고싶은데, 여러 연산에서 재활용하고 있다면 쉽게 추가하기 어려워진다.

<br>
👉 별도의 패키지 안에 별도의 컨트롤러를 만들고, 클래스명은 유스케이스를 최대한 반영해서 짓자.
👉 컨트롤러마다 자체적인 모델 클래스를 사용하자. 


e