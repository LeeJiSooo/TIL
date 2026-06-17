## 가져가야 할 핵심

1. 외래키를 주 테이블에 두느냐, 대상 테이블에 두느냐에 따라 연관관계의 주인과 비즈니스 편의성이 달라진다.
2. 대상 테이블에 외래키를 둘 때 지연 로딩이 파괴되는 메커니즘을 프록시 생성 조건으로 이해한다.
3. 기획의 변경으로 인해 @OneToOne 설계를 기피하고 처음부터 1:N 구조로 열어두는 이유를 파악한다.

## 일대일(1:1)관계

- 관계의 정의
    - 하나의 엔티티가 다른 하나의 엔티티와 단 1:1로만 매핑되는 구조이다.(ex. [사용자 : 프로필 사진], [주문 : 결제 정보])
- 데이터베이스이 1:1 보장 메커니즘
    - 관계형 데이터베이스에는 테이블 간의 1:1 관계를 강제하기 위해, 외래키 컬럼을 생성한 뒤  그 컬럼에 유일성 제약조건을 걸어버린다.
    - 이렇게 하면 똑같은 외래키 값이 테이블에 두번 있을 수 없으므로 물리적으로 1:1관계가 보장된다.

## 주 테이블에 외래 키가 있는 경우

주로 많이 조회하는 주체인 User(주 테이블)가 외래키를 쥐고 관리한다.

### 1:1 단방향 매핑(주 테이블에 외래키)

```java
@Entity
public class User { // 주 테이블 역할을 하는 엔티티
    @Id @GeneratedValue
    private Long id;
    private String name;

    @OneToOne
    @JoinColumn(name = "profile_id", unique = true) // FK + UNIQUE 제약조건
    private UserProfile profile; // 연관관계의 주인
}

@Entity
public class UserProfile { // 대상 테이블
    @Id @GeneratedValue
    private Long id;
    private String profileImagePath;
}
```

User를 조회할 때 프로필 테이블을 조인하거나 확인하기 편해서 주 테이블 중심의 비즈니스 로직을 짜기 매끄럽다. user_profile 입장에서는 유저를 알 방법이 없지만, 보통 유저를 먼저 찾고 프로필을 꺼내 쓰는 도메인 흐름상 단방향으로도 충분히 구현이 가능하다.

### 1:1 양방향 매핑(주 테이블에 외래키)

만약 프로필 쪽에서도 역방향으로 사용자를 참조하고 싶다면, 반대편에 mappedBy를 붙여서 단방향 길을 하나 더 열어준다.

```java
@Entity
public class UserProfile {
    @Id @GeneratedValue
    private Long id;
    private String profileImagePath;

    @OneToOne(mappedBy = "profile") // 읽기 전용 (진짜 주인은 User.profile 필드다)
    private User user;
}
```

외래키를 쥔 User.profile이 여전히 주인이고, UserProfile.user는 읽기 전용이 되어 객체 그래프 탐색(profile.getUser())이 지원된다. 

## 대상 테이블에 외래 키가 있는 경우(지연로딩 파괴)

비즈니스 관점에 따라 User 테이블은 건드리지 않고, 대상 테이블인 UserProfile테이블 쪽에 user_id 외래키를 두는 구조이다.

```java
// 주 테이블 User (외래키 없음, 주인이 아님)
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;

    @OneToOne(mappedBy = "user") // 읽기 전용으로 밀려남
    private UserProfile profile;
}

// 대상 테이블 UserProfile (외래키를 가짐, 연관관계의 주인)
@Entity
public class UserProfile {
    @Id @GeneratedValue
    private Long id;

    @OneToOne
    @JoinColumn(name = "user_id", unique = true) // 물리 외래키 
    private User user;
}
```

### 왜 이 구조에서는 지연로딩이 작동하지 않고 즉시 로딩될까?

JPA가 어떤 엔티티를 조회할 때 연관 필드에 지연 로딩을 적용하려면 결정적인 조건이 필요하다.

- 이 필드에 담길 연관 객체가 ‘진짜 존재하는지(프록시)’ 아니면 ‘원래 없는(null) 건지’를 명확히 판별할 수 있어야 한다.
- 데이터가 없는 사용자에게 가짜 프록시 객체를 넣었다가, 나중에 개발자가 접근했을 때 에러가 나면 안 되니까 프로필이 아예 없는 유저라면 필드에 반드시 null을 주입해 줘야 한다.
- 주 테이블에 외래키가 있을 때(지연 로딩 성공)
    - User 테이블을 SELECT로 조회해 오는 순간에 내 테이블 안에 profile_id컬럼이 있다. 그 컬럼 값이 NULL이면 자바 필드에 그냥 null을 저장하고, 어떤 숫자 값이 채워져 있으면 값이 있다는 소리니까 가짜 프록시 객체를 채워 넣어서 지연 로딩을 할 수 있다.
- 대상 테이블에 외래키가 있을 때(지연 로딩 실패)
    - User테이블을 SELECT해봤자 내 테이블 안에는 프로필에 대한 컬럼이 없다. 이 사용자가 프로필을 등록한 상태인지 아닌지 자바 메모리 단에서는 알 방법이 없다. 결국 JPA는 이 필드에 프록시를 넣을지 null을 넣을지 판단하기 위해서, 어쩔 수 없이 건너편 UserProfile테이블에 직접 SELECT쿼리를 날려서 데이터가 있는지 조회해봐야만 한다.

→ 프로필이 있는지 확인해보려고 이미 DB로 SELECT쿼리를 날렸다. DB를 뒤져서 데이터를 진짜로 가져왔는데, 굳이 가짜 프록시 객체를 만들어서 넣어둘 이유가 전혀 없다. 그래서 JPA는 프록시를 안 만들고 진짜 객체를 채워버린다.

→ 즉, 개발자가  **@OneToOne(fetch = FetchType.LAZY)를 작성해봤자 JPA가 이를 구조적 한계 때문에 강제로 무시하고 즉시 로딩처럼 즉각 쿼리를 날리며 N+1문제를 유발한다.**

**Q. JPA에서 OneToOne 양방향 매핑할때 발생할 수 있는 성능상 이유가 무엇인가?**

대상 테이블에 외래키를 두는 구조로 매핑하면 문제가 생길 수 있다.

프록시 객체 생성 여부를 판단하기 위해서 강제로 대상테이블을 조회하게 되고 그러면 지연 로딩 설정이 무시되고 즉시 로딩처럼 동작해서 문제가 생긴다.

## 정리

- 테이블 간의 일대일을 강제하기 위해 DB영역에서는 외래키 컬럼에 유니크 제약조건을 반드시 걸어둔다.
- 주체가 되는 엔티티가 외래키를 권리할 때는 지연 로딩이 정상 작동한다.
- 대상 테이블에 외래키를 넣으면 안 된다. 주 테이블 입장에서 연관 데이터 존재 여부를 즉시 알 수 없어 무조건 대상 테이블을 먼저 조회하므로, 지연 로딩이 파괴되어 즉시 로딩으로 작동하는 한계가 존재한다.
