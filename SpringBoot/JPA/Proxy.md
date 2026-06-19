## 가져가야 할 핵심

1. 프록시의 물리적 구조 : 원본 엔티티 클래스를 상속받아 동적으로 만들어지며, 식별자와 진짜 엔티티를 가리킬 참조 필드만 가지고 있다.
2. 초기화 과정의 본질 : 프록시 초기화는 가짜가 진자로 변하는 게 아니라, 내부에 진자 엔티티의 주소값을 연결하고 메서드를 위임하는 과정이다.
3. 프록시 때문에 터지는 타입 비교 문제(instanceof)와 준영속 상태 예외(LazyInitializationException)의 원인과 해결책을 알아야 한다.

## 프록시(Proxy)

JPA에서 지연 로딩을 사용할 때, 실제 엔티티 객체 대신 먼저 생성되어 진짜 데이터가 필요한 순간까지 물리적인 DB조회를 미루는 가짜 껍데기 객체이다.

## 프록시 객체의 내부 구조와 초기화 메커니즘

### 프록시를 획득하는 방법

- em.find() : 호출 즉시 DB에 SELECT 쿼리를 날려 진짜 엔티티를 반환한다.
- em.getReference() : DB 조회를 미루고 하이버네이트가 동적으로 가공한 가짜 프록시 객체를 반환한다.

```java
// 하이버네이트가 생성한 동적 프록시 클래스명이 출력됨
User proxyUser = em.getReference(User.class, 2L);
System.out.println(proxyUser.getClass().getName());
// 출력: kr.adapterz.jpa_practice.entity.User$HibernateProxy$xxxx
```

### 프록시 초기화의 단계별 흐름

proxyUser.getNickname()을 호출하여 실제 필드 값을 요구하는 순간, 내부에서는 다음과 같은 과정이 일어난다.

```java
System.out.println(proxyUser.getId());       // 1. 쿼리 안 나감(ID는 이미 프록시가 들고 있음)
System.out.println(proxyUser.getNickname()); // 2. 쿼리 발생(프록시 초기화 시작)
```

1. 메서드 호출 : 개발자가 프록시 객체의 getNickname()을 호출한다.
2. 영속성 컨텍스트 요청 : 프록시 객체는 내부에 진짜 엔티티 주소가 비어있는 것을 확인하고, 자신을 관리하는 영속성 컨텍스트에게 진짜 데이터를 찾아와달라고 요청한다.
3. DB조회 : 영속성 컨텍스트는 DB를 조회(SELECT * FROM user WHERE id = 2)한다.
4. 진짜 엔티티 생성 : DB 데이터를 기반으로 진짜 엔티티 객체를 영속성 컨텍스트 내부에 생성한다.
5. 참조 연결 및 위임 : 프록시 객체 내부의 target 필드에 방금 만든 진짜 엔티티의 메모리 주소값을 꽂아준다. 이제 프록시는 진짜 엔티티에게 메서드 호출을 위임하여 닉네임을 반환한다.

→ 프록시 객체가 초기화된다고 해서 프록시 객체가 진짜 엔티티 클래스로 변하는 것은 아니다. 껍데기는 그대로 유지된 채, 내부의 target이 진짜 객체를 가리키게 될 뿐이다.

## 타입 비교 시 == 사용 금지

**버그가 터지는 상황(안티패턴)**

```java
// loggedInUser는 실제 엔티티 (User)
// postAuthor는 지연 로딩된 프록시 객체 (User$HibernateProxy$xxxx)
if (loggedInUser.getClass() == postAuthor.getClass()) { 
    // 비즈니스 로직상 둘 다 똑같은 '유저' 엔티티인데 클래스명이 달라 false로 튕겨나감
    // 게시글 수정 권한이 없다는 억울한 예외 발생
}
```

올바른 해결책 : **instanceof** 사용

하이버네이트가 프록시 객체를 만들 때, 원본 엔티티 클래스를 상속 받아서 동적으로 구현한다. 자바 문법상 자식 클래스는 부모 클래스의 instanceof검사를 통과한다.

```java
if (postAuthor instanceof User) {
    // 진짜 엔티티든 가짜 프록시든 부모가 'User'인 것은 변함없으므로 정상적으로 true 반환
}
```

JPA엔티티 타입을 비교하거나 equals() 메서드를 재정의할 때는 .getClass() == 를 쓰면 안 되고, instanceof연산자를 사용해 설계한다.

