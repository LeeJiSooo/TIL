<img width="1448" height="744" alt="image" src="https://github.com/user-attachments/assets/57fdb841-1ecb-4ef6-b4af-0a19e88c8ada" />


## 가져가야 할 핵심

1. 트랜잭션이 JPA 영속성 컨텍스트와 어떻게 결합하여 DB동기화(Flush)시점을 결정하는지 이해한다.
2. **`@Transactional(readOnly = true)`** 옵션이 왜 스냅샷 메모리 낭비를 줄여 주는지 파악한다.
3. OSIV(Open Session In View)의 편리함 뒤에 숨겨진 **커넥션 풀 고갈 장애의 원인과 대안을 이해한다.**

## 트랜잭션

- 트랜잭션의 기본 정의
    - 데이터베이스의 상태를 변화시키기 위해 수행하는 하나의 논리적인 작업 단위
- JPA에서 트랜잭션이 갖는 진정한 의미
    - 영속성 컨텍스트의 생명주기와 완벽하게 결합되어 객체의 상태 변화를 DB와 안전하게 동기화하는 바운더리(경계선)역할을 한다.
    - JPA는 엔티티의 상태 변화나 `persist()` 호출을 감지하더라도 즉시 DB에 반영하지 않고, 1차 캐시와 쓰기 지연 저장소에 차곡차곡 기록만 해둔다.
    - 이 모아둔 작업을 실제 DB에 적용하라고 누르는 버튼이 ‘트랜잭션 커밋’이다.
- 트랜잭션 바깥의 세계
    - 트랜잭션 경계 바깥에서는 영속성 컨텍스트가 정상적으로 유지되지 않으며, JPA 핵심 기능들이 동작하지 않는다. 오진 트랜잭션 바운더리 안에서만 유효하다.

## 트랜잭션의 동작 메커니즘과 플러시(Flush)

1. **트랜잭션 시작** : 스프링에서 `@Transactional`이 붙은 메소드가 실행되면, JPA는 “아 이제부터 일어나는 엔티티의 모든 변화를 추적해야겠구나”하고 준비한다.
2. **비즈니스 로직 수행** : 자바 코드로 필드 값을 바꾸면 영속성 컨텍스트 내부에서 변경 사항이 관리된다.
3. **트랜잭션 커밋(플러시 발동)** : 메서드가 정상 종료되어 커밋되는 순간, 모아둔 쿼리를 DB로 밀어 넣는 플러시가 내부적으로 먼저 일어나고 실제 DB 커밋이 완료된다.
4. **예외 발생 시(롤백)** : 만약 로직 중간에 예외가 발생해 롤백되면, 영속성 컨텍스트에 모아둔 모든 변경 사항과 쓰기 지연 쿼리들은 깔끔하게 무효화된다.

## 조회 전용 트랜잭션 최적화(`readOnly = true` )

우리가 만드는 애플리케이션은 데이터를 생성/수정 하는 쓰기 작업보다, 데이터를 읽어오는 읽기(조회)작업이 압도적으로 많다.

**데이터를 읽기만 하는데도 `@Transactional`을 붙여야 하나?**

붙이는게 좋다. 지연로딩 같은 JPA 핵심 기능들이 기본적으로 트랜잭션 범위 안에서만 안전하게 동작하기 때문이다.

### 일반 트랜잭션을 썻을 때의 메모리 낭비

데이터를 전혀 수정할 일이 없는 단순 조회 메서드인데도 일반 **`@Transactional`** 을 열어두면, JPA 입장에서는 “이 개발자가 언제 데이터를 바꿀지 모르니 일단 감시하자”하고 1차 캐시에 스냅샷(최초 상태 복사본)을 생성한다.

- 만약 목록 페이지에서 수만 건의 데이터를 한 번에 조회한다면, 수정하지도 않을 수만 건의 엔티티마다 스냅샷 사진을 전부 찍어야 해서 메모리 사용량이 폭발하고 낭비가 심해진다.

### @Transactional(readOnly = true)

