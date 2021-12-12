# node_gitops
### kubernetes configurations manifest repository for node-app(node_devops)
- reference
  - https://kangwoo.kr/2019/12/18/gitops-and-kubernetes/ 
    <br>kustomize, devops/gitops 구성
  - https://kubernetes.io/ko/docs/tutorials/kubernetes-basics/create-cluster/cluster-intro/
    <br>kubernetes minikube 구성
  - https://velog.io/@aylee5/Kubernetes-Argo-CD-%EC%84%A4%EC%B9%98-%EB%B0%8F-%EC%84%9C%EB%B9%84%EC%8A%A4-%EB%B0%B0%ED%8F%AC-GitOps
    <br>https://argo-cd.readthedocs.io/en/stable/getting_started/
    <br>ArgoCD Helm 설치 및 ArgoCD 설치 가이드

## 1. Kubernetes 구성
1) minikube 설치 및 구성
  - 자신의 운영체제에 맞는 형식으로 minikube 설치
  ``` sh
  brew update && brew install minikube
  minikube start
  ```
2) ArgoCD 설치
  - 자신의 운영체제에 맞는 형식으로 helm 설치
  ``` sh
  brew update && brew install helm
  ```
  - kubernetes namespace 생성
  ``` sh
  kubectl create namspace argo
  ```
  - helm repository 추가 및 업데이트
  ``` sh
  helm repo add argo https://argoproj.github.io/argo-helm
  helm repo update
  ```
  - helm chart 받아오기 및 압축풀기
  ``` sh
  helm fetch argo/argo-cd
  tar -xvzf argo-cd-${argo-cd version}.tgz
  cd argo-cd
  ```
  - ./argo-cd/values.yaml 파일 수정
    <br> server.service.type : ClusterIP -> LoadBalancer
  - argo-cd 설치
  ``` sh
  helm install argo -n argo argo/argo-cd -f values.yaml
  ```
  - argo-cd cli 설치(optional)
  ``` sh
  brew install argocd
  ```
  - 필요한 경우 port-foward를 이용하여 포트 변경
  ``` sh
  nohup kubectl port-forward service/argo-argocd-server -n argo ${원하는 포트}:${argo-argocd-server port(ex 443)} &
  ```
  - 기본 비밀번호 가져오기
  ``` sh
  kubectl -n argo get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
  ```
  - argo-cd 접속 후 admin, 기본 비밀번호 입력 후 로그인
  
## 2. gitops repository 구성
  - https://github.com/questcollector/node_gitops

## 3. jenkins 구성(jenkins를 docker로 구성했을 경우)
  - jenkins Dockerfile을 이용하여 kustomize 설치
  ``` Dockerfile
  FROM jenkins/jenkins:latest AS docker
  USER root
  RUN apt-get update && apt-get install -y lsb-release
  RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
    https://download.docker.com/linux/debian/gpg
  RUN echo "deb [arch=$(dpkg --print-architecture) \
    signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
    https://download.docker.com/linux/debian \
    $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
  RUN apt-get update && apt-get install -y docker-ce-cli

  FROM docker AS kustomize
  WORKDIR /usr/bin/
  RUN curl --silent --location --remote-name https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv4.4.1/kustomize_v4.4.1_linux_arm64.tar.gz
  RUN tar -xzf kustomize_v4.4.1_linux_arm64.tar.gz

  FROM kustomize AS jenkins
  USER jenkins
  RUN jenkins-plugin-cli --plugins "docker-workflow:1.26"
  ```
  - devops repository의 Jenkinsfile 구성(Stage: Deploy to gitOps Repository)
  <br> https://github.com/questcollector/node_devops/blob/master/Jenkinsfile

## 4. ArgoCD 새로운 앱 만들기
  - 애플리케이션용 namespace 생성
  ``` sh
  kubectl create namspace node-app
  ```
  - 앱 생성
  ``` yaml
  Application Name: node-app
  Project: default
  SYNC POLICY: Automatic
  Repository URL: https://github.com/questcollector/node_gitops.git
  Revision: HEAD
  Path: overlays/dev/
  Cluster: https://kubernetes.default.svc
  Namespace: node-app
  ```
## 5. minikube service open
  ``` sh
  nohup minikube service --url node-service -n node-app &
  ```
