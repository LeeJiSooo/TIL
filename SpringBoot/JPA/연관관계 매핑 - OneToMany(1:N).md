## 가져가야 할 핵심

1. 1 : N 단방향의 위험성 : 자바 객체 중심의 착각으로 1쪽을 주인으로 삼았을 때 터지는 추가 UPDATE 쿼리와 @JoinColumn생략 시 생기는 조인 테이블의 피해를 이해한다.
2. 다대일 양방향이 표준인 이유 : 1:N양방향이 존재하지만 결국 외래키가 있는 N쪽을 주인으로 삼는 다대일 양방향이 표준 구조가 됨을 인지한다.
3. 영속성 전이(Cascade)와 고아 객체 제거(Orphan Removal) : 부모-자식 간의 생명주기를 묶어주는 도구들의 제약사항을 이해한다.

## 일대다(1:N)매핑

- 관계의 정의
    - 1에 해당하는 엔티티가 N에 해당하는 엔티티들을 관리하는 구조이다.
    - 자바 객체 세상에서는 여러 개의 객체를 담아야 하므로 List,Set 같은 컬렉션 타입을 이용해 1:N관계를 표현한다.
- 패러다임 충돌
    - 객체는 1쪽에 List<Post>를 쥐고 주도적으로 관리할 수 있지만, 관계형 데이터베이스에 외래키는 무조건 N쪽에 있다. 이 구조적 차이 때문에 1:N 단방향 설계을 하는 순간 문제가 생긴다.

## 일대다 단방향 매핑의 문제

### 왜 추가 UPDATE쿼리가 터질까?

- User에만 List<Post>가 있고, Post에는 User참조 필드가 아예 없는 상황이다.
- 새로운 회원과 게시글 2개를 저장(persist)한다고 가정해보자
    1. JPA는 먼저 User를 INSERT하고, Post 2개를 INSERT한다.
    2. 하지만 Post엔티티 안에는 작성자 정보가 없으므로, Post를 INSERT할 당시에는 외래키(user_id)값을 채울 방법이 없어 null로 먼저 들어간다.
    3. 이후 트랜잭션이 끝날 때 JPA가 User객체의 List<Post>를 보고 나서야 뒤늦게 post테이블의 외래키를 채워 넣는 수동 UPDATE쿼리가 발생한다.

### @JoinColumn을 빼먹으면 발생하는 문제

만약 List<Post>필드 위에 @OneToMany만 적고 @JoinColumn을 적지 않으면 JPA는 외래키가 어디에 있는지 차지 못해 중간에서 두 테이블을 강제로 엮어주는 ‘조인 테이블’전략을 선택한다.

→ 원래 없어야 할 user_post라는 매핑용 연결 테이블이 DB에 강제로 새로 생성된다.

→ 테이블이 하나 더 늘어나니 데이터를 저장할 때마다 중간 테이블에 INSERT쿼리가 또 나가고, 조회할 때도 테이블 3개를 조인해야 하므로 성능이 안 좋아 진다. 따라서 실수로라도 1:N단방향을 쓸 떈 @JoinColumn을 사용해야 한다.

## 다대일 양방향 매핑을 사용하자

- 1:N양방향(안티패턴) : 1쪽인 User.posts를 주인으로 삼고, N쪽인 Post.user를 읽기 전용으로 만드는 방식 → 1:N 단방향의 단점(UPDATE 쿼리 무한)을 그대로 가져가므로 안 쓴다.
- N:1양방향 : 물리 외래키가 있는 N쪽을 주인으로 삼고, 1쪽에 mappedBy를 붙여 읽기 전용으로 만드는 방식

```java
// 1쪽 (주인이 아님 - 읽기 전용)
@Entity
public class User {
    @OneToMany(mappedBy = "user") 
    private List<Post> posts = new ArrayList<>();
}

// N쪽 (외래키를 관리하는 진짜 연관관계의 주인)
@Entity
public class Post {
    @ManyToOne
    @JoinColumn(name = "user_id") 
    private User user; 
}
```

