## 설명할 수 있어야 하는 것

- 도커 컴포즈가 무엇이고 왜 필요한지
- 파일 하나에 어떤 시스템 구조랑 관계를 표현하는지

## Dockerk Compose의 정의와 필요성

<img width="1106" height="502" alt="image" src="https://github.com/user-attachments/assets/be0205af-40ff-414a-8a53-ea14417eb9ab" />


여러 컨테이너로 구성된 애플리케이션의 전체 아키텍처(컨테이너, 포트, 네트워크, 볼륨 등)를 docker-compose.yml파일에 선언하고, 단일 명령어로 일괄 실행, 관리할 수 있도록 지원하는 Docker의 공식 도구

**왜 필요한가?(컴포즈가 없을 때의 고통)**

- 실제 웹 서비스는 컨테이너 1개로 끝나지 않음(최소 Nginx, Spring Boot, DB 등 3~4개 필요)
- 컴포즈가 없다면 : 각 컨테이너마다 긴 명령어를 일일이 순서에 맞춰 사람이 손으로 타이핑해야 한다.
- 문제점 : 포트 하나, 네트워크 이름 하나만 오타를 내도 컨테이너끼리 연결이 끊어지며, 이를 다른 사람이 같이 재현하기가 어렵다.

→ 전체 구성을 텍스트 파일(docker-compose.yml)로 작성해두면, 누구나 docker compose up 한줄로 동일한 멀티 컨테이너 환경을 복제 및 구동할 수 있다.

## docker-compose.yml파일 구조 및 작성법

```yaml
services:                                        # 최상위 블록 (앱을 구성하는 서비스들 나열)
  app:                                           # 1. 백엔드 서비스 정의 (스프링 부트)
    image: my-username/spring-app:latest         # 사용할 도커 이미지 지정
    depends_on:                                  # 실행 순서 관계 선언 (DB가 뜬 뒤 앱 실행)
      - db
    networks:                                    # 서비스가 상주할 가상 네트워크 선언
      - app-net

  proxy:                                         # 2. 리버스 프록시 서비스 정의 (Nginx)
    image: nginx:stable-alpine
    ports:                                       # 외부에 노출할 진입점 (호스트 80 ➔ 컨테이너 80)
      - "80:80"                                  
    depends_on:
      - app
    networks:
      - app-net

  db:                                            # 3. 데이터베이스 서비스 정의 (MySQL)
    image: mysql:8.0
    environment:                                 # DB 환경변수 설정
      MYSQL_ROOT_PASSWORD: secret
    volumes:                                     # 데이터 영속성을 위한 볼륨 마운트
      - db-data:/var/lib/mysql
    networks:
      - app-net

networks:                                        # 공유할 브릿지 네트워크 생성
  app-net:

volumes:                                         # 영구 보존할 도커 볼륨 생성
  db-data:
```

| **지시어** | **설명** |
| --- | --- |
| **`services:`** | 애플리케이션을 구성하는 실행 단위(컨테이너 역할)들을 모아두는 최상위 블록. |
| **`ports:`** | **호스트 포트 : 컨테이너 포트** 매핑. (외부 진입점인 `proxy` 서비스에만 열어둠) |
| **`depends_on:`** | 컨테이너 간의 **시작 순서(의존성)** 제어. (예: DB ➔ App ➔ Nginx 순서로 띄움) |
| **`networks:`** | 작성된 서비스들을 **동일한 사용자 정의 브릿지 네트워크**에 연결. (서로 서비스 이름인 `app`, `db` 등으로 자동 DNS 통신 가능) |
| **`volumes:`** | 컨테이너가 삭제되어도 DB 데이터를 영구히 보존하기 위한 **도커 볼륨 연동**. |

서비스란?

`services:` 아래의 `app`, `proxy`, `db` 같은 단위를 '서비스'라고 부름. 기본적으로는 서비스 1개당 컨테이너 1개가 생성되지만, 필요 시 서비스 정의 1개를 여러 개의 실행 인스턴스로 스케일링(늘리기)하는 구성을 뜻한다.

## Docker Compose의 작동 원리(선언적 방식)

docker-compose.yml파일 자체가 스스로 컨테이너를 구동하는 주체가 아니다.

```
[ 개발자 ] ─── (docker compose up 명령어 실행) ───► [ Docker Compose CLI ]
                                                           │
                                                           │ 1. yml 파일 해석 (원하는 상태 확인)
                                                           ▼
                                                    [ Docker Engine (dockerd) ]
                                                           │
                                                           │ 2. 도커 데몬이 실제 명령 수행
                                                           ▼
                                            [ 네트워크 생성 + 볼륨 연결 + 컨테이너 3개 동시 구동]
```

1. docker-compose.yml작성

원하는 최종 인프라 상태(어떤 이미지, 어떤 포트, 어떤 네트워크/볼륨을 갖출지)를 파일에 명시한다.

1. docker compose up실행

Docker Compose 도구가 YAML파일을 읽어 전체 프로젝트 구조를 파악한 후, 도커 데몬에게 실행을 지시한다.

1. 도커 엔진의 일괄 생성

도커 엔진이 자동으로 브릿지 네트워크를 뚫고, 볼륨을 할당하며, depends_on 순서에 맞춰 프로젝트 단위로 컨테이너들을 띄움

## Docker Compose vs Container Orchestration(Kubernetes)

- Docker Compose : 단일 서버(EC2 1대) 안에서 여러 컨테이너로 구성된 애플리케이션을 코드로 정의하고 관리하는 도구 → 개발 환경, 테스트 서버, 소규모 단일 서버 배포에 적합
- 컨테이너 오케스트레이션(Kubernetes / K8s) : 수십~수천 대의 여러 물리/가상 서버(클러스터)에 컨테이너들을 자동으로 분산 배치하고, 컨테이너가 죽으면 자동으로 살려내며, 트래픽에 따라 컨테이너 수를 자동으로 늘려주는(Auto-scaling)대규모 프로덕션 전용 도구

## 정리

- 도커 컴포즈는 여러 컨테이너로 이루어진 어플리케이션의 구성을 yml파일 하나에 선언하고 그 구성을 하나로 다루는 도구
- 선언적으로 작성 → 어떤 구성이 있어야하는지를 적어둔다. 실제로 컨테이너나 네트워크를 다루는 주체는 도커엔진이고 컴포즈는 그 파일을 읽어서 전달하는 역할이다.
- 파일의 중심이자 시작은 services이고 그 아래에 어떤 이미지에서 출발하는지 어떤 포트로 요청이 들어오는지, 어떤 네트워크 연결되는지, 의존관계가 있는지 작성한다.
    
    → 일관성과 재현 가능성을 얻는다.
