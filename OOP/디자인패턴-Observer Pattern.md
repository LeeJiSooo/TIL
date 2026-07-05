## 가져가야 할 핵심

- 옵저버 패턴은 상태를 바꾸는 객체랑, 그 변화에 반응하는 객체를 떼어놓는 패턴이다. 앞에 있는 걸 서브젝트, 뒤에 있는 걸 옵저버라고 부른다. 둘 사이를 이어주는 변경 사실을 담은 Event까지 해서 3가지 요소이다.
- 패턴이 푸는 문제는 상태 변경 코드 안에 후속 처리가 계속 쌓여 비대해 지는 것이다. → 문제상황을 먼저 떠올리고 뭘 해결하는지 이야기할 수 있어야 한다.

## 정의

한 객체의 상태 변화가 다른 여러 객체에게 자동으로 통지되도록, 상태를 가진 주체와 변화를 감지하는 관찰자를 분리해 설계하는 디자인 패턴이다.

→ 관찰자 라는 단어 때문에 옵저버들이 Subject를 계속 주기적으로 쳐다보거나 들여다보는(Polling)그림을 떠올리기 쉽다.

→ 하지만 옵저버 패턴은 완전히 반대이다. 옵저버들은 가만히 대기하고, 상태 변화를 일으킨 Subject가 자신에게 등록된 관찬자들에게 자동으로 소식을 말하는 구조이다.

## 사용이유

상태 변화에 따른 후속 처리를 Subject 내부에 직접 쌓지 않고, 여러 반응 객체로 분리해 확장과 변경 범위를 통제하기 위해서이다.

```java
public class OrderService {
    // 주문 상태 변경 메서드
    public void changeStatus(String orderId, String status) {
        System.out.println("주문 상태 변경: " + orderId + " -> " + status);

        // 치명적인 문제점: 주문 상태 변경이라는 본래 책임 외에, 변경 이유가 완전히 다른 후속 처리들이 얽혀있음
        sendEmail(orderId);      // 이메일 정책 바뀌면 이 메서드 열어야 함
        writeAuditLog(orderId);  // 로그 포맷 바뀌면 이 메서드 열어야 함
        issueCoupon(orderId);    // 쿠폰 로직 바뀌면 이 메서드 열어야 함
        // 만약 여기에 '앱 푸시 알림'까지 추가해달라고 하면 이 메서드 한 줄을 또 수정해야 함
    }
}
```

→ changeStatus 메서드는 주문 상태만 관리하지 않는다. 

→ 이메일, 로그, 쿠폰, 푸시 등 기획 변경 요구사항 때문에 계속 수정되어야 하는 위험한 코드가 된다.

→ 단일 책임 원칙(SRP)과 개방-폐쇄 원칙(OCP)을 위반한다.

- 변하지 않는 것(핵심 흐름) - 주문 상태가 변경되고 상태 갱신이 완료 되었다는 사실 자체
- 자꾸 변하는 것(후속 반응) - 그 상태 변화에 맞춰 무엇을, 얼마나 많이 실행할 것인가에 대한 세부 정책들

→ Subject는 상태가 바뀌었다는 사실을 통지하는 데서 책임을 끝내고, 구체적으로 어떤 일을 할지는 각각 독립된 Observer객체 속으로 완전히 격리시킨다.

## 사용방법

Observer인터페이스를 정의하고, Subject가 Observer 목록을 관리하며, 상태 변경 시 등록된 Observer들에게 이벤트를 통지하도록 구성

→ Subject가 구체 Observer클래스를 직접 알지 않게 만드는 것

→ 서브젝트는 인터페이스만 알고 등록된 옵저버들에게 똑같은 형태로 변경 사실을 전달한다.

#### 상태 변경 주체 식별

어떤 객체의 상태 변화가 후속 처리를 만들어내는지 찾는게 필요하다.

주문 상태가 바뀔때 여러 후속처리가 필요하다는 가정

```java
// 상태 변경 사실을 담아 Observer에게 전달하는 데이터
public class OrderStatusChangedEvent {

    // 후속 처리에 필요한 최소 정보만 보관
    private final String orderId;
    private final String status;

    public OrderStatusChangedEvent(String orderId, String status) {
        this.orderId = orderId;
        this.status = status;
    }

    // 각 Observer가 필요한 값만 꺼내 쓰도록 열어 두는 통로
    public String getOrderId() { return orderId; }
    public String getStatus()  { return status; }
}
```

→ 변화를 표현할 수 있는 객체를 만든다. → Event

