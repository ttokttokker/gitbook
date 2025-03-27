# 섹션 6 - Docker Compose: 다중 컨테이너 오케스트레이션

## 1. Docker Compose란?

### ✅ 개요

Docker Compose는 다중 컨테이너 애플리케이션을 **단일 설정 파일(YAML)** 로 관리할 수 있게 해주는 도구입니다.

`docker run`, `docker build` 명령어를 반복하지 않고 **자동화된 오케스트레이션**을 제공합니다.

### ✅ 주요 특징

- 여러 Docker 명령어 → **1개의 `docker-compose.yaml`**
- 단일 명령어로 **모든 컨테이너 실행/중지**
- **버전 관리, 협업, 유지보수**에 탁월
- 개발환경에 **일관성** 제공

---

## 2. Docker 명령어 기반 수동 실행 방식 (Before Compose)

- docker-commands

  ```bash
  ---------------------
  Create Network
  ---------------------

  docker network create goals-net

  ---------------------
  Run MongoDB Container
  ---------------------

  docker run --name mongodb \
    -e MONGO_INITDB_ROOT_USERNAME=max \
    -e MONGO_INITDB_ROOT_PASSWORD=secret \
    -v data:/data/db \
    --rm \
    -d \
    --network goals-net \
    mongo

  ---------------------
  Build Node API Image
  ---------------------

  docker build -t goals-node .

  ---------------------
  Run Node API Container
  ---------------------

  docker run --name goals-backend \
    -e MONGODB_USERNAME=max \
    -e MONGODB_PASSWORD=secret \
    -v logs:/app/logs \
    -v /Users/maximilianschwarzmuller/development/teaching/udemy/docker-complete/backend:/app \
    -v /app/node_modules \
    --rm \
    -d \
    --network goals-net \
    -p 80:80 \
    goals-node

  ---------------------
  Build React SPA Image
  ---------------------

  docker build -t goals-react .

  ---------------------
  Run React SPA Container
  ---------------------

  docker run --name goals-frontend \
    -v /Users/maximilianschwarzmuller/development/teaching/udemy/docker-complete/frontend/src:/app/src \
    --rm \
    -d \
    -p 3000:3000 \
    -it \
    goals-react

  ---------------------
  Stop all Containers
  ---------------------

  docker stop mongodb goals-backend goals-frontend

  ```

---

## 3. Compose 설정 구조 및 작성법

### ✅ 기본 구성 키

| 키워드              | 설명                       |
| ------------------- | -------------------------- |
| `services`          | 실행할 각 컨테이너 정의    |
| `build`             | Dockerfile 경로 설정       |
| `image`             | 사용할 이미지 지정         |
| `ports`             | 포트 매핑                  |
| `volumes`           | 볼륨 마운트 지정           |
| `env_file`          | 환경변수 파일 지정         |
| `depends_on`        | 컨테이너 간 의존성 정의    |
| `stdin_open`, `tty` | 인터랙티브 모드 지원       |
| `container_name`    | 컨테이너 이름 지정(선택적) |

### a. **환경 변수 설정 방법**

1. `environment` 키로 직접 key-value 형태 지정

   ```yaml
   environment:
     MONGO_INITDB_ROOT_USERNAME: max
     MONGO_INITDB_ROOT_PASSWORD: secret
   ```

2. 또는 `.env` 파일을 만들어 `env_file`로 로딩

   ```yaml
   env_file:
     - ./env/mongo.env
   ```

### b. **네트워크 설정**

- Docker Compose는 **자동으로 단일 기본 네트워크**를 생성하고 모든 서비스들을 여기에 포함시킴
- 별도 네트워크 지정도 가능하지만 기본 설정이면 생략해도 문제 없음

### c. **볼륨 정의**

- `services:`와 **동등한 최상위 수준**에 `volumes:` 키로 명명된 볼륨을 정의해야 함
  ```yaml
  volumes:
    data:
  ```

### d. **도커 컴포즈 사용 시 유의사항**

- `-rm`, `d` 같은 플래그는 Compose에서 기본 적용됨 (자동 삭제 및 백그라운드 실행)
- Compose 파일에서는 `network`, `volume`, `env_file` 등의 항목을 **명시적으로 선언** 가능
- **바인드 마운트, 익명 볼륨**은 `volumes:`에 정의할 필요 없음

### e. **YAML 문법 팁**

- **들여쓰기는 반드시 공백 두 칸**
- `:`로 key-value 형식이면 없이 기술 가능
- 리스트 항목은 로 시작해야 함

---

## 4. `docker-compose.yaml` 예시 (MongoDB + Node + React)

```yaml
services:
  mongodb:
    image: mongo
    volumes:
      - data:/data/db
    env_file:
      - ./env/mongo.env

  backend:
    build: ./backend
    ports:
      - "80:80"
    volumes:
      - logs:/app/logs
      - ./backend:/app
      - /app/node_modules
    env_file:
      - ./env/backend.env
    depends_on:
      - mongodb

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    volumes:
      - ./frontend/src:/app/src
    stdin_open: true
    tty: true
    depends_on:
      - backend

volumes:
  data:
  logs:
```

---

## 5. `.env` 파일 예시

### 📄 `env/mongo.env`

```
MONGO_INITDB_ROOT_USERNAME=max
MONGO_INITDB_ROOT_PASSWORD=secret

```

### 📄 `env/backend.env`

```
MONGODB_USERNAME=max
MONGODB_PASSWORD=secret

```

---

## 6. 실행 및 정리 명령어

### ▶️ 실행

```bash
docker-compose up         # 로그 출력됨
docker-compose up -d      # 백그라운드 실행
docker-compose up --build # 코드 변경 시 이미지 강제 빌드

```

### ⏹ 종료

```bash
docker-compose down       # 컨테이너/네트워크 제거 (볼륨 유지)
docker-compose down -v    # + 볼륨까지 삭제

```

---

## 7. 기타 유용한 팁

### 🔍 컨테이너 이름 자동 생성 규칙

```
[프로젝트폴더명]_[서비스명]_1
예: docker-complete_backend_1

```

### 🏷 명시적 컨테이너 이름 지정

```yaml
container_name: mongodb
```

---

## 최종 요약

| 항목        | 설명                                                      |
| ----------- | --------------------------------------------------------- |
| 목적        | 멀티 컨테이너 앱을 자동화 및 일관성 있게 실행             |
| 사용 이유   | 명령 반복 제거, 유지보수 편리, 협업 간결                  |
| 주요 기능   | 이미지 빌드, 컨테이너 실행, 포트 매핑, 볼륨/네트워크 설정 |
| 추천 사용처 | 백엔드 + DB + 프론트로 구성된 현대 웹앱 개발 환경         |
