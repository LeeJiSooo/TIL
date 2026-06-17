## 가져가야 할 핵심

1. 외래키는 항상 N쪽에 존재하므로, 매핑할 때도 N쪽 필드가 연관관계의 주인이 된다.
2. 객체 그래프 탐색을 위한 단방향과 양방향의 차이를 명확히 안다.(mappedBy활용)
3. 지연로딩을 기본 컨벤션으로 써야 하는 이유와 즉시 로딩이 일으키는 N+1문제의 메커니즘을 이해한다.

## 다대일 단방향 vs 양방향 매핑 구조

### 다대일(N : 1) 단방향

N쪽에서 1쪽을 참조하는 필드를 둔다. DB 테이블 관점에서는 단방향이든 양방향이든 외래키 하나라 똑같지만, 자바 객체 세계에서만 방향이 단방향이다.

```java
**@Entity
public class Post {
    @Id @GeneratedValue
    private Long id;
    private String title;

    @ManyToOne
    @JoinColumn(name = "user_id") // DB의 user_id FK 컬럼과 매핑
    private User user; // 연관관계의 주인!
}**
```

- 한계 : post.getUser().getName()으로 객체 그래프 탐색은 잘 되지만, “특정 사용자가 작성한 게시글 목록을 다 가져와라”라는 요구사항이 오면 객체 그래프 탐색이 불가능해져서 복잡한 JPQL을 짜야한다.

### 다대일(N : 1) 양방향

역방향 조회를 위해 1쪽(User)에 List<Post>필드를 추가한다. DB 스키마는 동일하며 자바 구조만 바뀐다.

```java
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @OneToMany(mappedBy = "user") // 외래키가 없는 주인이 아닌 쪽 (읽기 전용)
    private List<Post> posts = new ArrayList<>();
}
```

- mappedBy : 주인이 아니고 읽기 전용이다. 진짜 주인은 Post클래스의 user필드여서 그걸 보고 외래키를 관리해달라고 JPA에게 알려주는 역할이다.

## Fetch 방식 전략 : LAZY(지연 로딩) vs EAGER(즉시 로딩)

연관관계에 있는 데이터를 “어느 시점에 DB에서 끌어올 것인가?”를 결정하는 성능 최적화 개념이다.

### 지연로딩(FetchType.LAZY) : 성능 최적화의 기본

- 연관된 데이터를 즉시 조회하지 않고, 실제 코드 상에서 그 데이터가 필요해져서 사용되는 순간까지 DB 쿼리를 미루는 방식이다.

```java
@ManyToOne(fetch = FetchType.LAZY) 
@JoinColumn(name = "user_id")
private User user;
```

**지연로딩 내부 메커니즘**

```java
// 1. Post만 조회 (User는 건드리지 않음)
Post post = em.find(Post.class, 1L);  
System.out.println(post.getTitle());  
// 실행 SQL: SELECT * FROM post WHERE id = 1 (User 테이블 조인 안 함)
```

- 이 시점에 JPA는 Post 객체 내부의 User user 필드에 실제 데이터가 없는 텅 비어있는 가짜 객체인 ‘프록시 객체’를 대신 채워둔다.

```java
// 2. 나중에 실제 User의 데이터가 필요한 순간
System.out.println(post.getUser().getName());  
// 실행 SQL: SELECT * FROM user WHERE id = ? (뒤늦게 쿼리 발생)
```

- 비즈니스 로직 상에서 getName()을 호출해 실제 값이 필요한 순간이 오면, JPA가 프록시 객체를 진짜 DB 데이터로 채워 넣는 ‘프록시 초기화’가 일어난다.

→ 게시글 제목만 보여주는 화면이라면 쓸데없이 회원 테이블까지 가지 않기 때문에 서버 자원과 네트워크 비용을 아낄 수 있따.

### 즉시 로딩(FetchType.EAGER)

- 엔티티를 조회하는 그 순간, 연관된 데이터까지 JOIN쿼리로 한 번에 다 긁어오는 방식이다.

```java
@ManyToOne(fetch = FetchType.EAGER) 
@JoinColumn(name = "user_id")
private User user;
```

