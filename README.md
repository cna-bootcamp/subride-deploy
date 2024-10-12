# 구독관리 서비스 배포 가이드
이 가이드는 구독관리 서비스의 Front, Backend, Spring Cloud 서버를 배포하는 방법을 안내 합니다.   

![main](subride.png)

## 배포 가이드 클론
Docker, Docker Compose, Kubectl이 설치된 작업 머신에 배포 가이드를 클론합니다.  
이 작업 머신은 k8s cluster에 애플리케이션을 배포할 수 있도록 환경이 구성되어 있어야 합니다.  

```
mkdir -p ~/install && cd ~/install
git clone https://github.com/cna-bootcamp/subride-deploy.git
cd subride-deploy
```

## 내용 수정
### 환경변수 .env 파일
- FRONT_HOST: Backend application에 API를 요청하는 Front 주소. CORS설정을 위해 필요.   
- MYSQL, RabbitMQ 관련 환경변수: 이 값을 변경하면 k8s배포 시 mysql.yaml, rabbitmq.yaml도 수정해야 함      
> Tip  
  사실 이 변수는 로컬에서 container로 실행할 때 필요한 값이라, k8s배포만 한다면 수정 안해도 됨.  

### build-jar.yaml: Jar생성 정의 파일
Docker Compose로 Spring Cloud와 Subride backend application의 jar파일을 생성하는   
정의 파일입니다.   
이 파일에서 수정할 내용은 없습니다.   

### build.yaml: Image 빌드 및 Container 배포 정의 파일  
이 파일에는 Container image를 빌드하고, 현재 머신에 container로 application을 실행하는   
방법이 정의되어 있습니다.   
우리는 k8s에서 배포할 것이므로 image 빌드 부분만 사용합니다.   

- 공통 수정 내용: image 경로를 본인의 것으로 변경  
  ```  
  sed -i'' "s@docker.io/hiondal@docker.io/gappa@g" build.yaml
  ```
  
  image tag명도 필요시 변경합니다.  
  ``` 
  sed -i'' "s/:2.0.0/:1.0.0/g" build.yaml
  ```


- config: environment하위의 'GIT'으로 시작하는 Config 저장소 관련 변수 수정

### deploy/ 하위 배포 yaml 파일
- deploy디렉토리로 이동
  ```
  cd deploy
  ```
- mysql.yaml: Helm chart로 배포하기 위한 custom values
  - storageClass: Dynamic provisioning이 설정된 Storage class명으로 지정  
  - auth 항목 밑의 설정. 이 값은 .env와 일치해야 함. replicationPassword는 적절히 지정. 
- rabbitmq.yaml: 
  - 환경변수 RABBITMQ_DEFAULT_USER, RABBITMQ_DEFAULT_PASS를 .env와 동일하게 지정. 
  - namespace를 배포할 namespace로 변경.
    ```
    sed -i'' "s/namespace:.*/namespace: ondal/g" rabbitmq.yaml
    ```
  - serviceAccount, serviceAccountName: 별도 Service Account를 만들었으면 변경  
   
- subride하위의 yaml
  - subride디렉토리로 이동
    ```
    cd subride
    ```
  - 공통 수정: namespace, ingress host, image경로를 일괄적으로 수정(아래 예제 참조)  
    **모든 yaml파일에 대해 수행**함.
    아래는 namespace를 'ondal'로,  
    ingress domain을 'cna.com'으로,   
    image 경로에서 organization을 바꾸는 예시임.  
    ```
    sed -i'' "s/namespace:.*/namespace: ondal/g" config.yaml
    sed -i'' "s/msa.edutdc.com/cna.com/g" config.yaml
    sed -i'' "s@docker.io/hiondal@docker.io/gappa@g" config.yaml
    ```

    image tag명도 위 build.yaml과 동일하게 바꿉니다.
    ```
    sed -i'' "s/:2.0.0/:1.0.0/g" config.yaml
    ```

  - config.yaml
    - ConfigMap의 'GIT_'으로 시작하는 Config 저장소 관련 변수 수정
    - Secret의 'GIT_TOKEN'값 수정
  - member.yaml
    - ConfigMap의 FRONT_HOST
    - FRONT_HOST: Backend application에 API를 요청하는 Front 주소. CORS설정을 위해 필요.  
      ```
      FRONT_HOST: http://subride-front.msa.edutdc.com,http://localhost:3000
      ```
    - ConfigMap의 RABBITMQ_USERNAME: rabbitmq.yaml에서 지정한 값과 동일해야 함
    - Secret의 RABBITMQ_PASSWORD: rabbitmq.yaml에서 지정한 값과 동일해야 함
    - Secret의 DB_PASSWORD: mysql.yaml에서 지정한 값과 동일해야 함
  - scg.yaml
    - ConfigMap의 'ALLOWED_ORIGINS'. Backend application에 API를 요청하는 Front 주소. CORS설정을 위해 필요.   
  
## Jar파일 Build
build-jar.yaml이 있는 디렉토리로 이동하여 수행   

