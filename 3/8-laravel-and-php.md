# 섹션 8 - Laravel & PHP 도커화 프로젝트

## Laravel & PHP 앱에 대한 간단한 설명
![캡처](https://github.com/user-attachments/assets/e3f6ac10-8b3c-4690-8009-781574c034c0)
- Application Container
  - MySQL Database
  - PHP Interpreter
  - Nginx Web Server
- Utility Container
  - npm: 반환하는 뷰에 자바스크립트 사용하는 경우 
  - Laravel Artisan: 데이터베이스 초기 데이터 관련 설정
  - Composer: Laravel 애플리케이션을 생성, 써드파티 패키지 관리

## 도커화
0. docker-compose.yaml에 services에 같은 indent로 컨테이너를 선언하면 굳이 네트워클 설정을 하지 않아도, 컨테이너들끼리 기본적으로 같은 네트워크를 사용한다.
1.  Nginx 컨테이너 추가
    - Web Server 역할
    - 요청 처리 설정에 대한 Nginx conf 파일 (포트, 리다이렉션 등)을 생성
      ```
      server {
          listen 80;
          index index.php index.html;
          server_name localhost;
          root /var/www/html/public;
          location / {
              try_files $uri $uri/ /index.php?$query_string;
          }
          location ~ \.php$ {
              try_files $uri =404;
              fastcgi_split_path_info ^(.+\.php)(/.+)$;
              fastcgi_pass php:9000;  # 같은 네트워크 안 컨테이너 이름 활용  #php 9000포트와 네트워크 연결
              fastcgi_index index.php;
              include fastcgi_params;
              fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
              fastcgi_param PATH_INFO $fastcgi_path_info;
          }
      }
      ```
    - docker-compose.yaml 파일에 추가해야 하는 사항
      - image: 도커 허브에서 Nginx 이미지 가져오기
      - ports: Nginx의 기본 포트 80을 로컬 호스트 머신의 포트와 연결
      - volumes:
        - (바인드 마운트1): 웹 서버로 들어오는 요청은 PHP 컨테이너에 넘겨서 처리해야 하니까 PHP 소스 코드(로컬 호스트 머신에도 있는)가 웹 서버에도 노출되어야 함
        - (바인드 마운트2): 요청 처리 설정에 대한 Nginx conf 파일 (포트, 리다이렉션 등)을 컨테이너 내부에 마운트
    - 배포 시에는 Nginx conf 파일을 컨테이너 내부에 복사하는 방식으로 커스텀 Nginx 이미지를 이용하는 방식을 사용 
      ```
      version: '3.8'
      
      services:
        server:
          image: 'nginx:stable-alpine'
          # build:
          #   context: .
          #   dockerfile: dockerfiles/nginx.dockerfile
          ports:
            - '8000:80'
          volumes:  # 요청을 php 컨테이너로 전달하기 위해 필요 
            - ./src:/var/www/html  # 웹 서버로 들어오는 PHP 요청은 php 인터프리터에 넘겨서 처리해야 하니까 php파일이 서버에도 노출되어야 함 nginx.conf에서 root 부분에서도 여기를 참조함
            - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro  # 특정한 하나의 파일을 컨테이너에 복사하고자 한다면 이렇게 지정가능
      ```
      
2.  PHP 컨테이너 추가
    - PHP Interpreter 역할
    - PHP dockerfile을 생성
      - PHP에는 실행되는 디폴트 명령어가 존재하므로 끝에 CMD, ENTRYPOINT 등은 작성하지 않아도 된다. -> 디폴트 명령을 사용
      ```
      FROM php:8.2-fpm-alpine
       
      WORKDIR /var/www/html
      
      RUN docker-php-ext-install pdo pdo_mysql
      ```
    - docker-compose.yaml 파일에 추가해야 하는 사항
      - build: 커스텀 PHP 이미지 가져오기
      - volumes: (바인드 마운트1) 로컬 호스트 머신에 있는 PHP 소스 코드를 PHP 컨테이너에도 연결
        - delegated 옵션을 사용하면 컨테이너가 데이터를 기록할 때 그 결과를 호스트 머신에 즉시 반영하지 않고 Batch로 처리 -> 속도 향상, 물론 안정성은 저하
      - ports: PHP의 기본 포트 9000을 로컬 호스트 머신 포트에 연결
      ```
      php:  # 커스텀 이미지 사용
        build: 
          context: ./dockerfiles
          dockerfile: php.dockerfile
        volumes:
          - ./src:/var/www/html:delegated    #컨테이너가 데이터를 기록해야 할 때 그 결과를 호스트 머신에 즉시 반영하지 않고 배치로 처리 -> 안정성은 떨어지지만 속도 향상
          # laravel 프레임워크 실행할 때 특정 파일(뷰) 생성하기에 ro 모드로 안 한다.
        ports:
          - '3000:9000'  # 로컬 호스트 머신 포트: php 컨테이너 포트
      ```
        
3.  MySQL 컨테이너 추가
    - MySQL 환경변수 파일 생성
      - 데이터베이스 이름, 유저 이름, 비밀번호 등 환경변수 설정
      ```
      MYSQL_DATABASE=homestead
      MYSQL_USER=homestead
      MYSQL_PASSWORD=secret
      MYSQL_ROOT_PASSWORD=secret
      ```
    - docker-compose.yaml 파일에 추가해야 하는 사항
      - image: 도커 허브에서 MySQL 이미지 가져오기
      - env_file: 로컬 호스트 머신에 있는 MySQL 환경변수 파일 위치 기재
      
4.  Composer 유틸리티 컨테이너 추가
    - Laravel 애플리케이션을 생성, 써드파티 패키지 관리
    - Composer dockerfile 생성
      ```
      FROM composer:latest
      
      WORKDIR /var/www/html  
      # Laravel 앱이 생성될 루트 폴더
      
      ENTRYPOINT ["composer", "--ignore-platform-reqs" ] 
      ```
    - docker-compose.yaml 파일에 추가해야 하는 사항
      - build: 커스텀 Composer 이미지 가져오기
      - volumes: (바인드 마운트1) 로컬 호스트 머신에 있는 PHP 소스 코드를 Composer 컨테이너에도 연결 
      ```
      composer:  # Laravel 애플리케이션 생성
          build:
            context: ./dockerfiles
            dockerfile: composer.dockerfile
          volumes:
            - ./src:/var/www/html  # 소스 폴더를 컨테이너 내부의 /var/www/html 폴더에 바인딩 -> composer를 사용하여 컨테이너 내부의 이 폴더에서 Laravel 애플리케이션을 생성한다면 그러면 로컬 호스트 머신의 소스 폴더로 다시 미러링됨
      ```
5.  Composer 유틸리티 컨테이너로 Laravel 앱 만들기
      ```
      docker-compose run --rm composer create-project --prefer-dist laravel/laravel .
      # composer create-project 이지만 composer dockerfile에 CMD로 composer을 추가했으니 생략
      # 프로젝트가 생성될 폴더를 마지막에 . 표시 
      # . 은 컨테이너 내부의 var/www/html 
      ```
  
7.  그외 유틸리티 컨테이너 추가 