→ “주문 상태가 이렇게 바뀌었다”라는 사실을 담아서 옵저버들에게 전달하는 데이터

→ 옵저버에게 주문 엔티티 통째로 넘기는 대신 이렇게 변경 사실만 추려서 별도의 값으로 넘기는 게 좋다.

→ 그러면 후속 처리 객체가 자기에게 필요한 정보만 꺼내 쓰게 되고, 엔티티의 다른 불필요한 부분에 의존하지 않게 된다.

#### Observer인터페이스 정의(공통 계약)

옵저버들이 공통으로 따라야할 계약을 인터페이스로 정의한다.

```java
// 모든 Observer가 공통으로 따르는 계약
public interface OrderStatusObserver {

    // 상태 변경 이벤트를 받았을 때 수행할 동작의 형태를 고정
    void onStatusChanged(OrderStatusChangedEvent event);
}
```

→ 인터페이스가 하는 일은 “주문 상태 변경 이벤트를 받았을 때 무엇을 할지”의 형태를 고정하는 것

→ Subject가 알아야하는 건 이 인터페이스 하나이다.

→ 이메일을 보내는지, 로그를 남기는지, 쿠폰을 발급하는지는 Subject의 관심사가 아님

→ Subject입장에서는 onStatusChange를 부를 수 있는 무언가 이것만 있으면 된다.

#### 구체 Observer구현

실제로 반응할 객체들을 하나씩 구현

```java
// 이메일 발송이라는 단일 반응만 담당하는 Observer
public class EmailNotifier implements OrderStatusObserver {
    @Override
    public void onStatusChanged(OrderStatusChangedEvent event) {
        System.out.println("이메일 발송: " + event.getOrderId());
    }
}

// 감사 로그 기록이라는 단일 반응만 담당하는 Observer
public class AuditLogger implements OrderStatusObserver {
    @Override
    public void onStatusChanged(OrderStatusChangedEvent event) {
        System.out.println("감사 로그 기록: " + event.getOrderId());
    }
}

// 쿠폰 발급이라는 단일 반응만 담당하는 Observer
public class CouponIssuer implements OrderStatusObserver {
    @Override
    public void onStatusChanged(OrderStatusChangedEvent event) {
        System.out.println("쿠폰 발급: " + event.getOrderId());
    }
}
```

세 클래스가 모두 같은 인터페이스를 구현하면서 각자 딱 하나의 반응만 담당한다.

→ 이메일 발송 규칙이 바뀌면 EmailNotifer만 건드리면 된다.

→ 서로 영향을 주지 않는다.

→ 주문 상태 바꾸는 쪽에서 이 규칙을 알 필요도 없다.

#### 구독/해지 구조 구성

Subject는 Ovserver목록을 관리해야한다.

```java
import java.util.ArrayList;
import java.util.List;

// 상태 변경과 통지만 책임지는 Subject
public class OrderStatusSubject {

    // 구체 클래스가 아닌 인터페이스 목록만 보관
    // OrderStatusObserver 등록을 리스트로 들고 있다
    // 인터페이스 목록을 들고 있다.
    // onStatusChanged를 부를 수 있는 것들의 목록
    private final List<OrderStatusObserver> observers = new ArrayList<>();
    private String orderId;
    private String status;

    // Observer를 목록에 등록
    public void addObserver(OrderStatusObserver observer) {
        observers.add(observer);
    }

    // 더 이상 알림이 필요 없는 Observer를 해지
    // 해지는 왜 필요한가요? 알림 받을 필요 없는 옵저버 있으면 제거. 계속 들고 있으면 그 객체가 참조되어서 메모리 잡혀있거나 필요없는 알림 날라가서 제거
    public void removeObserver(OrderStatusObserver observer) {
        observers.remove(observer);
    }

    public void changeStatus(String orderId, String status) {
        // 자신의 상태를 먼저 갱신
        this.orderId = orderId;
        this.status = status;

        System.out.println("주문 상태 변경: " + orderId + " -> " + status);

        // 변경 사실을 이벤트로 포장
        // 어떤 상태로 바뀌었다라는 이벤트를 생성
        OrderStatusChangedEvent event =
                new OrderStatusChangedEvent(orderId, status);

        notifyObservers(event); // 등록된 옵저버들에게 같은 이벤트를 한번씩 전달합니다.
    }

    private void notifyObservers(OrderStatusChangedEvent event) {
        // 순회 중 목록 변경에 따른 예외를 피하기 위한 복사본 순회
        // 목록을 그대로 돌지 않고 new ArrayList로 복사본 만들어서 돌고 있습니다.
        // 만약에 알림을 받은 옵저버가 처리 도중에 자기 자신을 해지하거나 새 옵저버 등록하면 원본 목록을 돌고 있는 도중에 목록이 바뀌게됩니다
        // 자바에서는 순회 중에 컬렉션을 지금 말한것처럼 수정한 ConncurrentModify..Exception 발생할 수 있습니다
        // 복사본을 돌면 순회중에 원본이 바뀌어도 이번에 통지하는 중에는 영향 주지 않아서 안전하게 할 수 있다
        for (OrderStatusObserver observer : new ArrayList<>(observers)) {
            // 구체 Observer를 모른 채 동일한 형태로 통지
            observer.onStatusChanged(event);
        }
    }
}
```

