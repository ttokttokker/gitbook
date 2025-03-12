# 섹션 2 - Docker 이미지 & 컨테이너

1.  Docker

    1. 컨테이너 기술: 컨테이너를 생성하고 관리하는 도구
    2. 필수 준비물: 리눅스,  도커 엔진, 컨테이너, 이미지


2. Container
   1. **이미지의 실행 인스턴스 (애플리케이션)**&#x20;
   2. 즉, 이미지를 기반으로 컨테이너를 실행시키는 것
   3. 장점
      1. 이미지만 있으면 어떤 컴퓨에서든지 독립적인 환경으로 실행할 수 있음
      2. 다음과 같은 경우에 용이함
         1. 개발/배포 환경이 다른 경우
         2. 모든 팀원이 같은 환경에서 같은 프로젝트 작업을 하는 경우
         3. 환경이 다른 프로젝트들을 작업해야 하는 경우
         4. 환경: runtimes, languages, frameworks, libraries

| Virtual Machine를 사용한 캡슐화                  | Container                             |
| ----------------------------------------- | ------------------------------------- |
| 각 VM에 OS 설치 필수 (같은 OS를 쓴다고 해도 다 설치해줘야 함)  | 내 컴퓨터에 리눅스만 있으면 각 컨테이너에 OS 설치할 필요 없음  |
| 내 컴퓨터 리소스를 크게 허비                          | 내 컴퓨터 리소스를 적게 허비                      |
| whole machine을 캡슐화                        | 환경 / 코드만 캡슐화                          |



3. Image
   1. 컨테이너 실행을 위한 탬플릿
   2. 앞서 말한 환경(runtimes, languages, frameworks, libraries) + code으로 구성
   3. read-only: 한번 이미지가 생성되면 변경될 수 없음
   4. Layer-based: 이미지는  다수의 레이어로 구성되어 있음



4. Layer 기반 아키텍처
   1. Layer: 각 레이어는 하나의 변경 사항을 담고 있으며, 파일 시스템에서 파일을 추가하거나 제거하거나 수정하는 작업을 수행
      1. 각 레이어는 이전 레이어에 대한 변경 사항을 포함한다.
   2. Dockerfile에서 모든 명령문은 레이어를 나타내고, 각 레이어는 캐쉬된다.
   3. 명령문(레이어)의 실행 결과가 달라지면 다시 실행한다.
      1. 즉, 코드를 수정해서 다시 이미지를 빌드한다면 아래 명령문의 실행 결과는 이전과 달라질 것이다. 따라서 다시 이미지 빌드할 때 아래 명령문의 이전 캐쉬는 적용되지 않는다. 이 내용을 고려하여 최적화가 가능하다.&#x20;
      2. 최적화: 캐쉬된 걸 사용해도 되는 명령문은 레이어의 실행 결과가 달라지는 부분 이전으로 배치

<pre class="language-docker"><code class="lang-docker"><strong>#레이어의 실행 결과가 달라지는 부분 이전 명령어는 캐쉬된 걸 사용
</strong><strong>
</strong><strong>#레이어의 실행 결과가 달라지는 부분
</strong><strong>COPY . /app  
</strong><strong>
</strong><strong>#레이어의 실행 결과가 달라지는 부분 이후 명령어는 다시 실행
</strong></code></pre>



5. Docker 사용터미널 명령어 정리

```
docker build .   //이미지 생성

docker ps   //현재 실행 중인 컨테이너 출력

docker ps -a   //모든 컨테이너 출력

docker run {imageId}   //해당 이미지로 컨테이너 실행

docker run -p {localPort}:{containerPort} {imageId}  //해당 컨테이너 포트로 수신되는 것을 로컬 컴퓨터의 포트로 수신할 수 있게 컨테이너 실행

docker stop {containerName}   //현재 실행 중인 컨테이너 중지
```



6. Dockerfile 명령문 정리

```docker
#baseImage를 설정, 도커허브에서 받아온 node 이미지
FROM node

#도커의 기본 작업 디렉토리는 컨테이너 파일 시스템의 루트 폴더
#아래 명령문을 통해 도커의 작업 디렉토리를 설정
#도커에게 모든 후속 명령이 /app 내부에서 실행될 것임을 알림
WORKDIR /app    

#현재 이 dockerfile이 있는 디렉토리의 package.json을 이미지/컨테이너 파일 시스템의 /app 디렉토리로 복사
COPY package.json /app

#패키지를 설치하겠다는 명령어를 실행
RUN npm install

#현재 이 dockerfile이 있는 디렉토리의 모든 파일을 이미지/컨테이너 파일 시스템의 /app 디렉토리로 복사
#절대경로로 명시해주는 것이 관행상 좋다
COPY . /app

#테이너의 프로세스가 이 포트를 개방할 것을 "문서화", 즉 기술적으로 하는 일은 없음 
EXPOSE 80

#"node" CMD로 설정 (첫번째 원소 - 특정 CMD 설정)
#"server.js"를 실행한다는 의미
CMD ["node", "server.js"]

#아래 구문은 이미지가 생성될 때 server.js를 실행시키는 것이기 때문에 틀린 명령문
#RUN node server.js
```

[https://docs.docker.com/get-started/docker-concepts/building-images/writing-a-dockerfile/](https://docs.docker.com/get-started/docker-concepts/building-images/writing-a-dockerfile/)
