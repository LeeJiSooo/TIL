## 가져가야 할 핵심

1. SQL은 데이터베이스에게 무엇을 원하는지 말하는 구조화된 질의 언어이다.
2. DDL(정의), DML(조작), DCL(제어)의 명확한 역할 차이를 구분할 수 있어야 한다.
3. 작성한 쿼리가 데이터베이스 성능과 서비스 안정성에 어떤 심각한 영향을 미치는지 파악할 수 있어야 한다.

현재 백엔드 개발은 직접 SQL을 짜기보다 ORM(JPA 등)을 이용해 자바 문법으로 데이터를 다루고, 복잡한 쿼리는 AI의 도움을 받아 작성하는 경우가 많다. 그럼에도 SQL을 깊이 이해해야 하는 이유는 다음과 같다.

 

- 성능 : ORM이 생성해 준 쿼리가 언제나 최적화되어 있지는 않다. 이 때문에 어떤 기업은 코드 리뷰 올릴 때 ORM이 생성한 실제 SQL구문을 무조건 첨부하게 하여 성능 문제를 검증한다. 개발자는 이를 보고 성능상 문제를 짚어내고 해결할 수 있는 판단력이 있어야한다.
- 복잡한 작업 및 마이그레이션 : 복잡한 조건의 대용량 조회, 데이터를 한 번에 밀어 넣는 작업, 스키마 마이그레이션 등은 직접 SQL을 작성해야 한다.
- 최종 책임은 개발자의 몫 : 데이터 관련 작업은 한 번의 실수가 서비스 전체의 문제(데이터 유실 및 장애)로 이어진다. AI가 검수해 주더라도 결국 책임을 지는 것은 개발자이기 때문에 SQL을 볼 줄 알아야 한다.

#### 선언형 언어로서의 SQL

- SQL은 대표적인 선언형 언어이다. 자바나 파이썬 같은 대중적인 언어는 개발자가 작업의 순서와 방법을 하나하나 코드로 명시하는 ‘명령형’구조이다.
- 반면 SQL은 “무엇을 원하는지”만 데이터베이스에 선언한다.
- “회원 테이블에서 서울에 사는 20대 사용자를 찾아줘”라고 조건만 명시하면, 어떻게 효율적으로 가져올지는 DBMS내부의 옵티마이저가 실행계획을 세워(인덱스 탈지 등)최적의 방법으로 결과를 가져다준다.

## DML(Data Defination Language) - 데이터 정의어

데이터베이스의 구조(테이블, 스키마)를 생성, 수정, 삭제하는 언어이다.

### 테이블 추가 : CREATE TABLE

```jsx
CREATE TABLE users (
  id INT(6) UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(30) NOT NULL,
  email VARCHAR(30) NOT NULL,
  phone VARCHAR(50),
  gender VARCHAR(30),
  age INT CHECK (age > 0),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NULL,
  deleted_at TIMESTAMP NULL
);
```

- `id`: 컬럼 이름. `INT(6)`는 6자리 길이의 정수형 타입. `UNSIGNED`는 양수만 저장하겠다는 뜻.
- `AUTO_INCREMENT`: 데이터가 추가될 때마다 자동으로 1씩 증가합니다.
- `PRIMARY KEY`: 기본키 지정 (중복 불가, 비어있기 불가 - `NOT NULL` & `UNIQUE`).
- `NOT NULL`: 해당 컬럼에 `NULL` 입력을 허용하지 않는 제약조건.
- `CHECK (age > 0)`: 데이터가 특정 조건을 만족하는지 DB 레벨에서 검사하는 규칙 (어플리케이션에서도 검사하지만 DB에서도 다중 방어선으로 구축, 필수는 아님).
- `DEFAULT CURRENT_TIMESTAMP`: 데이터 삽입 시 현재 시간이 자동으로 입력됨.

### 테이블 수정 : ALTER TABLE

```jsx
ALTER TABLE users
ADD gender VARCHAR(30) AFTER phone,  -- phone 컬럼 뒤에 gender 컬럼 추가
MODIFY email VARCHAR(100),           -- email 컬럼의 데이터 타입을 100자로 확장
DROP age;                            -- age 컬럼을 테이블에서 완전 삭제
```

**운영 DB에서 ALTER TABLE 변경 시 대형 장애 위험**

- Table Lock : 이미 수백만 건의 데이터가 있고 사용자가 활발히 이용 중인 서비스의 운영DB에 ALTER명령을 날리면, DB가 데이터를 정리하는 동안 테이블 전체에 락이 걸려버린다. 이로 인해 사용자는 갑자기 로그인이 안되거나, 주문 버튼을 눌렀을 때 시스템이 멈춘 것처럼 느끼게 된다.

