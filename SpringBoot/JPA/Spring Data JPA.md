## 가져가야 할 핵심

- EntityManager 직접 다루는 순수 JPA랑 Spring Data JPA의 차이와 도입 이유
- JpaRepository가 제공하는 기본 메소드들이 내부적으로 EntityManager의 어떤 동작을 호출하는지

## 스프링 데이터 JPA

- 우리가 겪었던 불편함(순수 JPA)
    - 테이블이 100개면 유저 저장용 em.persist(user), 게시글 저장용 em.persist(post)가 담긴 리포지토리 클래스 100개를 복사 붙여넣기 하며 노가다를 해야 했다.
- 스프링 데이터 JPA
    - JPA 구현체를 스프링 진영에서 한 겹 더 감싸서 개발자는 인터페이스만 선언하고 반복되는 CRUD 구현체는 스프링 애플리케이션을 켤 때 가짜 프록시 객체로 동적 생성해서 주입해 주는 라이브러리이다.

## ddl-auto : create정책

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: create # 로컬 실습용, 운영 서버 적용 시 문제 발생
```

- create의 동작 : 애플리케이션이 켜질 때 DB에 기존 테이블이 있으면 DROP으로 밀고 엔티티 구조에 맞춰 새 테이블을 만든다.
- 운영 DB에 이게 켜진 채 배포하면 수천만 명의 회원 데이터가 1초 만에 흔적도 없이사라진다. 현업에서는 none 또는 validate(엔티티와 DB구조가 맞는지만 검증)로 지정하고, 테이블 변경은 Flyway나 Liquibase같은 전문 DB 마이그레이션 도구를 사용한다.

## JpaRepository가 EntityManager를 대체하는 방식

```java
// [순수 JPA 방식] 개발자가 일일이 다 짜고 관리해야 함
@Repository
public class UserJpaRepository {
    @PersistenceContext private EntityManager em;
    public void save(User user) { em.persist(user); }
    public User findById(Long id) { return em.find(User.class, id); }
    public List<User> findAll() { return em.createQuery("select u from User u", User.class).getResultList(); }
    public void delete(User user) { em.remove(user); }
}

// =========================================================================

// [스프링 데이터 JPA 방식] 인터페이스 하나 선언하고 상속받으면 끝
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
}
```

## save() 내부(persisst vs merge)

#### 1) 신규 엔티티인 경우(ID가 없을 때)

- JPA의 판단 : 객체에 식별자ID(null)가 없네? 완전히 처음 보는 새 데이터구나
- 내부 동작 : `EntityManager.persist()`를 호출하여 1차 캐시에 올리고 쓰기 지연 저장소에 `INSERT` 쿼리를 저장한다.

#### 2) 기존 엔티티인 경우(ID가 이미 채워져 있을 때)

- JPA의 판단 : 식별자ID가 이미 적혀있네? DB에 이미 있구나
- 내부 동작 : `persist()`가 아닌 `EntityManager.merge()`(병합)를 호출한다.

**merge()의 내부 매커니즘**

1. 현재 영속성 컨텍스트가 관리하고 있지 않은 준영속 객체인데 식별자 ID만 가지고 save()에 들어오는 상황이다.
2. merge()가 발동하면 JPA는 그 ID를 가지고 1차 캐시나 물리 DB에서 진짜 영속 상태인 기존 엔티티를 찾아서 꺼내온다.
3. 그리고 파라미터로 넘어온 가짜 객체 값들을 방금 찾아온 진짜 영속 엔티티에 복사해준다.

→ 파라미터로 전달한 객체 자체가 영속 상태가 되는게 아니라, 값이 복사된 완전히 새로운 영속 상태의 객체가 반환되는 구조이다.

**수정할 때 save()를 다시 부르면 안된다.**

merge()는 내부적으로 DB를 한 번 더 뒤져야 하고 원치 않는 필드가 null로 덮어써질 위험이 있는 등 복잡성이 커서 수정용으로 사용하지 않는다.

```java
// 비추천 (기존 ID를 억지로 쥐여주며 save를 호출하는 방식 -> merge 유발)
User user = new User(1L, "newNickname");
userRepository.save(user);

// 표준 정석 (조회 후 수정 -> 깔끔한 더티 체킹/변경 감지 활용)
User user = userRepository.findById(1L).orElseThrow();
user.setNickname("newNickname"); // 트랜잭션 종료 시 자동으로 최적화된 UPDATE 쿼리 방출!
```

수정할 때는 save()를 또 호출할 필요가 없다. 트랜잭션 안에서 조회 후 값만 바꾸는 ‘변경 감지’가 가장 안전하고 깔끔하다.

## 리포지토리 반환 타입

스프링 데이터 JPA는 리포지토리 메서드를 선언할 때 적어둔 자바 반환 타입을 보고 내부 예외처리를 알아서 가공해준다.

#### 컬렉션 반환(List<User>)

```java
List<User> findByNickname(String nickname);
```

- 스프링 데이터 JPA는 조건에 맞는 데이터가 한 건도 없으면 null이 아니라 텅 빈 컬렉션(`Collections.emptyList()`)을 안전하게 반환한다.

#### 단건 반환(User)

```java
User findByNickname(String nickname);
```

- **순수 JPA의 한계:** 순수 JPA에서 단건 조회를 할 때 쓰는 `getSingleResult()`는 데이터가 없으면 `NoResultException` 예외를 던진다.
- **스프링 데이터 JPA의 처리:** 스프링은 이 예외를 자기가 중간에 가로채고, 개발자에게 `null`을 넘겨준다.
- 반환 타입이 `null`이면 개발자가 널 체크를 깜빡하는 순간 런타임에 에러가 터질 위험이 여전히 남아 있다.

#### 표준 : Optional 단건 반환 (`Optional<User>`)

```java
Optional<User> findByNickname(String nickname);
```

- 데이터가 없으면 Optional.empth()를 반환해 준다.
- 호출하는 서비스 레이어단에서 `userRepository.findByNickname("SOO").orElseThrow(() -> new IllegalArgumentException("존재하지 않는 회원입니다"))` 처럼 **가독성 높고  안전한 에러 핸들링 코드를 유도**할 수 있기 때문에,  단건 조회를 할 땐 `Optional`로 감싸서 선언한다.

**Q. 스프링 데이터 JPA와 순수 JPA의 차이점은 무엇이며, 생산성이 좋음에도 JPQL이나 QueryDSL을 섞어 써야 하는 이유는 무엇인가요?**

순수 JPA는 개발자가 EntityManager를 활용해 반복적인 CRUD코드를 직접 구현해야 하지만, 스프링 데이터JPA는 인터페이스 선언과 JpaRepository 상속만으로 구현체를 동적으로 생성해 주어 코드를 줄여준다.

다만, 스프링 데이터 JPA는 복잡한 다중 조인, 통계성 쿼리, 또는 조건에 따라 쿼리가 변하는 ‘동적 쿼리’를 메서드 이름만으로 표현하는데 한계가 있다. 메서드명이 무한정 길어지거나 성능 최적화가 어렵기 때문에, 복잡한 특수 쿼리는 JPQL을 직접 작성하거나 QueryDSL을 도입해 결합하는 것이 좋다.

**Q. Spring Data JPA의 save() 메소드 내부적으로 어떻게 동작하는지 아세요?**

A. 파라미터로 넘어온 엔티티 식별자 확인해서 값이 없으면 새로운 객체로 판단해서 persist 호출하고 값이 있으면 이미 존재하는 객체로 판단해서 merge() 호출합니다.
