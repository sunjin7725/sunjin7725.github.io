---
title: 미니버젼 MLOps 플랫폼 구축하기
categories: [kubernetes]
tags: [kubernetes, mlflow, mlops, devops]
comments : true
---

# 미니버젼 MLOps 플랫폼 구축하기
관세청 빅데이터 플랫폼 사업을 3년간 진행하면서 제일 많이 사용했던 솔루션이 DevOps 솔루션이였던 것 같다(그중에서도 CI/CD만 엄청 돌렸다. 무한개발배포).  
사용된 DevOps 솔루션은 우리가 코드를 작성하고 깃에 `Push`하고, 그 다음 솔루션 내에 연결된 파이프라인을 실행하면 쿠버네티스에 배포해주는 형태였는데, 해당 솔루션을 보면 `Rancher`나 `Harbor`등의 오픈 소스 기반으로 되어있다.  
이런 오픈소스들의 집합을 사업 외, 사내 서버에 구축하여 사내 미니 프로젝트나, PoC 등에서 활용하면 유용할 것 같아 구축해보고자 한다.  
본래는 MLFlow나 분석가들을 위한 내용이 들어가있지 않고, 어플리케이션 배포 자동화에 치중되어있었지만 MLFlow나 다른 MLOps 관련 프로젝트들도 추가하여 연동해보려 한다.  
실제 내부에서 사용했던 플랫폼의 세부 사항?! 오픈소스?! 들과는 몇몇 다른 부분은 있지만 똑같은 기능을 구현 할 수 있는 
오픈 소스 프로젝트 들로 구성하려 하며 예상 Arichitect는 아 래 구성도와 같다.  
**Note** 미니버젼이라고 한 이유는 하둡에코시스템도 만들어야될 것 같고, ELK도 넣어서 로그도 쌓고 하면 좋겠지만 너무 비루하다.

## 구성도 
![MLOps 플랫폼 Architect](/assets/img/post/mlops.png)

## ToDo
할일은 kubernetes 클러스터를 구축하고 사용할 오픈소스 프로젝트들을 Kubernetes에 설치만 하면 된다. 사실 MLFlow 외에 다른 프로젝트들은 Kubernets에 다올렸지만 글을 올리면 체크박스들을 체크할 예정!   
위에 구성도에는 없지만 RDBMS도 한개 설치된다.
- [x] Kubernetes 클러스터 구축하기(실제로는 k3s) [바로가기](/posts/k3s-installation)
  - helm 설치 [바로가기](/posts/helm-installation)
  - 외부 배포용 ingress-controller 설치 [바로가기](/posts/ingress-controller)
  - SSL 인증서 갱신을 위한 cert-manager 설치 [바로가기](/posts/cert-manager)
- [x] Rancher 구축하기 [바로가기](/posts/rancher)
- [ ] Harbor 구축하기
  - docker hub registry 연결
- [ ] Jenkins 구축하기
  - 깃허브와 연동하기
  - host 서버에 있는 docker, kubernetes와 연동하기
- [ ] 분석 데이터 저장 및 마트로 활용할 RDBMS 설치(oracle21c)
- [ ] MLFlow 구축하기
- [ ] 분석 API 배포하기