# 섹션 9 - ECS로 EFS 볼륨 사용 \~ 문제 이해

﻿

## 148. ECS로 EFS 볼륨 사용하기

현재 저희는 ECS 서비스에서 데이터를 영구적으로 보존할 수 있도록 개선이 필요한 상황입니다.

우선 기존에 존재하던 goals에 느낌표(!)를 추가하고, 변경된 내용을 기반으로 새로운 Docker 이미지를 빌드한 뒤 **Docker Hub에 업로드했습니다**. 이후 ECS에서 해당 **서비스의 Service** 항목에 들어가 **Update 버튼**을 클릭하여 새로운 이미지를 기반으로 **서비스 업데이트 및 재배포를 진행**했습니다.

그러나 이렇게 업데이트 후 Postman을 통해 서비스를 호출해보면, 이전에 저장된 데이터가 모두 유실된 것을 확인할 수 있습니다.

이유는 명확합니다. ECS는 새로운 Task를 실행하면서 기존 컨테이너를 종료하고 새로운 컨테이너로 교체하는데, 이 과정에서 컨테이너 내부에 존재하던 데이터는 모두 삭제됩니다.

이는 로컬 환경에서도 동일하게 발생합니다. 로컬에서 컨테이너를 재시작하면 컨테이너 내부 데이터는 사라지지만, Docker Volume을 설정해두면 데이터를 외부 볼륨에 저장함으로써 데이터 유실 문제를 해결할 수 있습니다.

ECS 환경에서도 마찬가지로 볼륨을 활용해야 합니다. 하지만 ECS에서 일반적인 Docker 볼륨(Local Volume)을 사용할 수는 없습니다. ECS에서의 볼륨은 클러스터 내 여러 인스턴스에서 공유되어야 하고, 높은 가용성과 안정성을 갖춰야 하기 때문입니다.

### 그렇다면 왜 EFS를 사용해야 할까

* EFS(Elastic File System)는 AWS에서 제공하는 완전관리형 NFS(Network File System) 스토리지입니다.
* 다중 ECS Task 또는 EC2 인스턴스에서 동시에 접근 가능하여, 확장성과 공유성을 확보할 수 있습니다.
* EC2 또는 Fargate 기반 ECS 작업 모두에서 마운트 가능하며, 파일시스템 수준의 영속성과 가용성을 제공합니다.
* 컨테이너가 재시작되거나 교체되더라도, EFS에 저장된 데이터는 그대로 유지되므로 상태 유지(Stateful) 서비스에 적합합니다.

<table data-header-hidden><thead><tr><th width="249"></th><th></th><th></th></tr></thead><tbody><tr><td>항목</td><td>Docker Local Volume</td><td>EFS Volume</td></tr><tr><td>접근 범위</td><td>단일 컨테이너/호스트</td><td>다수의 컨테이너/호스트에서 공유 가능</td></tr><tr><td>가용성</td><td>호스트에 종속적</td><td>고가용성, 멀티 AZ 지원</td></tr><tr><td>백업 및 복구</td><td>수동 설정 필요</td><td>자동 백업 및 AWS 통합 기능</td></tr><tr><td>네트워크 접근</td><td>불가능</td><td>가능 (NFS 기반)</td></tr></tbody></table>

그러므로 저희는 AWS에서 EFS 볼륨을 정의해줍니다.

이제 컨테이너에 볼륨을 연결해줘야 합니다. 그리하여 MongoDB 컨테이너의 이름을 클릭하여 Config를 열어 Straoge And Logging 부분에 Mount Points 에 저희가 정의한 볼륨을 선택하고,

Docker Image에 정의된 경로를 Container Path 부분에 넣어줍니다. 그러고 난 후 업데이트를 진행시켜 주면 볼륨설정은 끝입니다

## 150. 현재 아키텍처

<figure><img src="../.gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

## 151. 데이터베이스 & 컨테이너 : 중요한 고려사항

<figure><img src="../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

컨테이너 환경에서도 자체적으로 데이터베이스를 구축하고 운영하는 것은 가능합니다.

예를 들어 MongoDB나 MySQL을 Docker 컨테이너로 띄워 직접 관리하는 방식입니다.