이 옵션을 켜면 스프링과 JPA에게 “나 이 메서드 안에서는 데이터를 읽기만 할게”라고 하는 것이다.

- 스냅샷 생성 전략 : 읽기 전용임을 알아챈 JPA는 아예 1차 캐시에 스냅샷을 생성하지 않는다. → 메모리 사용량이 획기적으로 줄어든다.
- 더티 체킹 생략 : 트랜잭션이 끝날 때 원본 스냅샷이 없으니 당연히 값이 바뀌었는지 대조하는 변경 감지 과정 자체를 패스한다. → CPU 연산 성능 최적화

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true) // 1. 클래스 레벨에 기본적으로 '읽기 전용'을 깔고 간다!
public class UserService {

    private final UserRepository userRepository;

    // 클래스 레벨의 설정을 이어받아 readOnly = true로 안전하고 가볍게 조회
    public List<User> findAllUsers() {
        return userRepository.findAll();
    }

    // 2. 쓰기(CUD)가 필요한 메소드에만 명시적으로 일반 @Transactional을 붙인다
    @Transactional 
    public void updateNickname(Long userId, String newNickname) {
        User user = userRepository.findById(userId).orElseThrow();
        user.changeNickname(newNickname); // 변경 감지(더티 체킹) 정상 동작!
    }
}
```

- 우선순위 원칙 : 자바와 스프링에서는 항상 더 구체적인 설정(메서드 레벨)이 더 넓은 설정(클래스 레밸)보다 우선순위가 높다. 따라서 `updateNickname` 은 읽기 전용 설정을 덮어쓰고 정상적인 쓰기 트랜잭션으로 작동하게 된다.

## OSIV(Open Session In View) 전략과 트레이드 오프

말 그대로 View(컨트롤러나 Thymeleaf 뷰 렌더링 영역)까지 Session(영속성 컨텍스트)을 Open(열여둔다)해두는 전략이다.

기본 원칙대로라면 비즈니스 로직을 처리하는 Service계층이 끝나는 순간 트랜잭션이 닫히고 영속성 컨텍스트도 종료되어야 한다.

하지만 서비스 계층 밖으로 나가는 순간 엔티티는 준영속 상태가 되어버린다. 이 상태에서 `UserController`(컨트롤러 계층)나 API 응답을 위해 JSON으로 변환하는 단계에서 연관된 데이터를 뒤늦게 끄집어 쓰려고 하면 문제가 터진다.

- *ex) `user.getPosts().size()` 를 호출해 유저가 쓴 게시글 목록을 가져오려고 할 때 (지연 로딩)*
- 엔티티가 이미 준영속 상태라 JPA는 데이터를 가져오고 싶어도 조율해 줄 영속성 컨텍스트가 없고, DB 커넥션도 끊겨있다.
- 결과는 **`org.hibernate.LazyInitializationException` (세션이 없어서 지연 로딩 초기화 실패)** 에러를 뱉으며 서버가 뻗어버린다.

### 스프링의 해결책 : OSIV On(Default)

스프링은 개발자가 컨트롤러나 뷰 영역에서도 편하게 지연 로딩을 할 수 있도록 했다. 클라이언트의 HTTP요청이 들어오는 순간부터 응답이 완전히 나갈 때 까지 영속성 컨텍스트를 죽이지 않고 계속 살려둔다. 

→ 컨트롤러 계층에서도 영속성 컨텍스트가 살아있어서 아무 제약 없이 연관 객체를 지연 로딩해서 가져다 쓸 수 있어 유연하고 편하다.

### 커넥션풀 고갈 장애

하지만 영속성 컨텍스트를 응답 나갈 때까지 길게 살려두면 그 시간 동안 데이터베이스 커넥션을 놓지 않고 가지고 있다는 뜻이다.

- 만약 우리 서비스의 컨트롤러에서 외부 카카오 API를 호출하거나, 대용량 파일을 업로드하는 등 시간이 오래 걸리는 작업이 진행된다면?
- 트랜잭션(비즈니스 로직)은 이미 끝났는데도, 컨트롤러가 끝날 때까지 DB커넥션을 가지고 있게된다.
- 동시 접속자(트래픽)가 많다면 커넥션 풀의 모든 선이 마비되어 뒤이어 들어오는 유저들이 커넥션을 얻지 못해 시스템 전체가 다운되는 커넥션 풀 고갈 장애가 발생한다.

### 실무 권장 : OSIV끄기**(`spring.jpa.open-in-view=false`)**

그래서 실무 및 대규모 트래픽 환경에서는 이 옵션을 명시적으로 끄고 개발하는 것이 표준 권장사항이다.

**OSIV를 끄면 발생하는 지연 로딩 에러는 어떻게 해결해야되나?**

애초이 트랜잭션이 살아있는 서비스 계층 안에서 지연 로딩을 끝내거나 Fetch Join으로 데이터를 다 긁어온 뒤, 엔티티가 아닌 순수 데이터 덩어리인 DTO로 변환해서 컨트롤러로 넘겨준다.

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class UserService {

    private final UserRepository userRepository;
		
		// service 계층 안에서 DTO 변환을 모두 마칩니다.
    public UserProfileDto getUserProfile(Long userId) {
        User user = userRepository.findById(userId).orElseThrow();

        // 트랜잭션 안에서 지연 로딩 발생 (정상 동작)
        int postCount = user.getPosts().size();

        // 엔티티를 밖으로 내보내지 않고 순수한 데이터 덩어리인 DTO로 변환
        return new UserProfileDto(user.getName(), postCount);
    }
}

@RestController
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @GetMapping("/users/{id}/profile")
    public UserProfileDto getProfile(@PathVariable Long id) {
        // Controller는 지연 로딩에 신경 쓸 필요 없이 DTO만 받아서 JSON으로 응답
        return userService.getUserProfile(id);
    }
}
```

