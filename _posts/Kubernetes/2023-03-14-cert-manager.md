---
title: Cert Manager 설치하기
categories: [kubernetes]
tags: [kubernetes, mlflow, mlops, devops, ingress]
comments: true
---

# Cert Manager 설치하기
## Cert Manager 란?
![Cert Manager](/assets/img/post/cert-manager.png)
Cert Manager는 Kubernetes 내부에서 HTTPS 통신을 위한 SSL인증서를 생성하고, 인증서가 만료 되면 자동으로 인증서를 갱신해주는
`Certificate manager controller`이다.  

Kubernetes 내에서 외부에 존재하는 Issuer를 활용하거나, 내부에서 `Selfsigned Issuer`를 직접 생성하여 `Certificate`를 생성하고, 이 Certificate를 관리하여
인증서의 만료시간이 가까워 지면 인증서를 `자동으로 갱신`해준다.

Cert Manager가 지원하는 외부 Issuer는 다양한데, 무료로 제공하는 `Let's enscrypt`를 많이 사용한다.

![Cert-manager Issuer](/assets/img/post/cert-manager-issuer.png)

**P.S** 외부 Issuer를 활용하려면 도메인을 사용자가 직접사서 공개된 DNS에서 접속할 수 있는 도메인으로 Certificate의 인증을 받아야 한다!!!   
만약 인트라넷에서 활용하고 싶으면 추가적으로 설정해보고 업로드 해보겠습니다!!! **현재는 실패했다.**

<br>

## Installation
### 기본 설치 방법
Cert manager는 간단하게 Helm 을 사용하여 설치하면 된다.
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/${VERSION}/cert-manager.crds.yaml

helm repo add jetstack https://charts.jetstack.io

helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set startupapicheck.timeout=5m \
  --version ${VERSION}
``` 

### 손쉬운 설치!
Cert-Manager를 쉘파일 하나로 설치하고 싶으시다면 [개인 깃허브](https://github.com/sunjin7725/cert-manger-installation-shell)에 올려놓았습니다.
```bash
curl https://raw.githubusercontent.com/sunjin7725/cert-manger-installation-shell/master/cert-manager.sh | sh -s ${VERSION}
```