**단건 조회 시의 동작**

```java
Post post = em.find(Post.class, 1L);
// 실행 SQL: SELECT p.*, u.* FROM post p JOIN user u ON p.user_id = u.id
```

- 단건 조회(em.find)를 할 때는 JPA가 알아서 쿼리를 날려주니 아무 문제가 없어 보이지만 전체 목록을 조회하는 순간 문제가 발생한다.

## 즉시 로딩이 불러오는 ‘N+1문제’

만약 메인 화면에 띄울 전체 게시글 목록 10개를 JPQL로 가져오는 상황을 가정해보자

```java
// 모든 Post를 JPQL로 조회
List<Post> posts = em.createQuery("select p from Post p", Post.class).getResultList();
```

1. 개발자의 의도 : 게시글 전체를 가져와 줘 → JPA는 JPQL을 순수하게 해석해서 DB에 SELECT * FROM post 쿼리를 딱 한번 날린다.
2. DB의 응답 : DB는 10개의 게시글 데이터를 반환한다.
3. JPA EAGER규칙 발동 : JPA가 메모리에 10개의 Post객체를 올리고 보니, 내부의 User 필드가 즉시 로딩으로 설정되어 있다. 규칙상 유저 객체도 다 채워진 상태여야 한다.
4. 추가 쿼리 방출 : JPA는 각 Post 객체를 돌면서 비어있는 작성자 정보를 채우기 위해 DB에 추가로 쿼리를 따로띠로 날리기 시작한다.
    - 1번 게시글 유저 찾기: `SELECT * FROM user WHERE id = ?`
    - 2번 게시글 유저 찾기: `SELECT * FROM user WHERE id = ?`
    - ... (총 10번 반복)

→ N + 1문제 : 게시글 목록 가져오려고 쿼리를 딱 1번 날렸는데, 연관된 유저 데이터를 가져오려고 추가 쿼리가 N번 더 실행되는 성능 저하 현상이 터진다. 만약 글이 1,000개면 추가 쿼리만 1,000번이 나간다.

## LAZY는 통제권을 주는 도구이다.

사람들이 오해하는 게 “LAZY를 쓰면 N+1 문제가 완전히 해결된다”라고 생각하지만, 그렇지는 않다. LAZY 상태에서도 게시글 목록 10개를 가져온 뒤, 자바 반복문을 돌면서 post.getUser().getName()을 호출하면 결국 뒤늦게 프록시가 초기화되면서 똑같이 10번의 추가 쿼리가 날라간다.

### 그럼에도 LAZY를 사용하는 이유

- EAGER문제 : 쿼리가 실행되는 그 시점에 개발자가 개입하거나 통제할 방법이 없이 N+1쿼리가 터진다.
- LAZY의 가치 : 일단 쿼리가 폭탄으로 나가는 타이밍을 미루고 통제권을 개발자에게 넘겨준다. 개발자는 일단 전체 연관관계를 LAZY로 다 막아서 쿼리 폭탄을 방어해두고, 진짜로 한 번에 조인해서 가져와야 하는 화면(게시글 + 작성자 목록)이 필요할 때만 fetch join이나 @EntityGraph를 사용해 쿼리 한번으로 묶어서 가져오는 최적화전략을 취할 수 있다.

## 정리

- 1: N관계에서 외래키는 N쪽에 존재하며, 객체 세계에서도 외래키를 쥐고 있는 N쪽 필드가 진짜 연관관계의 주인이 된다.
- mappedBy : 주인이 아닌 1쪽에 설정하며, 읽기 전용임을 뜻하고 진짜 주인의 필드명을 JPA에게 명시해 준다.
- 즉시 로딩 상태에서 JPQL 목록 조회를 하면 전체 조회 쿼리(1) 외에 데이터 개수만큼의 추가 유저 조회 쿼리(N)가 발생한다.
- 다대일 매핑은 LAZY로 선언하여 쿼리 통제권을 개발자가 쥔 상태로 개발하고, 한 번에 조인이 필요한 시점에만 fetch join을 사용해 해결한다.
