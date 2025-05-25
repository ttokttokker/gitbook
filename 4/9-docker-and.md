# 섹션 9 - 수동 배포에서 관리형 서비스로 ~ 안정적인 도메인을 위해 로드 밸런서 사용하기

### 수동 배포 vs. 관리형 서비스
- 수동 배포
  - 직접 관리할 수 있지만 그만큼 보안 등에 에대한 책임감도 필요
  - 또한 컨테이너를 생성, 관리, 업데이트하고 서버를 모니터링하고 관리해야 함
  - ex) AWS EC2
- 관리형 서비스
  -  생성, 업데이트, 모니터링, 스케일링 모든 것이 단순해짐
  -  단순히 배포할 거라면 이 서비스를 활용하는 것이 편함
  -  도커를 사용하는 것이 아니라, 이 클라우드의 서비스를 사용하는 것 (컨테이너 배포 및 실행이 더 이상 도커 명령으로 실행되지 않는다는 의미)
  - ex) AWS ECS
 
### AWS ECS
- Amazon Elastic Container Service (프리티어 X)
- Udemy 강의 내용에서 많이 업데이트됨
- Task definition
  - 애플리케이션의 블루프린트
  - [Amazon ECS 작업 정의](https://docs.aws.amazon.com/ko_kr/AmazonECS/latest/developerguide/task_definitions.html)
  - AWS에 컨테이너를 시작하는 방법을 알릴 수 있음
  - 'docker run'을 실행하는 방법이 아니라, 이를 실행하는 서버를 구성하는 방법
  - Task를 EC2와 비슷하다고 생각하면 됨, AWS에 컨테이너를 실행하는 방법과 컨테이너를 위한 환경을 알려주고 있음
  - FARGATE: 컨테이너를 시작하는 특정 방법으로 서버리스라고 부르는 모드에서 구동
  - AWS ECS는 EC2 인스턴스를 생성하지 않고 대신 컨테이너와 그 실행 설정을 저장함 (필요할 때만 서버를 구동) 
  - 컨테이너의 작업 요청이 있을 때마다 컨테이너를 시작하고 요청을 처리한 후에 중지함 (비용 효율적)
  - 1) Create new revision 으로 같은 설정의 새로운 Task 생성
  - 새 Task에서 서비스를 사용하면 AWS ECS가 업데이트된 이미지를 자동으로 가져옴
  - 2) Update Service -> Force new Deployment
- Service
  - 장기 실행되는 상태 비저장 애플리케이션
  - 모든 태스크가 서비스에 의해 실행된다
- Cluster
  - 우리 서비스가 실행되는 전체 네트워크
    