그러나 이런 방식은 다음과 같은 중요한 문제점들을 동반합니다.



* **확장성과 가용성 확보의 어려움**
  * 컨테이너 기반으로 운영될 경우, 단일 인스턴스에 장애가 발생하면 전체 데이터베이스 접근이 차단될 수 있습니다. 가용성을 높이기 위해 Replica Set 구성, 클러스터링, 장애 조치 등을 직접 구현해야 하며 이는 매우 복잡합니다.
* **읽기/쓰기 병행 처리**
  * 실시간 서비스에서는 동시다발적인 읽기/쓰기 요청이 발생합니다. 이럴 경우 락 경쟁, I/O 병목 등으로 인해 성능 저하가 발생할 수 있습니다.
* **트래픽 급증에 대한 대응 부족**
  * 유저 수나 요청량이 급격히 증가할 경우, 컨테이너 기반 DB는 쉽게 한계에 도달합니다.
* **백업 및 보안 관리의 부담**
  * 주기적인 스냅샷, 장애 발생 시 데이터 복구, 암호화 및 접근 제어 설정 등을 직접 구현해야 하며 이는 인프라 지식이 반드시 요구됩니다.

이러한 문제를 해결하기 위해 **자체 DB 컨테이너 운영에서 벗어나, 관리형 데이터베이스 서비스로의 전환**을 고려해야 합니다.

<figure><img src="../.gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>

## 152. MongoDB Atlas로 이동하기

저희는 위와같은 단점들이 존재하기 떄문에 관리형 데이터베이스 서비스로 옮기도록 하겠습니다.

https://www.mongodb.com/products/platform/atlas-database

해당 사이트에 들어가 회원가입을 한 이후 계정을 생성하여 Cluster 를 만들어 줍니다.

<figure><img src="../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

저희는 단순히 공부를 하기위한 목적이기 떄문에 Free를 선택해줍니다.

<figure><img src="../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

클러스터를 생성 해줍니다.

그리고 Cluster 옆에 존재하는 Connect 를 눌러 어떻게 연결할지 선택할 수 있습니다.

<figure><img src="../.gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

저희는 Application을 통해서 연결하니 Drivers를 선택해줍니다.

<figure><img src="../.gitbook/assets/image (15).png" alt=""><figcaption></figcaption></figure>

그리고 3번째 요소에 존재하는 설정을 통해 Connect를 해줄 수 있습니다.

이제 MongoDB Atlas를 사용할 때 중요한 결정 중 하나는 **개발(Dev) 환경과 운영(Prod) 환경에서 같은 DB 인스턴스를 사용할 것인지 여부입니다**.

1\. 운영 환경 전용 클러스터를 분리하는 경우

* **장점**: 운영 데이터의 안전성 확보, 개발자의 실수로 인한 데이터 손상 방지
  * 주의사항:
    * Dev와 Prod 환경의 MongoDB 버전이 완전히 동일해야 함
    * 스키마 차이, 기능 차이로 인해 예기치 않은 오류 발생 가능
    * CI/CD 또는 테스트 단계에서 장애 재현이 어려워질 수 있음

2\. 동일한 클러스터를 Dev/Prod가 공유하는 경우

* **장점**: 개발-운영 간 환경 차이를 최소화, 이슈 발생 시 원인 파악 용이
* **단점**: 테스트 과정에서 실 운영 데이터 훼손 위험 (Read-only 권한 분리로 보완 가능)



저희는 운영 안정성과 코드 일관성을 고려하여 **Dev와 Prod 환경을 동일하게 유지**하기로 했습니다.

그럼이제 저희가 사용하는 Docker-Compose 파일은 무엇을 의미하는건가 생각을 해봐야합니다

이제 저희는 Cloud DataBase가 존재하기 때문에 더 이상 자체 MongoDB를 사용하지 않습니다.

또한 이제 컨테이너를 재시작할 일이 없어 데이터 유실의 걱정도 할 필요가 없어졌습니다 .그러므로 데이터 볼륨 또한 사용할 필요가 없습니다.

그러므로 해당 부분들을 이제 전부 없애도록 하겠습니다.