→ 운영 중인 서비스에서는 매우 신중해야하므로 `pt-online-schema-change` 같은 특수 도구나 `Flyway` 같은 마이그레이션 툴을 사용해 점검 시간 또는 무중단으로 스크마 마이그레이션을 진행한다.

- 무중단 배포 기법(Expand-Contract 배포):
    - Expand(확장)단계 : 기존 기능이 깨지지 않도록 NULL을 허용하거나 기본 값을 저장하여 새 컬럼만을 추가한다. 이후 어플리케이션 코드를 배포해 새 컬럼에 값을 쓰기 시작하고, 필요시 과거 데이터도 백그라운드에서 천천이 채운다.
    - Contract(축소)단계 : 데이터가 안정화되고 나면, 더 이상 사용하지 않는 과거 컬럼을 제거하거나 관련된 레거시 코드를 정리한다.

## DDL(Data Manipulation Language) - 데이터 조작어

테이블 내부의 실제 데이터를 삽입, 조회, 수정, 삭제하는 언어이다.

### 데이터 삽입 : INSERT INTO

```jsx
INSERT INTO users (name, age, email) VALUES 
('John Doe', 25, 'johndoe@example.com'),
('Jane Smith', 30, 'janesmith@example.com');
```

### 데이터 조회 : SELECT 기초 및 안티패턴 위험성

```jsx
SELECT * FROM users;
```

절대 하면 안되는 실수(`SELECT *` 아스테리스크) :

- 성능 저하 : 쓰지 않는 불필요한 데이터까지 모두 퍼오기 때문에 네트워크 대역폭 낭비 및 메모리 사용량이 급증한다.
- 유지보수 어려움 : 추후 DB에 새 컬럼이 추가되면, 어플리케이션 레이어에서 예상치 못한 데이터 구조가 들어와 에러를 발생할 수 있다.
- 가독성 저하 : 쿼리문만 보고는 어플리케이션이 정확히 어떤 컬럼을 가져다 쓰려는지 명확히 파악할 수 없다.

#### 다양한 조건 검색(WHERE절)

```jsx
-- age가 25 초과인 레코드
SELECT id, name FROM users WHERE age > 25;

-- age가 10 이상 25 이하 범위 내 레코드
SELECT id, name FROM users WHERE age BETWEEN 10 AND 25;

-- age가 동시에 10이면서 25인 레코드 (논리적으로 아무것도 조회되지 않음)
SELECT id, name FROM users WHERE age = 10 AND age = 25;

-- age가 10이거나 25인 레코드
SELECT id, name FROM users WHERE age = 10 OR age = 25;

-- age가 25가 아닌 모든 레코드
SELECT id, name FROM users WHERE NOT age = 25;
```

#### IN / NOT IN절의 성능 안티패턴

```jsx
SELECT id, name FROM users WHERE age IN (10, 26);
```

`IN` 절 내부에 들어가는 파라미터 배열의 크기가 비대하게 커지면 성능이 크게 떨어진다. `NOT IN` 또한 옵티마이저가 인덱스를 타지 못하게 방해하는 성능 저하의 주범 중 하나이다. 이런 문제가 생기면 쿼리를 여러 개로 쪼개거나 어플리케이션 레이어로 로직을 이관하여 해결해야 한다.

#### 문자열 검색(LIKE절)과 인덱스 이슈

```jsx
-- 'Jane'으로 시작하는 모든 레코드 (글자 수 제한 없음)
SELECT id, name FROM users WHERE name LIKE 'Jane%';

-- 'Jane'으로 시작하고 뒤에 정확히 5개의 임의 문자가 오는 레코드
SELECT id, name FROM users WHERE name LIKE 'Jane _____';

-- 성능 위험: 'Smith'로 끝나는 레코드
SELECT id, name FROM users WHERE name LIKE '%Smith';
```

**성능 이슈:** `LIKE '%검색어'`처럼 **와일드카드(`%`)가 문자열 앞에 붙으면 DB는 인덱스를 전혀 사용하지 못하고 Full Table Scan을 타게 되어 성능이 폭락한다.** 현업에서 이런 텍스트 검색 쿼리가 잦다면 일반 DB 대신 `ElasticSearch`를 도입하거나 `Full Text Index`를 셋팅해야 한다.

### 데이터 수정 : UPDATE

```jsx
-- 대형 사고 유발 쿼리 (WHERE 절 없음!)
UPDATE users SET email = 'newemail@example.com';
```

**주의:** `WHERE` 절을 빼먹고 `UPDATE`를 실행하면 **테이블 내 전체 유저의 이메일이 단 하나의 값으로 동기화**되는 대참사가 일어난다.