## 프록시의 함정: `LazyInitializationException`

영속성 컨텍스트가 종료되거나 영속 상태가 해제(준영속 상태)된 프록시 객체를 뒤늦게 조회(초기화)하려고 할 때 발생하는 에러이다.

- **서비스 계층 실행:** `BoardService.getPost()` 메서드에 `@Transactional`이 걸려있어 영속성 컨텍스트가 켜진 상태로 `Post`를 조회한다. `Post` 안의 `User`는 **지연 로딩(프록시)** 상태다.
- **트랜잭션 종료 및 영속성 컨텍스트 퇴근:** 서비스 메서드가 끝나서 컨트롤러 계층으로 넘어오는 순간, 트랜잭션이 커밋되고 영속성 컨텍스트는 완전히 닫히며(종료) 유저 프록시 객체는 '준영속 상태'가 된다.
- **컨트롤러에서 직렬화(JSON 변환) 시도:** `BoardController`가 게시글 정보를 웹 브라우저에 JSON으로 내려주기 위해 객체를 파싱한다. 이때 Jackson 라이브러리가 유저의 닉네임까지 표현하려고 **`post.getUser().getNickname()`을 강제로 호출**한다.
- **에러 발생:** **영속성 컨텍스트는 이미 종료되어 사라진 상태**다. DB에 `SELECT` 쿼리를 날리지 못 하며 프록시는 결국 `org.hibernate.LazyInitializationException` 예외를 내던지며 서버를 다운시킨다.

### 해결책

#### DTO 사용

엔티티 객체를 컨트롤까지 가공 없이 그대로 던지는 것은 아주 위험하다. 트랜잭션이 살아있는 서비스 계층 안에서 필요한 데이터만 뽑아 담은 DTO객체로 변환해서 반환해야 한다.

```java
@Transactional(readOnly = true)
public PostResponseDto getPostDto(Long postId) {
    Post post = postRepository.findById(postId).orElseThrow();
    
    // 영속성 컨텍스트가 살아있는 이 공간에서 프록시 데이터를 꺼내 DTO로 옮겨 담는다
    return new PostResponseDto(
        post.getTitle(),
        post.getUser().getNickname() // 이 순간 안전하게 초기화 완료
    ); 
}
```

#### Fetch Join으로 한 번에 가져오기

컨트롤러단에서 쪼개져 쿼리가 나가는 것이 싫다면, 서비스단에서 fetch join 쿼리를 사용해 프록시 껍데기가 아닌 진짜 알맹이 데이터를 1번의 조인 쿼리로 미리 탑재해서 가져오는 최적화를 적용한다.

Q. JPA에서 `LazyInitializationException`이 발생하는 원인과 이를 방지하기 위한 해결 방법을 설명해 주세요.

지연 로딩된 프록시 객체는 스스로 DB를 조회할 수 없고, 반드시 자신을 생성한 영속성 컨텍스를 통해서만 초기화 가능하다.

이 예외는 @Transactional이 적용된 서비스 계층을 벗어나 영속성 컨텍스트가 종료된 컨트롤러나 뷰 계층에서 프록시 객체의 필드를 조회하려고 할 때 발생한다.

이를 방지하기 위해 영속성 컨텍스트가 살아있는 서비스 계층 내에서 필요한 연관 데이터를 모두 조회하여 DTO 객체로 완전히 변환 뒤 계층 간에 이동시키는 방식을 사용한다.

혹은 처음에 목록을 조회할 때부터 Fetch Join을 사용해 조인 쿼리 한번으로 연관 데이터를 함께 영속화하여 준영속 상태에서의 초기화 시도를 차단한다.

## 정리

- 프록시 : 지연 로딩을 구현하기 위해 실제 DB 조회를 미루고 반환하는 가짜 껍데기 객체이다.
- 프록시 내부의 target 참조 필드에 진짜 엔티티 객체의 메모리 주소를 연결하여 메서드 호출을 위임한다.
- 프록시는 클래스명이 동적으로 변하므로 클래스 동등비교(==)가 불가능하다. instanceof연산자를 사용해야 안전하다.
- 프록시는 영속성 컨텍스트가 닫히면 쓸모가 없다. 준영속 상태에서 필드 접근 시 발생하는 LazyInitializationException을 막기 위해 서비스단에서 DTO 변환을 끝내는 구조를 사용한다.