#### 상태 변경 시 알림 전파

```java
OrderStatusSubject subject = new OrderStatusSubject();

// 반응할 Observer들을 Subject에 등록
subject.addObserver(new EmailNotifier());
subject.addObserver(new AuditLogger());
subject.addObserver(new CouponIssuer());

// 상태 변경 한 번으로 등록된 모든 Observer에게 통지
subject.changeStatus("ORDER-1", "PAID"); // OrderId, Status
```

changeStatus를 호출하면 서브젝트는 주문 상태를 바꾸고 등록된 세 옵저버에게 변경 이벤트를 전달한다.

→ 서브젝트는 이메일, 로그, 쿠폰이라는 구체적인 이름을 모른다.

→ 서브젝트가 하는 일은 상태가 바뀌었다라는 말을 하는 것이고 그 말을 들은 옵저버가 자기 할 일을 한다.

```java
// 기존 코드를 고치지 않고 새로 추가하는 후속 처리
public class PushNotifier implements OrderStatusObserver {
    @Override
    public void onStatusChanged(OrderStatusChangedEvent event) {
        System.out.println("앱 푸시 발송: " + event.getOrderId());
    }
}
```

```java
// Subject 수정 없이 등록 한 줄로 끝나는 확장
subject.addObserver(new PushNotifier());
```

→ 새로운 알람이 필요하더라도 그냥 새 옵저버 클래스를 만들고 등록해주면 된다.

→ 서브젝트는 건드리지 않고 할 수 있다.

→ 변경 지점이 상태 변경 메서드 안에서 새 옵저버를 만들고 등록하는 자리로 옮겨갔다.

위에서 짠 고전적인 옵저버 패턴은 자바의 동일한 스레드 메모리 안에서 순서대로 직접 메서드를 호출하는 '동기(Sync) 구조'다. 여기엔 한계가 있다.

1. **예외 전파의 위험:** 만약 1번 이메일 옵저버 로직 안에서 예외가 터져버리면, 뒷순위에 줄 서 있던 쿠폰 발급이나 로그 기록 옵저버들에게는 알림이 전달되지 못하고 전체 주문 프로세스가 멈춰버린다.
2. **성능 지연(Bottleneck):** 어떤 옵저버가 외부 알림톡 API를 호출하느라 3초 동안 지연되면, 정작 주문 상태를 변경하는 메인 트레드가 3초 동안 대기하게 돼서 전체 시스템이 엄청나게 느려진다.
- 이를 막기 위해 후속 옵저버 행위들을 **비동기** 스레드로 분리하거나, 시스템 규모가 커지면 **메시지 큐(RabbitMQ, Kafka)나 Pub/Sub 브러커** 같은 이벤트 기반 아키텍처(EDA)로 확장해서 처리하게된다.

## 스프링부트 환경과 ApplicationEventPublisher

우리가 직접 이 서브젝트를 손으로 만드는 일은 거의 없다.

→ 스프링부트 프레임워크 차원에서 지원을 해준다.

→ ApplicationEventPublisher로 이벤트 발행하고 @EventListner로 이벤트를 받는 방식

```java
@Service
public class OrderService {

    // Observer 목록 관리를 대신 맡아 주는 스프링 발행기
    private final ApplicationEventPublisher publisher;

    public OrderService(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public void changeStatus(String orderId, String status) {
        // 상태 변경 처리 부분 생략

        // 누가 듣는지 모른 채 이벤트만 발행
        publisher.publishEvent(new OrderStatusChangedEvent(orderId, status));
    }
}
```

```java
@Component
public class EmailNotifier {

    // 발행된 이벤트에 반응하는 Observer 역할
    @EventListener
    public void on(OrderStatusChangedEvent event) {
        System.out.println("이메일 발송: " + event.getOrderId());
    }
}
```

