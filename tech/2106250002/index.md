---
layout: post
type: tech
date: 2021-06-25 00:02
category: CI/CD
title: Github Action으로 자동화 배포 구현하기
subtitle: Github Action과 K8s를 통해 CI/CD를 운영해보자!
writer: 970421
post-header: true
header-img: img/header.JPG
hash-tag: [Github_Action, K8s, Docker, CI/CD]
---

# 배포 전략 세우기

서비스를 운영하면서 어떻게 배포 전략을 세울지 고민을 해보았습니다. 그렇게 떠오르는 방안으론 두 가지가 있었습니다.

1. 젠킨스를 사용하여 배포하기
2. 떠오르는 샛별! 깃허브 액션으로 배포하기

젠킨스를 통해 배포하는 것도 좋지만, 그보다는 도커와 쿠버네티스를 활용하여 DevOps를 운영하고 싶었습니다. 그래서 CI 툴을 고민하다 요새 다양하게 활용하는 깃허브 액션을 사용해보기로 결정하게 되었습니다.

<img src="img/trials.JPG" style="zoom: 70%; display: center;">

~~수없이 실패한 깃허브 액션~~

## 배포 방법 정하기
-------------------------

깃허브 액션으로 배포를 진행하기로 했지만, 세부적으로 들어가면 또 하나 정해야할 게 존재합니다. 그건 바로 어떤 인프라를 구축하냐입니다. 클라우드 서비스를 활용하는 건 같지만, IaaS(GCP GCE, AWS EC2 등)를 사용할건지 SaaS(GCP GKE, AWS EKS 등)를 사용할건지에 따라도 달라집니다. 

각각의 장단점이 존재하지만, 저는 데브옵스 초보이기 때문에, 돈이 더 드는 SaaS를 사용하기로 결정했습니다. ~~(사장님 죄송해요)~~


## 배포 구현하기
---------------------------

그럼 본격으로 배포 인프라를 구축해봅시다.

다음과 같은 순서로 진행해보겠습니다.
1. GKE 생성
2. 깃허브 액션 작성
3. 액션과 관련된 secret 설정
4. Dockerfile 작성
5. deployment.yml, kustomization.yml 

### 1. GKE 생성

GKE를 생성하는 예제는 인터넷에 다양하니 넘어가도록 하겠습니다.
중요한 건 container registry를 활성화하는 것입니다. [시작하기](https://cloud.google.com/container-registry/docs/quickstart)

구글에서 친절하게 문서를 제공하니 찬찬히 따라보고 오세요!

### 2. 깃허브 액션 작성

이것도 사실 엄청 간단합니다. 

왜냐! 이미 구글에서 만들어서 github에 등록을 해놓았기 때문이죠.

<img src="img/github_action.JPG" style="zoom: 60%; display: center;">

~~못생긴 네모..~~

이미 정의된 깃허브 액션을 사용하시면 됩니다. 

~~~yml
# Set permission
- name: Grant execute permission for gradlew
    run: chmod +x gradlew
- name: Build with Gradle test
    run: ./gradlew test 
- name: Run copyDocument task
    run: ./gradlew copyDocument
- name: Build with Gradle
    run: ./gradlew build
~~~

저희 서비스 같은 경우, SpringBoot의 RestDocs를 사용하기 때문에, 위와 같은 Job을 추가로 작성했습니다. 간략하게 설명하자면, test를 실행하고, 실행된 test를 기반으로 생성된 RestDocs Snippets를 기반으로 Api Docs를 완성하고, 이제 이 모든것을 빌드하는 것입니다.

### 3. 액션과 관련된 secret 설정

깃허브 액션 내용을 살펴보면, 자세히 설명해 놓은 주석을 확인할 수 있습니다. 주석에 맞춰, 
secret과 환경변수를 설정해주시면 됩니다. 

대부분은 쉽게 검색해서 얻을 수 있는 정보지만, 이 중에서 DEPLOYMENT_NAME 같은 경우, 추후 작성할 
deployment.yml 파일에 정의된 이름을 사용하셔야 합니다!


### 4. Dockerfile 작성

액션 내에 정의된 docker build를 사용하기 위해 Dockerfile을 Root Folder에 작성해봅시다.

~~~Dockerfile
FROM openjdk:11-jdk

ARG JAR_FILE=build/libs/*.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
~~~

정말 짧죠?? 설명하면 jar 파일로 만들어서 실행시킨다는 의미 입니다!

### 5. deployment.yml, kustomization.yml 

이제 마지막입니다. 빌드한 도커 이미지를 Google Container Registry에 올렸다면, 해당 이미지를 통해 쿠버네티스 환경 구성을 위해 두 파일을 작성해야 합니다.

~~~yml
# deployment.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: sillock-app-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sillock-app-dev
  template:
    metadata:
      labels:
        app: sillock-app-dev
    spec:
      containers:
        - name: sillock-app-dev
          image: gcr.io/PROJECT_ID/IMAGE:TAG
          resources: {}
          ports:
            - containerPort: 8080
~~~

~~~yaml
# kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yml
~~~

## 최종 모습
---------------------------

<img src="img/steps.JPG" style="zoom: 80%; display: center;">

하나하나 제대로 돌아가는 지 확인하기 위해 잘게 쪼개다 보니 이미지와 같이 엄청난 step을 수행하게 되었습니다..😂


<img src="img/success.JPG" style="zoom: 60%; display: center;">

<img src="img/RestDocs.JPG" style="zoom: 50%; display: center;">

그래도 성공한 모습을 보니 뿌듯합니다. 

이렇게 26층 개발자팀에선 배포 환경을 구축하고, 최종적으로 프론트엔드 측에게 Rest API 문서를 제공합니다.

아직은 고칠 것도 많고(예를 들면, 캐시를 사용한 배포 속도 개선?), 맘에 들지 않는 모습도 많이 보이지만, 하나하나 개선해 나가는 모습 보여드리겠습니다!


<img src="img/final.JPG" style="zoom: 80%; display: center;">

## 참고 사이트

-----------------------------

[참고 사이트1](https://devopswithkubernetes.com/part-3/2-deployment-pipeline)

[참고 사이트2](https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment/)

[참고 사이트3](https://cloud.google.com/kubernetes-engine/docs/tutorials/hello-app)