먼저 Spring Cloud 프로젝트, Subride backend, Subride frontend 소스를 clone합니다.   
```
cd ~/install/subride-deploy
git clone https://github.com/cna-bootcamp/sc.git
git clone https://github.com/cna-bootcamp/subride.git
git clone https://github.com/cna-bootcamp/subride-front.git

```

jar파일을 생성합니다.  
```
docker-compose -f build-jar.yaml up
```

## Container image Build/Push

build.yaml이 있는 디렉토리에서 수행   
- Build image
  ```
  docker-compose -f build.yaml build
  ```
- 이미지 저장소에 로그인

  ```
  docker login -u {user id}
  ```

- Push image
  ```
  docker-compose -f build.yaml push
  ```
> Tip: 서비스명은 build.yaml의 'service'섹션 하위에 정의된 이름 사용   
  - 특정 서비스만 build: docker-compose -f build.yaml build {서비스명}  
  - 특정 서비스만 push: docker-compose -f build.yaml push {서비스명}  

## k8s에 배포
- namespace 생성 또는 이동  
  ```
  kubectl create namespace {namespace명}  
  kubens {namespace명}
  ```
- Image pull secret 객체생성  
  ```
  kubectl create secret docker-registry dockerhub --docker-server=docker.io --docker-username={userid} --docker-password={password}
  ```

- helm 저장소 추가
  bitnami helm chart 저장소가 추가 안되어 있으면 수행하세요.  
  ```
  helm repo add bitnami https://charts.bitnami.com/bitnami 
  helm repo update
  ```
  
- MySQL배포

  ```
  helm upgrade mysql -i -f deploy/mysql.yaml bitnami/mysql  
  ```

- RabbitMQ 배포
  ```
  kubectl apply -f deploy/rabbitmq.yaml 
  ```

- 구독관리 서비스, Spring cloud 배포  
  ```
  kubectl apply -f deploy/subride
  ```

- 파드 상태 확인     
  모든 pod가 실행될때까지 기다립니다.   
  ```
  watch kubectl get po
  ``` 

- 테스트
  - /etc/hosts등록
    만약 DNS에 ingress host를 등록하지 못한다면, 로컬PC의 /etc/hosts파일에 주소를 등록해야 합니다.
    아래 예제를 참고하여 등록합니다.
    IP는 k8s cluster를 접근할 수 있는 Public IP이어야 합니다.
    k8s node가 외부에 오픈되어 있다면 k8s노드 중 아무 노드의 IP를 입력하고,
    proxy서버를 통해 k8s cluster를 접근하는 경우는 Proxy서버의 IP를 지정하여야 합니다.    
    ```
    18.141.104.160 subride-front.msa.edutdc.com scg.msa.edutdc.com config.msa.edutdc.com eureka.msa.edutdc.com
    18.141.104.160 member.msa.edutdc.com subrecommend.msa.edutdc.com mygrp.msa.edutdc.com mysub.msa.edutdc.com transfer.msa.edutdc.com
    ```   

  - Eureka에 서비스 등록 확인
    브라우저에서 eureka주소로 접근 합니다. eureka주소는 deploy/subride/eureka.yaml에 지정되어 있습니다.   
    MEMBER-SERVICE, MYGRP-SERVICE, MYSUB-SERVICE, SCG, SUBRECOMMEND-SERVICE,
    TRANSFER-SERVICE라는 이름으로 총 6개의 서비스가 등록 되어야 합니다.
    약간 시간이 걸립니다. 모든 서비스가 등록될때까지 기다리셨다가 다음 단계를 진행 합니다.  

  - 브라우저에서 Frontend 주소로 접근
    'SIGN UP'버튼을 눌러 회원가입을 먼저 합니다.
    id는 아무거나 등록해도 상관 없지만 테스트 지출 데이터가 user01 ~ user05에 대해 생성되어 있으므로,
    테스트를 위해 이 5개 계정 중 하나로 등록합니다.
    

## 배포 객체 삭제 
아래 명령으로 모든 객체를 삭제 합니다.  
```
cd ~/install/subride-deploy

helm delete mysql
kubectl delete -f deploy/rabbitmq.yaml
kubectl delete -f deploy/subride  
```

PVC를 삭제합니다.  
```
kubectl get pvc
kubectl delete pvc data-mysql-primary-0 data-mysql-secondary-0
```

PV를 삭제합니다.  
PVC 목록에서 'VOLUME'항목에 있는 값이 PV이름입니다. 
```
kubectl delete pv {PV이름}
```

> Tip: PV상태가 'Released'인 PV를 한꺼번에 삭제할 수도 있습니다.
  ```
  kubectl get pv -o json | jq -r '.items[] | select(.status.phase=="Released") | .metadata.name' | xargs kubectl delete pv
  ``` 

NFS Dynamic provisioning으로 자동 생성된 물리적 볼륨도 삭제 합니다.   
데이터가 영구적으로 삭제 되므로 주의해서 삭제하시기 바랍니다.   
물리적 볼륨 디렉토리는 NFS서버가 설치된 머신에 있으며 그 디렉토리는 설정했을 때 지정 했습니다.  
본인이 인프라까지 관리 안 한다면 인프라 관리자에게 요청해야 합니다.  

