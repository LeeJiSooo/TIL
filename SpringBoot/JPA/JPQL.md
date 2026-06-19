## 가져가야 할 핵심

- JPQL은 테이블이 아닌 엔티티 객체를 대상으로 작성하는 쿼리 언어
- UPDATE, DELETE같은 벌크 연산을 수행할때 영속성 컨텍스트와 DB간의 상태 불일치 문제

## 왜 SQL을 두고 JPQL을 배워야 하는가?

- 우리가 직면한 한계
    - 지금까지는 em.find(User.class, 1L)처럼 PK기반 단건 조회나 post.getUser()같은 객체 그래프 탐색 위주로 데이터를 다뤘다.
    - 하지만 어드민, 정산, 내부 모니터링 시스템 등에서는 “나이가 20세 이상이고 가입일이 한달 이내인 유저 목록을 최신순으로 정렬해서 가져와라”같은 복잡한 다중 조건 검색이 쏟아진다.
- SQL vs JPQL
    - 순수 SQL : 데이터베이스의 테이블과 컬럼을 대상으로 직접 쿼리를 날린다. (`SELECT * FROM users`)
    - JPQL : 테이블이 존재하든 말든 상관없이, 자바의 엔티티 객체와 필드를 대상으로 작성하는 객체지향 쿼리 언어이다. (`select u from User u`)

**JPQL을 사용하는 이유?**

만약 우리가 직접 SQL을 작성해서 코드를 짜두면, 나중에 서비스가 커져서 DB를 MySQL에서 Oracle이나 PostgreSQL로 교체할 때 DB전용 함수나 문법이 달라져 쿼리를 다 고쳐야 하는 문제가 생긴다.

하지만 추상화되 JPQL로 쿼리를 작성해 JPA에게 던져주면, JPA가 우리가 설정한 DB방언에 맞춰서 물리 SQL문방으로 알아서 번역 및 실행을 해주기 때문에 유연한 아키텍처를 유지할 수 있다.

## JPQL 문법

```sql
SELECT u
FROM User u
WHERE u.nickname = 'tester'
GROUP BY u.userRole
HAVING count(u) > 1
ORDER BY u.createdAt DESC
```

- **원칙 1 (대상이 엔티티):** `FROM User`에서 `User`는 DB 테이블 이름이 아니라 자바의 **엔티티 클래스 이름이다.** 뒤에 붙은 필드명(`u.nickname`, `u.createdAt`)도 테이블 컬럼명이 아닌 엔티티 내부의 **자바 필드명**이며 대소문자를 구분한다.
- **원칙 2 (별칭 필수):** SQL과 달리 JPQL에서는 **엔티티의 별칭(Alias, 위 코드의 `u`)을 무조건 필수**로 부여해야만 정상 한다. (단, `as` 키워드는 생략 가능)
- **원칙 3 (키워드 무관):** `SELECT`, `FROM`, `WHERE` 같은 JPQL 자체 예약어는 대소문자를 구분하지 않아도 된다.

## 벌크연산 : 영속성 컨텍스트 문제

대량의 데이터를 수정해야 할 때 변경 감지(더티 체킹)를 쓰면 서버가 터진다.

- 변경 감지의 한계 : 100만 명의 유저에게 보너스 포인트를 1,000점씩 올려주려면 100만 명의 엔티티를 em.find()로 가져와 1차 캐시에 올린 뒤 자바 코드로 값을 바꿔야 한다. 수백만 개의 자바 객체가 메모리에 차는 순간 OOME(메모리 고갈 에러)로 서버가 터진다.
- 벌크 연산 : 이럴 때 단 한줄의 UPDATE 또는 DELETE 쿼리로 수많은 레코드를 한방에 밀어 버리는 기술이 벌크연산이다. JPQL로 직접 짜는 수정/삭제 문이 여기에 해당한다.

#### 영속성 컨텍스트와 DB데이터 불일치 버그 시나리오

벌크 연산은 강력하지만, 영속성 컨텍스트를 완전히 무시하고 데이터베이스에 직접 쿼리를 날리는 위험한 특징이 있다.

```java
// 1. 유저를 조회해서 1차 캐시에 올림
User user = entityManager.find(User.class, 1L); // 1차 캐시 속 닉네임: "tester"

// 2. 벌크 연산 (JPQL UPDATE) 단 한 번으로 DB를 직접 밀어버림
int updatedCount = entityManager.createQuery(
        "update User u set u.nickname = 'guest' where u.id = 1L")
    .executeUpdate(); // DB상의 닉네임은 "guest"로 교체 

// 3. 다시 유저의 닉네임을 출력해보면?
System.out.println("유저 닉네임: " + user.getNickname()); // 결과: "tester" 
```

- 벌크 연산이 물리 DB데이터는 “guest”로 바꿔놨지만, 정작 자바 메모리 속 1차 캐시에 살아있는 user객체는 이 사실을 전혀 모른 채 옛날 데이터인 “tester”를 쥐고 있다.

→ 데이터 정합성 문제

#### 해결책 : entityManager.cler()

