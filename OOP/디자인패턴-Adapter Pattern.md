## 가져가야 할 핵심

- 어댑터 패턴은 새 기능을 만드는 패턴이 아니라, 이미 있는 것을 연결하는 패턴이다. - 우리가 고칠 수 없는 외부 라이브러리나 레거시 객체를 손대지 않고 우리 시스템이 끼워쓰는 것이다.
- 세가지 요소로 나뉘어져있다. 우리가 기대하는 계약인 Target, 이미 존재하지만 그 계약과 맞지 않는 adptee, 그 사이에서 호출형태를 바꿔주는 adapter
- 지금 패턴의 효과는 변경 지점을 한 곳으로 모으는데 있다. 외부 규격이 바뀌었을 때 고쳐야 할 곳이 여기저기 흩어진게 아니라 어댑터 하나로 좁힐 수 있다.

## 정의

호환되지 않는 인터페이스를 가진 객체들이 함께 동작할 수 있도록, 중간 대역 객체가 호출 형태와 데이터 포맷을 변환해 주는 객체지향 구조 패턴이다.

- Target(우리 시스템의 계약) - 클라이언트(우리 비즈니스 서비스)가 기대하고 사용하는 표준 인터페이스이다. 외부가 어떻게 돌든 상관없이 우리 시스템 안의 도메인 언어로 정의된 기준점이다.
- Adaptee(외부/레거시 객체) - 이미 작동하고는 있지만, 인터페이스 규격이 우리 시스템과 맞지 않는 기존 객체나 외부 라이브러리.(우리가 소스코드를 직접 수정할 수 없는 대상인 경우가 많음.)
- Adapter(중간 변환 객체) - Target 인터페이스를 직접 구현하면서, 내부적으로는 Adaptee를 가지고 있으면서 호출 형식을 위임해 준다.

## 사용 이유

기존 코드, 외부 라이브러리, 레거시 API를 직접 수정하지 않고도 현재 시스템이 기대하는 인터페이스에 맞춰 사용하기 위해서

```java
// 우리 시스템이 결제 객체에게 기대하는 호출 형태
public interface PaymentClient {
    void pay(int amount); // 정수 금액을 받아 결제를 수행하는 단순한 계약
}
```

→ 우리가 가진 주문서비스가 결제 기능을 이런 형태로 쓰고 싶다고 가정

→ 우리 입장에서는 pay에 금액만 넘기면 결제가 되는 구조가 가장 단순

→ 실제로 연동해야하는 외부 PG 라이브러리는 아래와 같은 모양

```java
// 실제 결제를 수행하지만 우리 계약과 형태가 다른 외부 PG 객체
public class LegacyPgClient {
    // 메서드 이름은 requestPayment, 금액은 문자열로 받는 구조
    public void requestPayment(String priceText) {
        System.out.println("외부 PG 결제 요청: " + priceText);
    }
}
```

```java
// 외부 PG 객체에 직접 의존하는 적용 전 구조
public class PaymentService {

    private final LegacyPgClient legacyPgClient; // 외부 라이브러리 타입에 그대로 묶임

    public PaymentService(LegacyPgClient legacyPgClient) {
        this.legacyPgClient = legacyPgClient;
    }

    public void payOrder(int amount) {
        // 문자열 변환과 메서드 이름까지 서비스가 직접 알고 있음
        legacyPgClient.requestPayment(amount + "원");
    }
}
```

→ 주문 처리를 책임져야 할 핵심 비즈니스 서비스 코드가 외부 PG사 라이브러리의 세부 스펙(`requestPayment`라는 이름, '원'을 붙여야 하는 문자열 포맷 규칙 등)을 다 알고 있게된다.

→ 이 상태에서 나중에 다른 결제사로 바꾸거나 외부 라이브러리 버전이 아주 조금만 업그레이드되어도, 우리 서비스의 소스 코드를 다 찾아서 뜯어고쳐야 하는 리스크를 가지게 된다. 