## 153. 프로덕션에서 MongoDB Atlas 사용하기

이제 기존에 존재했던 ECS에서 Mongo 컨테이너를 제거하고 클라우드 RDS에 대한 연결과 데이터베이스 이름을 전달하도록 컨테이너 노드를 다시 시작하도록 하겠습니다.

그리하여 **Task Definitions에 Revison을** 선택하여 MongoDB에 대한 **컨테이너와 볼륨을 제거**해줍니다.

이제 저희가설정한 goals-backend에서 구성을 변경해줍니다.

기존에는 MONGODB\_URL 이 loaclhost로 설정되어있었지만, 저희는 발급받은 ClusterURL로 변경해주어야합니다. 또한 발급받은 유저네임과 패스워드도 변경해주도록 합니다.

그리고 해당 부분들을 다 변경해준 후 Update Service를 통해 서비스를 재베포 시켜줍니다.

이렇게 함으로써 저희는 RDS를 사용할 수 있게 되었습니다.

## 154. 업데이트 & Target 아키텍처

현재 저희는 기존에 존재하던 **자체 관리 데이터베이스** 에서 **관리형 데이터베이스**로 전환하였습니다

<figure><img src="../.gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>

위와같은 아키텍쳐를 가지고 있습니다

이제 저희는 React SPA를 보유하는 두 번쨰 컨테이너를 실행하고자 합니다.

<figure><img src="../.gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

## 155. 일반적인 문제 이해하기

<figure><img src="../.gitbook/assets/image (19).png" alt=""><figcaption></figcaption></figure>

**빌드 단계와 개발 vs 프로덕션 환경**

일부 프로젝트에서는 개발 중인 코드를 실행하기 위해 **빌드 단계**가 필요합니다. 이는 나중에 배포할 코드가 아니라 개발 과정에서 사용하는 코드로, 개발 환경과 프로덕션 환경 간에는 차이가 존재합니다.

도커만으로는 이 문제를 해결할 수 없습니다. 이는 프로젝트의 특성상 내재된 문제이기 때문입니다.

### React SPA 프로젝트

React SPA(Single Page Application)는 대부분 브라우저에서 실행되는 JavaScript 코드로 구성됩니다. 아래는 React 코드의 예입니다

```javascript
 return (
    <div>
      {error && <ErrorAlert errorText={error} />}
      <GoalInput onAddGoal={addGoalHandler} />
      {!isLoading && (
        <CourseGoals goals={loadedGoals} onDeleteGoal={deleteGoalHandler} />
      )}
    </div>
  );
}
```

**핵심 문제점**

위 코드는 **실제로 존재하는 표준 JavaScript 문법이 아닙니다**. **이는 React 프레임워크를 사용한 개발자 친화적인 코드**이며, 브라우저는 이 코드를 직접 실행할 수 없습니다.

컴파일 과정을 거친 후에는 브라우저가 이해할 수 있는 전혀 다른 형태의 코드로 변환됩니다.

이러한 변환과정을 통해 브라우저가 실제 기능을 수행할 수 있게 됩니다.

### 개발 환경과 프로덕션 환경의 차이

**개발 환경의 한계**

* npm start로 시작되는 **개발 서버는 프로덕션 환경에 적합하지 않음**
* 프로덕션 최적화가 되어있지 않음
* 실시간 컴파일 프로세스가 과도한 리소스를 사용
* 성능이 느려 프로덕션 환경에서 사용하기 부적절

**프로덕션 환경의 요구사항**

React와 유사한 프로젝트들은 프로덕션을 위해 start 대신 별도의 빌드 스크립트를 제공합니다.

**빌드 프로세스의 특징**

* 개발 서버를 실행하지 않음
* 코드 컴파일 및 최적화 수행
* 최적화된 정적 파일들을 생성
* 생성된 파일들은 선택한 웹 서버(Nginx, Apache 등)를 통해 서비스 제공

결론

이러한 이유로 **개발용 Docker 컨테이너를 그대로 프로덕션 환경으로 옮길 수 없으며, 개발과 프로덕션을 위한 별도의 Docker 설정과 빌드 전략**이 필요합니다.

﻿
