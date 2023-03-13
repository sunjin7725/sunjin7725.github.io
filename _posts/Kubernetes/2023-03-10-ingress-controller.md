---
title: Ingress Controller 설치하기
categories: [kubernetes]
tags: [kubernetes, mlflow, mlops, devops, ingress]
comments : true
---

# Ingress Controller 설치하기
## Ingress란?
![Ingress](/assets/img/post/ingress.png)
- 서비스 앞에 두는 로드밸런서로 쿠버네티스 클러스터로 들어온 외부 접근을 클러스터 내부 서비스로 라우팅함.
- `사용자가 정의한 규칙`(Prefix 규칙 등)에 따라 외부 접근을 라우팅함.

<br>

## Ingress Controller 란?
**Ingress를 위해서는 Ingress Controller가 반드시 필요하다!**
![Ingress Controller](/assets/img/post/ingress_controller.png)
- 쿠버네티스 클러스터의 `Ingress를 관리`함
- Ingress는 Ingress Controller가 인식할 수 있는 `annotation을 정의`하여 동작함.
  - 필자는 본 글에서 `NGINX` Ingress Controller를 사용할 예정이다. 많은 사람들에게 익숙하기도 하고 많이들 사용한다고 한다.
  - 이 외에도, Kong이나, Traefik 등을 사용할 수 있다.  

**P.S.**
K3S에서는 기본으로 Traefik을 채택하여 제공하고 있지만, 필자는 [K3S 설치하기](/posts/k3s-installation)에서 Traefik 설정을 끄고 설치하였다.

<br>

## Installation
### 기본 설치 방법
Ingress Controller NGINX의 설치는 굉장히 간단하다. Helm 을 사용하여 설치하면 된다.
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace
```

우리는 NGINX 의 Ingress Controller를 사용하기 때문에 각 서비스에 연결될 Ingress를 작성할 때
annotation에 `kubernetes.io/ingress.class: "nginx"`를 추가해서 ingress 유형을 알려주자!!!'

### 손쉬운 설치!
Ingress Controller NGINX를 쉘파일 하나로 설치하고 싶으시다면 [개인 깃허브](https://github.com/sunjin7725/ingress-controller-installation-shell)에 올려놓았습니다.
```bash
# 아래 명령어를 사용하면 바로 설치 됩니다.
curl https://raw.githubusercontent.com/sunjin7725/ingress-controller-installation-shell/master/ingress-nginx.sh | sh -
```

<br>
<br>

참고자료  
- [https://kim-dragon.tistory.com/139](https://kim-dragon.tistory.com/139)
- [https://velog.io/@dojun527/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EC%9D%B8%EA%B7%B8%EB%A0%88%EC%8A%A4Ingress](https://velog.io/@dojun527/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EC%9D%B8%EA%B7%B8%EB%A0%88%EC%8A%A4Ingress)
- [https://jangcenter.tistory.com/127](https://jangcenter.tistory.com/127)