이렇게 N쪽을 주인으로 삼아야 게시글을 INSERT할 때 외래키(user_id)가 쿼리 한 방에 깔끔하게 들어가서 양방향이 필요하면 이 구조를 사용한다.

## Cascade(영속성 전이)

부모 엔티티에서 내린 JPA명령(PERSIST, REMOVE 등)을 연관된 자식 엔티티에게도 똑같이 연쇄적으로 전파시키는 기능

- `CascadeType.PERSIST` (함께 저장):
    - 게시글을 작성하면서 사진 파일들 5장을 한 화면에서 한 번에 업로드하는 상황이다.
    - 이 옵션이 켜져 있으면, 개발자는 부모인 Post만 em.persist(post)로 저장해 주면 된다. JPA가 List<Image>안에 들어있는 자식 이미지 객체들까지 자동으로 추적해서 대신 persist해준다.
- `CascadeType.REMOVE` (함께 삭제):
    - 부모인 게시글이 삭제될 때, 그 글에 종속되어 있던 첨부 이미지 레코드들도 DB에서 같이 지워지도록 한다.

### Cascade의 절대 조건

잘못 쓰면 데이터가 연쇄 파쇄된다. 아래 조건을 만족할 때만 제한적으로 사용한다.

- 자식 객체를 관리하는 부모가 오직 단 하나(단일 소유자)일 때만 사용한다.
- 안전한 케이스(Post : Image) : 첨부 이미지는 오직 하나의 게시글 안에서만 의미가 있고 다른 클래스는 이 이미지를 알 필요가 없다. 완전히 종속적이므로 Cascade를 걸어도 안전하다.
- 위험한 케이스(User : Post) : 특정 회원이 탈퇴한다고 해서 그 회원이 과거에 썼던 모든 게시글을 시스템에서 연쇄 삭제하면 어떻게 될까?
    - 다른 유저들이 그 글을 북마크했거나 댓글을 달아두었다면 연쇄적으로 정합성이 다 깨져버린다. 게시글은 회원 외에도 댓글, 좋아요 등 다른 도메인과 엮여있는 독립적인 엔티티이기 때문에 절대 Cascade를 걸면 안 된다.

## Orphan Removal(고아 객체 제거)

부모 자체는 멀쩡히 살아있는데, 부모가 가진 컬렉션에서 연관 관계가 끊어진 자식 엔티티를 고아로 판단하고 DB에서 자동으로 DELETE쿼리를 날려 삭제하는 기능

**`CascadeType.REMOVE`와의 차이점:**

- `Cascade REMOVE`는 부모가 사라지면 자식도 사라진다.
- `orphanRemoval = true`는 부모는 살아있으나, 부모가 자식을 바구니에서 빼버렸을 때 자식을 DB에서 삭제하는 구조다.

**게시글 수정 - 사용자가 게시글 수정 페이지에서 기존에 올렸던 이미지 3장 중 필요 없어진 이미지 1장의 버튼을 지우는 상황이다.**

```java
@Entity
public class Post {
    @OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Image> images = new ArrayList<>();

    public void removeImage(Image image) {
        this.images.remove(image); // 1. 부모 바구니에서 자식을 제거
        image.setPost(null);       // 2. 연관관계 연결 끊기
    }
}
```

- 옵션이 꺼져있을 때 : 자바 컬렉션에서 이미지를 뺀다. 플러시 시점에 JPA는 단지 이미지 테이블의 외래키 값만 UPDATE image SET post_id = null로 바꾼다.
- 옵션이 켜져있을 떄 : JPA가 컬렉션에서 누락된 이미지 객체를 발견하고 고아객체라고 판단해서 트랜잭션이 커밋되는 플러시 시점에 DB에 물리적인 DELETE FROM image WHERE id = ? 쿼리를 날려서 청소해준다.
    
    → 이 기능 역시 이미지처럼 단일 부모에게만 종속된 자식일 때만 켜야 안전하다.