→ 외부 객체의 변경 이유가 우리 비즈니스 코드의 변경 이유가 되어버린다.

## 사용 방법

#### 호출되지 않는 인터페이스 상황

제일 먼저 인터페이스가 맞지 않는 상황을 분명하게 봐야 한다.

→ 우리 시스템은 정수금액을 받는 pay를 기대하고 외부 PG 객체는 문자열 금액을 받는 requestPaymentfmf 제공한다.

→ 목적은 둘 다 결제요청으로 같지만 호출하는 형태가 다르다 라는걸 짚을 수 있다.

→ 여기서 중요한 차이를 클라이언트 코드에 그대로 노출하면 클라이언트가 외부객체의 규격에 끌려간다.

→ 이 차이를 클라이언트 바깥에 흡수

#### Target인터페이스 정의

클라이언트가 기대하는 인터페이스를 고정하는 것

```java
// 클라이언트가 의존할 Target 인터페이스를 먼저 고정
public interface PaymentClient {
    void pay(int amount); // 외부 구현과 무관하게 우리 시스템 안쪽의 언어로 정의
}
```

우리 시스템이 결제 객체에게 기대하는 계약

→ 우리 시스템 안쪽의 언어를 먼저 정한다.

→ 외부 라이브러리 모양에 비즈니스 코드를 맞추는 게 아니라 비즈니스 코드가 필요로 하는 역할을 인터페이스로 먼저 표현한다.

#### 기존객체(Adaptee)확인

우리가 가져다 쓸 소스코드 수정 불가능한 외부 PG클래스 스펙을 확인해두는 단계이다.

```java
// 우리가 빌려 쓰는 외부 SDK 라이브러리 (수정 불가능)
public class LegacyPgClient {
    public void requestPayment(String priceText) {
        System.out.println("외부 PG사 모듈 작동: " + priceText);
    }
}
```

#### Adapter구현

우리 시스템 표준인 PaymentClient를 구현하면서, 내부에는 외부 객체(Adaptee)를 장착한다. 그리고 오직 “규격 번역 및 위임 책임”만 전담하게 만든다.

```java
public class LegacyPgAdapter implements PaymentClient {

    private final LegacyPgClient legacyPgClient; // 실제 일할 외부 객체(Adaptee) 보관

    public LegacyPgAdapter(LegacyPgClient legacyPgClient) {
        this.legacyPgClient = legacyPgClient;
    }

    @Override
    public void pay(int amount) {
        // 핵심: 정수형 금액을 외부에 맞게 문자열 규격("10000원")으로 가공 변환함
        String priceText = amount + "원"; 
        
        // 우리 표준 호출(pay)을 외부 규격 메서드(requestPayment)로 안전하게 토스
        legacyPgClient.requestPayment(priceText); 
    }
}
```

#### 클라이언트 호출 구조

```java
// Target 인터페이스에만 의존하는 적용 후 구조
public class PaymentService {

    private final PaymentClient paymentClient; // 어떤 구현이 들어오는지 알 필요 없음

    public PaymentService(PaymentClient paymentClient) {
        this.paymentClient = paymentClient;
    }

    public void payOrder(int amount) {
        paymentClient.pay(amount); // 결제 요청이 가능한 객체에게 그대로 위임
    }
}
```

→ requestPayment 직접 호출하던 페이먼트 서비스랑 비교하면 paymentClient에 의존한다.

→ 결제가 실제로 legacyPgClient를 통해 처리되는지 다른 PG사를 통해 처리되는지, 테스트용 가짜 객체를 통해서 되는지 알 필요가 없다.

→ 클라이어트가 알고 있는건 결제 요청을 할 수 있는 객체가 있다.

```java
LegacyPgClient legacyPgClient = new LegacyPgClient(); // 실제 결제를 수행할 Adaptee 준비
PaymentClient paymentClient = new LegacyPgAdapter(legacyPgClient); // Adaptee를 Adapter로 감싸 Target 타입으로 사용

PaymentService paymentService = new PaymentService(paymentClient); // 서비스에는 Target 계약만 주입
paymentService.payOrder(10000); // 외부 규격을 모른 채 결제 요청
```

