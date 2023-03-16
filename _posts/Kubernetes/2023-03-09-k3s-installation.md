---
title: K3S 설치하기
categories: [kubernetes]
tags: [kubernetes, mlops, devops, k3s]
comments : true
---

# K3S 설치하기
## K3S 란?
K3S는 쉽게 생각하면 가벼운 버젼의 Kubernetes라고 생각하면 될 것 같다. 
컨테이너 클러스터 관리를 하는 프로젝트인 Rancher를 만들 Rancher Labs에서 만들었는데 
공식홈페이지를 보면 `Edge device`나 `홈 IoT `처럼 `리소스가 부족한 환경`에서 쿠버네티스를 활용한 경우를 상정하고 개발한 듯하다.  
K3S는 kubernetes와 비교하면 크게 두가지 차이점이 있다.

- 경량화  
  K3S `외부 클라우드 서비스와의 연동 기능을 최소한`으로 줄이고, 고가용성 배포를 위해 기존 kubernetes에서 사용하던 `etcd 의존성을 없애고`
  `sqlite`를 기본값으로 사용한다. 또한 `Docker container를 사용하지 않으려` 하고 리눅스 체제의 `containerd`를 사용한다
  (etcd나, docker는 필요하다면 설정해서 사용할 수 있다). 또한 kubernetes에서 지원하는 과거 버전의 API도 대폭 줄여 경량화를 하였다.

- 설치의 용이성  
  K3S는 설치가 무척이나 간단하다. 경량화를 통해 시스템 요구사항을 극단적으로 줄여(rasberry와 같은 미니 PC나 edge device 등에도 사용가능!), 
  쉘 스크립트 하나로 kubernetes에서 주로 사용하는 기능을 활용할 수 있다. 

**공식홈페이지** [https://k3s.io/](https://k3s.io/)  
**K3S github** [https://github.com/k3s-io/k3s](https://github.com/k3s-io/k3s)

<br>  

## Installation
### Master Node
K3S는 위에서 언급한 것처럼 쉘스크립트 하나로 손쉽게 설치할 수 있다. 리눅스 체제 위에서 단순히 아래의 명령어만 날리면 
K3S 마스터 노드를 설치 할 수 있다.
```bash
curl -sfL https://get.k3s.io | sh -s - server
```

설치 이후 kubeconfig 파일은 /etc/rancher/k3s/k3s.yaml 에 자동으로 생성 되며 이를 환경 변수에 등록해주도록 하자!
```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

or

echo "KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> ~/.bashrc
```

<br>

### Worker Node
K3S를 여러 서버? 혹은 PC에 설치하고 클러스터로 활용하기 위해서 Worker Node에 설치하는 방법 또한 단순히 
쉘스크립트를 실행시키면 된다. 다만, 마스터 노드의 정보를 추가적으로 제공해주어 설치하면 된다.  
필요한 정보는 마스터 노드의 URL 정보와 Token을 가져오면 된다. 마스터 노드 Token은 `/var/lib/rancher/k3s/server/node-token`에서 
확인할 수 있다.

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://myserver:6443 K3S_TOKEN=XXX sh -
```

<br>

### 손쉬운 설치!
K3S를 손쉽게 설치하기 위해서 쉘 스크립트를 정리해서 [개인 깃허브](https://github.com/sunjin7725)에 정리해서 올려 놓았다.
위에 내용도 복잡하고 어렵다면, 필자의 깃허브에 있는 [k3s-installation-shell Repository](https://github.com/sunjin7725/k3s-installation-shell) 내의 쉘을 사용하면 쉽게 설치 할 수 있을 듯하다.  
해당 쉘의 설정은 개인적으로 인그레스 컨트롤러를 nginx로 활용하기 위해 `traefik 사용을 중지` 시켜 놓았고 기본 containerd 대신 
`docker 환경`을 사용하기 위해 해당 설정을 해놓았다.
```bash
## 아래 명령어를 사용하면 바로 설치 됩니다.
# Master Node
curl https://raw.githubusercontent.com/sunjin7725/k3s-installation-shell/master/master_node.sh | sh -

# Worker Node
curl https://raw.githubusercontent.com/sunjin7725/k3s-installation-shell/master/worker_node.sh | sh -
```

<br>
<br>

참고자료  
- [https://si.mpli.st/dev/2020-01-01-easy-k8s-with-k3s/](https://si.mpli.st/dev/2020-01-01-easy-k8s-with-k3s/)