OrderService는 이벤트 발행만 하고 누가 그 이벤트를 듣는지 모른다.

EmailNotifier는 @EventListener로 그 이벤트에 반응을 한다.

→ 차이가 있다면 우리가 직접 관리하던 옵저버 목록이랑 통지 책임을 스프링 컨테이너가 대신 가져갔다.

→ 새로운 후속 처리가 필요하면 EventListner가 붙은 컴포넌트 하나 더 추가하고 그 메소드에 @Async 붙이면 비동기로도 돌릴 수 있다.

## 실시간 알림시스템과 연결

- **기본 웹 통신의 제약 (HTTP):** 브라우저가 먼저 말을 걸어야(요청) 서버가 대답하는(응답) 단방향 구조다. 다른 사람이 내 글에 댓글을 달았다는 소식은 '서버'에서 먼저 발생하는데, HTTP는 서버가 브라우저에게 먼저 말을 건네지 못하므로 실시간 알림을 주기 힘들다.
- **과거의 노가다, 폴링(Polling):** 브라우저가 수 초마다 주기적으로 *"저한테 온 알림 있나요?"* 하고 서버에 끈질기게 물어보는 방식이다. 소식이 없어도 무한대로 트래픽 요청을 날려대니 서버 자원의 낭비가 심하다.

이 낭비를 뚫기 위해 서버와 브라우저 사이에 "옵저버 패턴처럼 통로를 미리 열어두고 밀어주는 기술"이 등장하는데, 그게 바로 웹소켓과 SSE다.

### WebSocket (웹소켓)

- 브라우저와 서버 사이에 통로를 한 번 연결해 두고 절대 끊지 않는 **양방향 통신 기술**
- 통로가 뚫려있으니 브라우저도 서버로 실시간 데이터를 쏘고, 서버도 브라우저로 아무 때나 선제적으로 데이터를 밀어 보낼 수 있다.
- 대화가 끊임없이 양방향으로 오가는 **실시간 채팅, 주식 전광판, 구글 Docs 같은 실시간 공동 편집 시스템**.

### SSE (Server-Sent Events)

- 이름 그대로 "서버가 보내는 이벤트"라는 뜻이야. 처음에 브라우저가 서버에 최초 한 번 통로 개설 요청을 보내면, 그 이후부터는 **서버에서 브라우저 방향으로만 소식을 일방통행으로 계속 흘려보내는 단방향 통신 기술이다.**
- 웹소켓보다 세팅이 훨씬 가볍고 HTTP 프로토콜을 그대로 쓰기 때문에 방화벽 문제도 없이 심플하다.
- 클라이언트가 서버로 딱히 보낼 건 없고 오직 소식을 받아만 보면 되는 **실시간 알림 시스템, 실시간 댓글 피드, 트위터 타임라인**

### 옵저버 패턴과의 아키텍처적 도킹

> **"웹소켓과 SSE는 옵저버 패턴의 발상을 실제 네트워크 케이블 위로 실어 나르기 위한 통신 인프라 기술이다."**
> 

서버 내부에서 댓글 작성 같은 사건을 감지한 스프링의 한 옵저버 컴포넌트(`@EventListener`)가 자기 반응 로직으로 **현재 웹소켓 세션이나 SSE 연결 통로에 메시지를 흘려보내면**, 그 데이터가 네트워크를 타고 브라우저 화면까지 도달해 실시간 알림 창이 팝업되는 아키텍처가 완성된다.

## 정리

1. 하나의 목적지 상태가 변경되었을 때, 뒤따라오는 **후속 반응 로직들이 메인 코드에 쌓여 비대해지는 결합도 를 방어**하기 위해 사용한다.
2. 변화의 중심인 **Subject**는 오직 상태 변경과 이벤트 통지만 책임지고, 그 소식을 수신하는 많은 **Observer**들이 각자 맡은 유저 반응(메일, 쿠폰 등)를 처리한다.
3. 새로운 알림 방식이 추가되어도 기존 Subject 코드는 손대지 않고, 새로운 Observer 클래스를 하나 더 만들어 구독자 리스트에 등록만 하면 되는 **OCP(개방-폐쇄 원칙)의 구조이다.**
4. 옵저버의 '통지 및 Push' 메커니즘을 네트워크 브라우저 경계로 확장하여 실시간 통로를 유지하는 기술이 바로 WebSocket(양방향 실시간 채팅)과 SSE(단방향 실시간 알림)이며, 이들의 아키텍처적 뿌리는 모두 옵저버 패턴과 깊게 연결되어 있다.
