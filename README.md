# 구독관리 서비스 배포 가이드
이 가이드는 구독관리 서비스의 Front, Backend, Spring Cloud 서버를 배포하는 방법을 안내 합니다.   

## 배포 가이드 클론
Docker, Docker Compose, Kubectl이 설치된 작업 머신에 배포 가이드를 클론합니다.  
이 작업 머신은 k8s cluster에 애플리케이션을 배포할 수 있도록 환경이 구성되어 있어야 합니다.  

```
mkdir -p ~/install/subride && cd ~/install/subride
git clone https://github.com/cna-bootcamp/subride-deploy.git
```

## 내용 수정
### 환경변수 .env 파일
- FRONT_HOST: Backend application에 API를 요청하는 Front 주소. CORS설정을 위해 필요.   
- MYSQL, RabbitMQ 관련 환경변수: 이 값을 변경하면 k8s배포 시 mysql.yaml, rabbitmq.yaml도 수정해야 함      
> Tip  
  사실 이 변수는 로컬에서 container로 실행할 때 필요한 값이라, k8s배포만 한다면 수정 안해도 됨.  

### build.yml: Jar생성 정의 파일
Docker Compose로 Spring Cloud와 Subride backend application의 jar파일을 생성하는   
정의 파일입니다.   
이 파일에서 수정할 내용은 없습니다.   

### docker-compose.yml: Image 빌드 및 Container 배포 정의 파일  
이 파일에는 Container image를 빌드하고, 현재 머신에 container로 application을 실행하는   
방법이 정의되어 있습니다.   
우리는 k8s에서 배포할 것이므로 image 빌드 부분만 사용합니다.   

- 공통 수정 내용: image 경로를 본인의 것으로 변경  
  ```  
  sed -i'' "s@docker.io/hiondal@docker.io/gappa@g" docker-compose.yaml
  ```
  
  image tag명도 필요시 변경합니다.  
  ``` 
  sed -i'' "s@image: .*:2.0.0@image: .*:1.0.0@g" docker-compose.yaml
  ```


- config: environment하위의 'GIT'으로 시작하는 Config 저장소 관련 변수 수정

### deploy/ 하위 배포 yaml 파일
- mysql.yaml: Helm chart로 배포하기 위한 custom values
  - storageClass: Dynamic provisioning이 설정된 Storage class명으로 지정  
  - auth 항목 밑의 설정. 이 값은 .env와 일치해야 함. replicationPassword는 적절히 지정. 
- rabbitmq.yaml: 
  - 환경변수 RABBITMQ_DEFAULT_USER, RABBITMQ_DEFAULT_PASS를 .env와 동일하게 지정. 
  - namespace를 배포할 namespace로 변경.  
- subride하위의 yaml
  - 공통 수정: namespace, ingress host, image경로를 일괄적으로 수정(아래 예제 참조)  
    모든 yaml에 대해 수행함. 아래는 namespace를 'ondal'로,  
    ingress domain을 'cna.com'으로,   
    image 경로에서 organization을 바꾸는 예시임.  
    ```
    sed -i'' "s/namespace:.*/namespace: ondal/g" config.yaml
    sed -i'' "s/msa.edutdc.com/cna.com/g" config.yaml
    sed -i'' "s@docker.io/hiondal@docker.io/gappa@g" config.yaml
    ```

    image tag명도 위 docker-compose와 동일하게 바꿉니다.
    ```
    sed -i'' "s@image: .*:2.0.0@image: .*:1.0.0@g" config.yaml
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
build.yml이 있는 디렉토리로 이동하여 수행   

먼저 Spring Cloud 프로젝트, Subride backend, Subride frontend 소스를 clone합니다.   
```
cd ~/install/subride
git clone https://github.com/cna-bootcamp/sc.git
git clone https://github.com/cna-bootcamp/subride.git
git clone https://github.com/cna-bootcamp/subride-front.git

```

jar파일을 생성합니다.  
```
docker-compose -f build.yml up
```

## Container image Build/Push

docker-compose.yml이 있는 디렉토리에서 수행   
- Build image
  ```
  docker-compose build
  ```
- Push image
  ```
  docker login 
  docker-compose push
  ```
> Tip: 서비스명은 docker-compose.yml의 'service'섹션 하위에 정의된 이름 사용   
  - 특정 서비스만 build: docker-compose build {서비스명}  
  - 특정 서비스만 push: docker-compose push {서비스명}  

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
- MySQL배포
  ```
  helm repo add bitnami https://charts.bitnami.com/bitnami 
  helm repo update
  helm upgrade mysql -i -f mysql.yaml bitnami/mysql  
  ```

- RabbitMQ 배포
  ```
  kubectl apply -f rabbitmq.yaml 
  ```

- 구독관리 서비스, Spring cloud 배포  
  ```
  kubectl apply -f deploy/subride
  ```

- 확인 및 테스트   
  모든 pod가 실행될때까지 기다립니다.   
  ```
  watch kubectl get po
  ``` 

  브라우저에서 Frontend 주소로 접근하여 확인합니다.  


