---
title: Helm 설치하기
categories: [kubernetes]
tags: [kubernetes, mlflow, mlops, devops, ingress]
comments: true
---

# Helm 설치하기
## Helm 이란?
helm은 Kubernetes의 패키지 매너저이다. redhat 기반 os의 yum 이나 debian 계열의 apt와 같은 개념이라고 보면 될것 같다.  
이런 helm을 쓰면 Kubernetes의 설치할 수 있는 패키지(Rancher에서는 App이라고도 한다)를 쉽게 설치 할 수 있다.

helm은 `Docker HUB` 같은 공개 `Repository`로 [Aritifact HUB](https://artifacthub.io/){:target="_blank"}를 사용하고 추가적으로 사용자들이 `git` 이나 `따로 구축한 Repository`에 `Chart`라고 하는 패키지들을 `Local repo`에 추가하여 활용한다.

## Installation
Helm은 손쉽게 [공식 홈페이지](https://helm.sh/){:target="_blank"} 에서 제공하는 명령어 한줄로 설치 할 수 있다.
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | sh -
```

<br>

## 활용방법
### helm repo add
`helm repo add`를 사용하면 다른 이들이 GitHub와 같은 repository에 다른 이들이 업로드 해놓은 helm 차트를 활용할 수 있다.
```bash
helm repo add ${REPO-NAME} ${REPO-URL}

ex)
helm repo add bitnami https://charts.bitnami.com/bitnami
```

### helm search
`helm search`를 사용하면 내가 로컬에 추가한 repo나 hub에서의 패키지 리스트를 볼 수 있다.
```bash
# 해당 repo의 전체 리스트를 확인한다.
helm search ${REPO-NAME}

# 해당 repo에서 keyword에 해당하는 Chart를 검색한다.
helm search ${REPO-NAME} ${SEARCH-KEYWORD}

ex)
helm search hub
helm search hub jenkins
```

### helm install 
`helm install`을 사용하면 해당 차트를 Kubernetes에 설치한다.
```bash
helm install ${RELEASE-NAME} ${REPO-NAME}/${CHART-NAME}

ex)
helm install my-jenkins hub/jenkins
```

### helm uninstall
`helm uninstall`을 사용하면 설치한 패키지를 삭제한다.
```bash
helm uninstall ${RELEASE-NAME}

ex)
helm uninstall my-jenkins
```

이외에도 pull 을 사용해서 차트 폴더를 가져오고 `values.yaml`을 활용해서 helm 차트에 다양한 config 내용을 바꾸어준다던가,
다양한 옵션 명령어를 활용하여 `특정 네임스페이스`에 패키지를 설치한다던가 하는 다양한 방법이 있다.  
정말 다양한 방법이 많아 자세한 부분들은 생략하고 필요한 부분이 있으시다면 `댓글 부탁드립니다~`

<br>
<br>

참고자료  
- [https://kim-dragon.tistory.com/269](https://kim-dragon.tistory.com/269)