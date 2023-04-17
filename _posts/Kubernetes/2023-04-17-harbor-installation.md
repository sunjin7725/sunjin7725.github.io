---
title: Harbor 설치하기
categories: [kubernetes]
tags: [kubernetes, mlops, devops, harbor]
comments : true
---

# Harbor 설치하기
## Harbor 란?
![harbor](/assets/img/post/harbor.png)  
Docker에서 다른 사람이 만든 이미지를 다운 받고, 많은 사람들에게 공개해주기 위해서 Docker Hub 라고 하는 Public image registry를 주로 사용한다.

하지만, 사용자가 만든 이미지를 공개하고 싶지 않거나, 기업 내의 프로젝트에서 만든 이미지를 관리하기 위해서는 Private image registry 가 필요하다.  

이런 Private image registry를 대표하는 것이 Harbor이다. 다수의 이미지를 프로젝트 별로 관리하기 용이하고, 프로젝트 별 권한관리 등을 Web UI를 통해 제공해주는 것이 Harbor의 장점이다.

### Harbor의 장점
- Docker Image 개인 저장소 제공
- Cloud-Native 기반 오픈 소스
- Web UI 제공
- 1개까지만 무료인 docker registry와 달리, 추가 요금 없이 다수의 registry 생성 가능(프로젝트별 이미지 분류 용이)

**공식홈페이지** [https://goharbor.io/](https://goharbor.io/)  

<br>  

## Installation
### 기본 설치 방법
Harbor는 helm을 통해 간단하게 설치 할 수 있다.
```bash
helm repo add harbor https://helm.goharbor.io
helm repo update

helm install harbor harbor/harbor --namespace=harbor --create-namespace
```
  
### 손쉬운 설치!
Harbor를 쉘파일 하나로 설치하고 싶으시다면 [개인 깃허브](https://github.com/sunjin7725/harbor-installation-shell)에 올려놓았습니다.
```bash
# 아래 명령어를 사용하면 바로 설치 됩니다.
curl https://raw.githubusercontent.com/sunjin7725/harbor-installation-shell/master/harbor.sh | sh -
```

<br>
<br>

참고자료  
- [https://goharbor.io/docs/2.8.0/](https://goharbor.io/docs/2.8.0/)
- [https://velog.io/@tkfrn4799/harbor-private-docker-registry](https://velog.io/@tkfrn4799/harbor-private-docker-registry)

## 사용방법
설치가 다된 후에 다음 명령어를 입력하면 사용할 수 있다.
```bash
docker login ${harbor registry url}
```