→ 이 구조에서 외부 PG라이브러리 호출 규격은 LegacyPgAdapter 안에만 존재한다.

→ 외부 라이브러리 메서드 이름이나 파라미터 구조가 바뀌면 어댑터만 수정하면 된다.

→ 서비스 코드는 PaymentClient 계약만 바라보기 때문에 변경 범위가 좁아진다.

## 포트 앤 어댑터 아키텍처

- 발생하는 문제 : 시스템이 커지면 중앙의 순수 비즈니스 로직(주문, 정산 등)과 바깥세상의 기술(MySQL 문법, 특정 결제사 SDK, AWS S3 등)이 한데 뒤엉킨다. DB나 결제사를 바꾸려고 할 때 서비스 핵심 로직까지 영향을 미친다.

→ 헥사고날 아키텍처는 이 엉킴을 시스템 최외곽 경계선에서 끊어버린다.

1. 중심부의 핵심 비즈니스 영역은 바깥 세상 기술을 직접 알지 못하게 완전히 격리한다.
2. 대신 핵심 코어가 필요로 하는 소통 인터페이스 구멍을 선언하는데, 이 경계 인터페이스를 포트라고 부른다.
3. 그리고 그 포트 구멍에 실제 외부 기술(Toss, JPA, AWS 등)을 연동하여 꽂아주는 대역 클래스들을 어댑터라고 부르는 구조이다.

```java
// 애플리케이션 코어가 기대하는 경계 인터페이스
public interface PaymentPort {
    void pay(int amount); // 외부 결제 기술과 분리된 핵심 로직의 계약
}
```

```java
// Port에 실제 외부 결제사를 연결하는 어댑터
public class TossPaymentAdapter implements PaymentPort {

    private final TossPaymentClient tossPaymentClient; // 특정 결제사 SDK를 안쪽에 감춤

    public TossPaymentAdapter(TossPaymentClient tossPaymentClient) {
        this.tossPaymentClient = tossPaymentClient;
    }

    @Override
    public void pay(int amount) {
        tossPaymentClient.request(amount); // 코어의 pay 요청을 Toss 호출로 변환
    }
}
```

```java
// 같은 Port에 테스트용 구현을 연결하는 어댑터
public class FakePaymentAdapter implements PaymentPort {

    @Override
    public void pay(int amount) {
        // 실제 결제사 없이 코어 로직만 검증하기 위한 구현
        System.out.println("테스트 결제 처리: " + amount);
    }
}
```

**어댑터 패턴과의 층위 차이:** 

**어댑터 패턴:** 클래스/객체 하나 수준에서 인터페이스 불일치를 푸는 미시적인 **디자인 패턴**

**포트 앤 어댑터:** 어플리케이션 아키텍처 전체 영역에서 중심부 로직과 외부 인프라 기술을 어떻게 격리하고 스타일링할지 결정하는 거시적인 **아키텍처 패턴**.

## 정리

- 소스코드 수정이 불가능한 외부 라이브러리나 낡은 레거시 객체의 규격이 우리 시스템의 표준 규격과 맞지 않을 때, 중간에서 인터페이스 불일치를 해결해 연동하는 패턴이다.
- 외부 규격이나 API 스펙이 바뀌더라도 비즈니스 로직은 건드리지 않고, 오직 어댑터 클래스 내부 한 곳만 고치는 형태로 변경 지점을 완벽하게 제한한다.
- 개발할 때 외부 라이브러리 스펙에 휘둘려 내 비즈니스 코드를 맞추지 말고, 우리 시스템 안의 언어로 표준 인터페이스(Target)를 정의한다.
- 객체 수주의 어댑터 번역 아이디어를 소프트웨어 경계 전체로 확장하여 바깥 기술과 코어를 완전히 격리한 구조가 핵사고날 아키텍처이다.