Q. OSIV가 무엇이고 끄는 것과 켜는 것중 어떤걸 선택해야하는가?

OSIV는 클라이언트의 HTTP요청 시점부터 응답이 완료되어 나갈 때가지 JPA 영속성 컨텍스트를 뷰/컨트롤러 계층까지 개방해두는 전략이다.

컨트롤러 계층에서도 자유롭게 지연 로딩을 사용할 수 있어 개발 생산성이 높아진다는 장점이 있지만, 영속성 컨텍스트를 살려두기 위해 DB커넥션까지 너무 오랜 시간 점유한다는 단점이 있다. 컨트롤러에서 외부 API를 호출하거나 무거운 작업을 할 때 커넥션을 반납하지 않아 트래픽이 몰릴 경우 커넥션 풀 고갈로인한 시스템 장애를 유발하기 쉽다.

따라서 대규모 트래픽을 감당해야 하는 실무 환경에서는 OSIV옵션을 명시적으로 끄는 것이 좋다. 대신, 지연 로딩 에러를 막기 위해 서비스 계층 내에서 JPQL의 Fetch Join을 활용해 필요한 데이터를 한 번에 조회하거나, 트랜잭션 범위 안에서 지연 로딩 초기화를 완벽히 마친 후 순수한 DTO로 변환하여 컨트롤러에게 반환하는 구조를 선택해야 한다.

## 정리

- 트랜잭션은 영속성 컨텍스트에 차곡차곡 쌓여있던 쓰기 지연 쿼리들을 데이터베이스에 한방에 밀어 넣고 동기화(Flush)시키는 통제 경계선이다.
- 단순 조회 메서드에는 `@Transactional(readOnly = true)` 를 필수로 붙여 불필요한 스냅샷 복사본 생성을 원천 차단함으로써 메모리와 CPU연산 성능을 최적화해야 한다.
- OSIV전략은 컨트롤러단 지연 로딩의 편리함을 주지만 대형 장애의 주범인 DB 커넥션 풀 고갈을 유발하므로, 명시적으로 끄고 서비스 계층에서 DTO로 변환하여 컨트롤러로 넘기는 방식이 가장 안전하다.