벌크 연산 실행 후에 영속성 컨텍스트를 청소해 주는 것을 사용한다.

```java
// 1. 벌크 연산 실행
entityManager.createQuery("update User u set u.nickname = 'guest' where u.id = 1L").executeUpdate();

// 2. 영속성 컨텍스트 초기화
entityManager.clear();

// 3. 이제 다시 조회하면 1차 캐시가 비어있으므로 DB에서 "guest"로 바뀐 최신 유저를 새로 긁어옴
User refreshedUser = entityManager.find(User.class, 1L);
System.out.println("유저 닉네임: " + refreshedUser.getNickname()); // "guest" 정상 출력
```

## 파라미터 바인딩

쿼리에 자바 변수 값을 넣어 동적으로 조회할 때 사용하는 파라미터 바인딩 방식은 2가지가 있다.

#### 위치 기반 바인딩 (물음표 `?1`) → 안 씀

```java
TypedQuery<User> query = em.createQuery("select u from User u where u.email = ?1", User.class);
query.setParameter(1, "tester@adapterz.kr");
```

지금은 `?1` 자리에 이메일이 들어가는 게 맞지만, 나중에 유지보수를 하다가 중간에 이름 검색 조건이 추가되어서 `where u.name = ?1 and u.email = ?2` 처럼 순서가 밀려버리면 기존에 세팅해 둔 숫자 넘버링을 다 찾아서 바꿔줘야 한다. 이걸 깜박해버리면 어뚱한 파라미터가 매핑되어 런타임에 데에터가 꼬이는 문제가 발생한다.

#### 이름 기반 바인딩 (콜론 `:name`)

```java
TypedQuery<User> query = em.createQuery("select u from User u where u.nickname = :nickname", User.class);
query.setParameter("nickname", "tester"); // 문장 중간에 다른 조건이 끼어들어도 이름 기준이라 안 깨짐
```

## JPQL의 한계와 대안(with QueryDSL)

JPQL의 단점은 쿼리가 결국 ‘자바 문자열(String)’이라는 점이다.

```java
// 개발자가 실수로 엔티티 이름에 r을 하나 더 붙여 오타를 냄 (Userr)
String jpql = "select u from Userr u where u.nickname = :nickname";
TypedQuery<User> query = entityManager.createQuery(jpql, User.class);
```

- 자바 컴파일러 입장에서는 쌍따옴표 안에 든 문자열이 오타가 났든 말든 통과시켜 줘서 컴파일 에러가 안 난다.  ‘런타임 시점’에 프로그램이 터져버린다.

QueryDLS : 오픈소스 라이브러리인데, 자바 코드로 완벽하게 쿼리를 짤 수 있게 도와준다.

```java
// QueryDSL 코드 예시
List<User> users = queryFactory
    .selectFrom(user)
    .where(user.nickname.eq("tester"))
    .fetch();
```

- `selectFrom()`, `where()`, `eq()` 같은 모든 문법이 **순수 자바 메서드**로 이루어져 있다.
- 따라서 엔티티 이름이나 필드명에 오타를 내면 **IDE가 그 즉시 빨간 줄을 긋고 컴파일 자체를 막는다.** → 런타임 에러를 안전한 **컴파일 타임 에러**로 끌어올려 준다.
- 단, QueryDSL도 결국 내부적으로는 자바 코드를 가공해서 **JPQL을 생성해 전달해 주는 빌더 역할**일 뿐이다.

**Q. 순수 SQL과 JPQL의 결정적인 차이점이 무엇인가?**

가장 큰 차이는 ‘쿼리의 대상’이다. 순수 SQL은 데이터베이스의 물리적인 테이블과 컬럼을 대상으로 작성되지만, JPQL은 자바 영속성 컨텍스트에 등록된 엔티티 객체와 그 내부 필드를 대상으로 작성하는 객체지향 쿼리이다.

이 덕분에 특정 데이터베이스 문법에 종속되지 않고 방언 추상화가 가능해져서, 추후 데이터베이스가 변경되더라도 비즈니스 로직의 변경 없이 JPA가 유연하게 물리 SQL로 번역 및 최적화를 수행해 준다.

**Q. JPA에서 벌크 연산을 수행할 때 발생할 수 있는 문제점과 해결책?**

벌크 연산은 영속성 컨텍스트를 완전히 우회하고 데이터베이스에 직접 UPDATE나 DELETE 쿼리를 날리는 특징이 있다. 이로 인해 DB데이터는 최신 상태로 수정 되었지만, 1차 캐시 메모리 속 엔티티 객체는 과거 데이터를 그대로 가지고 있는 ‘메모리-DB 간의 데이터 상태 불일치’장애가 발생한다.

이를 해결하기 위해 벌크 연산을 수행한 직후에는 entityManager.clear()를 명시적으로 호출하여 영속성 컨텍스트를 완전히 초기화해 주어야 한다. 그렇게 함으로써 이후 데이터 조회 시 1차 캐시가 아닌 DB에서 수정된 최신 데이터를 새로 안전하게 읽어오도록 유도해 데이터 정합성을 방어한다.
