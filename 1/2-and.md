---
description: Docker에서 컨테이너와 이미지를 효율적으로 관리하는 방법을 실습해보겠습니다.
---

# 섹션 2 - 이미지 & 컨테이너 관리

### **1. 컨테이너 및 이미지 관리 개요**

Docker는 **이미지와 컨테이너**를 기반으로 동작하는 가상화 플랫폼입니다.

* **이미지(Image)**: 컨테이너 실행을 위한 템플릿
* **컨테이너(Container)**: 이미지를 실행한 독립적인 환경

Docker를 효율적으로 사용하기 위해 **자주 사용되는 명령어 및 개념**을 정리하겠습니다.

***

### **2. 컨테이너 목록 조회 및 상태 확인**

```
docker ps          # 실행 중인 컨테이너 목록 확인
docker ps -a       # 중지된 컨테이너 포함 모든 컨테이너 목록 조회
docker ps --help   # ps 명령어 옵션 확인

```

***

### **3. 컨테이너 실행 모드**

#### **📌 Attached 모드 (기본)**

컨테이너 실행 결과가 터미널에 직접 출력됨. (포그라운드 실행)

```
docker run ubuntu  # 기본 실행 (터미널에 출력 연결)
docker run -it ubuntu /bin/bash  # 쉘 실행 후 직접 명령어 입력 가능

```

#### **📌 Detached 모드 (백그라운드 실행)**

컨테이너를 백그라운드에서 실행하여 터미널과 분리됨.

```
docker run -p 3000:80 -d nginx  # 백그라운드에서 실행
docker start CONTAINER_NAME      # 중지된 컨테이너를 백그라운드에서 실행

```

|              | **Attached Mode (포그라운드 모드)**      | **Detached Mode (백그라운드 모드)** |
| ------------ | --------------------------------- | ---------------------------- |
| 실행 방식        | 터미널과 연결됨                          | 백그라운드에서 실행됨                  |
| 터미널 사용 가능 여부 | 컨테이너가 종료되면 터미널도 종료됨               | 터미널이 종료되어도 컨테이너 계속 실행        |
| 실행 후 로그 확인   | 바로 확인 가능                          | `docker logs`로 확인해야 함        |
| 실무 활용        | 실시간 디버깅, 로그 확인, 테스트               | 장기 실행 서비스(Nginx, DB 등) 운영    |
| 실행 예제        | `docker run -it ubuntu /bin/bash` | `docker run -d nginx`        |

***

### **4. 실행 중인 컨테이너에 다시 연결**

```
docker attach CONTAINER_NAME  # 실행 중인 컨테이너에 연결 (출력 확인 및 조작 가능)
docker start -a CONTAINER_NAME  # 중지된 컨테이너를 다시 실행하면서 출력 확인

```

***

### **5. 컨테이너 로그 확인**

```
docker logs CONTAINER_NAME    # 컨테이너 실행 로그 확인
docker logs -f CONTAINER_NAME # 실시간 로그 확인

```

***

### **6. 컨테이너 삭제**

```
docker rm CONTAINER_NAME  # 컨테이너 삭제
docker rm -f CONTAINER_NAME  # 강제 삭제 (실행 중인 컨테이너 포함)

```

⚠ **실행 중인 컨테이너는 삭제할 수 없음! (`docker stop` 후 삭제 필요)**

***

### **7. 이미지 삭제 및 정리**

```
docker rmi IMAGE_ID  # 특정 이미지 삭제
docker image prune   # 사용되지 않는 모든 이미지 삭제

```

⚠ **컨테이너가 실행 중이면 해당 이미지 삭제 불가능** (컨테이너 먼저 삭제 필요)

***

### **8. 컨테이너 자동 제거 실행**

컨테이너가 중지되면 자동 삭제됨.

```
docker run -p 3000:80 -d --rm IMAGE_ID

```

***

### **9. 이미지 정보 조회**

```
docker image inspect IMAGE_ID  # 이미지의 상세 정보 조회

```

📌 **확인 가능한 정보**

* 이미지 전체 ID
* 생성 날짜
* 노출되는 포트
* 자동 설정 환경 변수
* 사용 중인 OS 등

***

### **10. 컨테이너와 로컬 파일 시스템 간 파일 복사**

```
docker cp dummy/. CONTAINER_NAME:/test  # 로컬 → 컨테이너 복사
docker cp CONTAINER_NAME:/test dummy  # 컨테이너 → 로컬 복사

```

⚠ **실행 중인 파일을 변경하는 것은 오류 발생 가능성이 있음**

***

### **11. 컨테이너 및 이미지 네이밍**

```
docker run -p 3000:80 --rm --name mycontainer IMAGE_ID  # 컨테이너 이름 지정
docker build -t goals:latest .  # 빌드 시 이미지에 태그 부여

```

***

### **12. Docker 이미지 공유 방법**

#### **📌 1. Dockerfile 공유**

* 소스 코드와 함께 `Dockerfile`을 제공하여 직접 빌드 가능.

#### **📌 2. 빌드된 이미지 공유**

* 이미 완성된 이미지를 공유하는 방법 (`docker push` 이용).

```
docker tag LOCAL_IMAGE_NAME REPOSITORY:TAG  # 태그 추가
docker push REPOSITORY:TAG  # 원격 저장소에 업로드

```

📌 **공유 가능한 저장소**

1. **Docker Hub (공개 저장소)**
2. **Private Registry (사설 저장소, 예: AWS ECR, GCP Artifact Registry)**

***

### **13. Docker Hub 활용**

```
docker login  # 로그인
docker push myrepo/myimage:latest  # 이미지 업로드
docker pull myrepo/myimage:latest  # 이미지 다운로드

```

⚠ `docker run`은 로컬에 이미지가 있으면 기존 버전을 사용하므로, 최신 업데이트를 반영하려면 `docker pull` 먼저 실행 필요!

***

### 14. 실습 코드

[nodejs-app-finished.zip](attachment:dcd25960-05a3-4c87-b409-13a0c356da8f:nodejs-app-finished.zip)

[python-app-finished.zip](attachment:b58243ac-a1d2-43dc-a396-66bd0cb7c33f:python-app-finished.zip)