```jsx
-- 안전한 수정 방식
UPDATE users SET email = 'contact@startupcode.kr' WHERE id = 1;
```

### 데이터 삭제 : DELETE

```jsx
DELETE FROM users WHERE id = 2;
```

## 정렬, 페이징 및 그룹화(성능 고도화 관점)

### 정렬(ORDER BY)

```jsx
SELECT id, age FROM users ORDER BY age DESC; -- 내림차순 (큰 값부터)
SELECT id, age FROM users ORDER BY age ASC;  -- 오름차순 (작은 값부터)
```

**성능 이슈:** 정렬 작업은 데이터가 많아질수록 서버 비용(CPU, 메모리)이 기하급수적으로 커진다. `ORDER BY`에 사용한 컬럼 순서와 DB의 `INDEX` 생성 순서가 일치하지 않으면, DB는 디스크나 메모리에 임시 공간을 만들어 직접 데이터를 정렬해야 하므로 성능이 느려진다.자주 호출되는 API라면 반드시 실행 계획(`EXPLAIN`)을 확인하여 정렬 비용을 체크해야 한다.

### 페이징(LMIT, OPSET)

```jsx
SELECT id, name FROM users ORDER BY id ASC LIMIT 10 OFFSET 20;
```

앞의 20개 데이터를 건너뛰고(`OFFSET 20`), 그 다음부터 최대 10개(`LIMIT 10`)의 데이터(즉, 21번째~30번째 레코드)를 가져온다. 게시판 3페이지를 구현할 때 주로 쓰인다.

- **OFFSET의 성능 한계:** `OFFSET` 값이 수십만, 수백만에 달할 정도로 커지면(예: `OFFSET 1000000`), DB는 **실제로 앞의 100만 건 데이터를 엔진에서 전부 읽은 후 그냥 버리는 무식한 작업**을 수행한다. 이 때문에 뒤 페이지로 갈수록 속도가 느려진다.
- **해결책:** 현업에서는 대용량 데이터 조회 시 `OFFSET`을 쓰지 않고, 직전 페이지의 마지막 ID 값을 조건문으로 걸어 타는 **커서 기반 페이징** 기술을 도입한다. 이 경우에도 정렬 기준이 항상 명확해야 한다.

### 그룹화 및 집계(GROUP BY, HAVING)

```jsx
-- 나이별로 그룹화하여 각 그룹에 속한 사용자 수 계산
SELECT age, COUNT(*) AS user_count FROM users GROUP BY age;

-- 그룹화 후, 나이가 30세 이상인 그룹만 필터링 (HAVING)
SELECT age, COUNT(*) AS user_count FROM users GROUP BY age HAVING age >= 30;
```

특정 컬럼의 값이 같은 데이터끼리 묶어 `COUNT()`, `SUM()`, `AVG()` 등의 집계 함수를 적용한다.

- **성능 이슈:** 데이터 양이 많을 때 무작정 그룹화를 때리면 대량의 임시 테이블 작업이 수반되어 매우 느려진다. 디폴트 전략은 **`WHERE` 절을 먼저 사용하여 그룹화할 대상 행의 모수를 최대한 줄인 후 `GROUP BY`를 수행**하는 것이다.
- 매번 호출되는 API에 전체 테이블 대상의 `COUNT()`나 중복 제거를 하는 `DISTINCT`가 섞여 있다면 시스템 리소스를 갉아먹으므로 성능 검증이 절대적이다.

## DCL(Data Control Languae) - 데이터 제어어

데이터베이스에 접근하고 명령을 수행할 수 있는 사용자 계정 및 접근 권한을 관리하는 언어이다. 사용자에게 권한을 부여하는 GRANT, 권한을 박탈하는 REVOKE가 있다.

- 권한 분리의 필요성 : 하나의 DB시스템 내에 여러 계정을 생성하고 직무에 따라 권한을 철저히 다르게 부여해야 보안과 무결성이 유지된다.
    - 기획자/데이터 분석 엔지니어 : 데이터를 함부로 저작하거나 테이블을 깨뜨리면 안 되므로 오직 조회 권한만 부여한다.
    - 백엔드 개발자 :회사 컨벤션마다 다르지만, 보통 로컬/개발 DB에는 모든 권한을 주더라도 실제 운영 DB에는 오직 읽기 권한만 부여하는 경우가 많다. 운영 데이터 수정이나 스키마 변경이 필요할 땐 Jira티켓 등을 발행해 인프라 부서에 공식 요청한다.
    

## 정리

- DDL로 테이블 구조를 디자인 및 빌드하고, DML로 그 구조 안에 데이터를 넣고, 빼고, 고치며, DCL로 그 데이터에 접근할 수 있는 사람들의 출입 계정과 권한을 철저히 통제한다.
