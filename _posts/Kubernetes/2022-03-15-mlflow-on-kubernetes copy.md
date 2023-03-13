---
title: MLFlow on Kubernetes
categories: [kubernetes]
tags: [kubernetes, mlflow, mlops]
comments : true
---

# MLFlow on Kubernetes

2022년 3월 MLFlow를 회사 쿠버네티스에 올린 경험을 정리합니다~

# MLFlow

## MLFlow 란?

MLFlow는 머신러닝의 실험이나 배포를 쉽게 관리할 수 있는 오픈 소스이다.

기존에 MLFlow가 없던 시절에는 연구일지에다가 Loss 측정해서 올리고 해당 베스트 모델들을 폴더에 구조화해서 정리하는 등 아주 어려운 그런 작업들을 진행했었다.

그런식으로 정리하다보니 Versioning이라든가 모델을 관리하는 측면에서 누락되는 부분이 생기곤 했었다.

하지만 MLFlow는 이러한 parameter나 metric들을 정리해서 가지고 있을 수 있고 모델을 가지고 있어 나중에 배포에서도 활용할 수 있도록 나온 오픈 소스이다.

## MLFlow 아키텍쳐

MLFlow는 간단하게는 

```bash
pip install mlflow
```

를 통해서 설치하고 

```bash
mlflow ui
```

명령어를 실행시키면 [localhost:5000](http://localhost:5000) 에서 쉽게 활용할 수 있다.

이는 간단하게 로컬 환경에서 활용할 경우고, 서버를 통해서 여러사람들이 다같이 활용하기 위해서는 MLFlow의 아키텍쳐에 대해서 약간의 이해가 필요하다.
아래는 MLFlow의 간단한 아키텍쳐를 보여준다.

![Untitled](/assets/img/post/mlflow.png)

MLFlow를 구성하기위해서는 크게 세가지 요소를 필요로 한다. 

1) 가장 중요한 머신러닝의 Parameter나 Metric, log, model 등을 tracking 해줄 `MLFlow Server`  
2) parameter 등과 같은 로그성 데이터를 관리할 `DB`  
3) 학습 모델과 활용 데이터셋을 관리한 `Artifact Storage`  

각각의 요소들이 모두 충족해야지만 정상적으로 작동한다!!!

## Persistence Volume 생성하기

MLFlow를 만들 때 gcp 나 aws 를 활용했다면 kubernetes의 `storage class` 를 활용해서 동적 프로비저닝(Dynamic Provisioning)을 활용하여 따로 PV를 생성하지 않고 PVC만 만들어서 사용할 수 있었을 것이다. 하지만 온프레미스 환경에서 사용하려다보니 현재는 kubernetes에서 local storage에 대한 동적 프로비저닝을 지원하지 않기 때문에 따로 PV를 생성하여 `DB`와 `Artifact Storage` 의 저장공간으로 활용하였다. DB에는 10기가의 볼륨, Artifact Storage에는 200기가의 볼륨을 할당해 주었다.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: minio-volume
spec:
  capacity:
    storage: 200Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  local:
    path: /mnt/mlflow-volume/minio-volume
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s-node2
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: db-volume
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  local:
    path: /mnt/mlflow-volume/db-volume
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s-node2
```

야물을 보면 nodeAffinity의 노드를 직접 정해주었는데 따라 하실 분들은 해당 사항을 고려해서 사용할것!

## DB on Kubernetes

활용하는 DB는 `postgresql`을 사용했다.

```yaml
# postgres.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mlflow-postgres-config
  labels:
    app: mlflow-postgres
data:
  POSTGRES_DB: mlflow_db
  POSTGRES_USER: mlflow_user
  POSTGRES_PASSWORD: mlflow_pwd
  PGDATA: /var/lib/postgresql/mlflow/data
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mlflow-postgres
  labels:
    app: mlflow-postgres
spec:
  selector:
    matchLabels:
      app: mlflow-postgres
  serviceName: "mlflow-postgres-service"
  replicas: 1
  template:
    metadata:
      labels:
        app: mlflow-postgres
    spec:
      containers:
      - name: mlflow-postgres
        image: postgres:11
        ports:
        - containerPort: 5432
          protocol: TCP
        envFrom:
        - configMapRef:
            name: mlflow-postgres-config
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
        volumeMounts:
        - name: db-pvc
          mountPath: /var/lib/postgresql/mlflow
volumeClaimTemplates:
  - metadata:
      name: db-pvc
      namespace: mlflow
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
      storageClassName: standard
      volumeMode: Filesystem
      volumeName: db-volume
---
apiVersion: v1
kind: Service
metadata:
  name: mlflow-postgres-service
  labels:
    svc: mlflow-postgres-service
spec:
  type: NodePort
  ports:
  - port: 5432
    targetPort: 5432
    protocol: TCP
  selector:
    app: mlflow-postgres
```

statefulset의 활용되는 이미지가 `postgres:11`로 되어있는데 해당 부분은 [dockerhub](http://hub.docker.,com) 에서 적절한 버젼을 맞추어 활용하자!

💥 PVC의 볼륨이름과 이전에 생성해 놓았던 PV의 이름을 잘 매칭하자!

## Aritifact Storage on Kubernetes

Artifact Storage는 `minio`를 사용했다.

해당 내용은 좀 더 공부 해볼까 한다. 활용도가 좀 있는 듯 하여 따로 공부해보면 좋을 듯하다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mlflow-minio
spec:
  selector:
    matchLabels:
      app: mlflow-minio
  template:
    metadata:
      labels:
        app: mlflow-minio
    spec:
      # 아래에서 만들어 둔 pvc를 볼륨으로 적용
      volumes:
      - name: minio-pvc
        persistentVolumeClaim:
          claimName: minio-pvc
      containers:
      - name: mlflow-minio
        # 사용할 컨테이너 이미지
        image: minio/minio:RELEASE.2020-07-27T18-37-02Z
        imagePullPolicy: IfNotPresent #Always
        args:
        - server
        - /data
        # /data를 볼륨에 마운트
        volumeMounts:
        - name: minio-pvc
          mountPath: '/data'
        # minio에 접속하기 위한 키 값
        env:
        - name: MINIO_ACCESS_KEY
          value: "minio"
        - name: MINIO_SECRET_KEY
          value: "minio123"
        # 9000 포트로 노출
        ports:
        - containerPort: 9000
---
apiVersion: v1
kind: Service
metadata:
  name: mlflow-minio
spec:
  type: LoadBalancer
  ports:
  - port: 9000
    targetPort: 9000
    protocol: TCP
    name: http
  selector:
    app: mlflow-minio
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 200Gi
  storageClassName: standard
  volumeMode: Filesystem
  volumeName: minio-volume
```

Deployment의 활용되는 이미지가 `minio/minio:RELEASE.2020-07-27T18-37-02Z` 되어있는데 해당 부분은 [dockerhub](http://hub.docker.,com) 에서 적절한 버젼을 맞추어 활용하자!

💥 PVC의 볼륨이름과 이전에 생성해 놓았던 PV의 이름을 잘 매칭하자!

## MLFlow Server on Kubernetes

이제 위에 올려놓은 두 `storage`를 연결해서 `mlflow server`를 배포한다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mlflow-deployment
spec:
  selector:
    matchLabels:
      app: mlflow-deployment
  template:
    metadata:
      labels:
        app: mlflow-deployment
    spec:
      containers:
      - name: mlflow-deployment
        # 미리 생성해 둔 mlflow image를 사용합니다.
        image: pdemeulenaer/mlflow-server:542
        imagePullPolicy: IfNotPresent #Always
        # args로 배포할 포트와 호스트, backend-store등을 넣습니다.
        args:
        - --host=0.0.0.0
        - --port=5000
        - --backend-store-uri=postgresql://mlflow_user:mlflow_pwd@${postgresql_server ip:port}/mlflow_db
        - --default-artifact-root=s3://mlflow/
        - --workers=2
        # 환경 변수로 minio에 대한 정보를 넣습니다.
        env:
        - name: MLFLOW_S3_ENDPOINT_URL
          value: http://${minio_server ip:port}/
        - name: AWS_ACCESS_KEY_ID
          value: "minio"
        - name: AWS_SECRET_ACCESS_KEY
          value: "minio123"
        # 5000번 포트로 노출합니다.
        ports:
        - name: http
          containerPort: 5000
          protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: mlflow-service
spec:
  type: LoadBalancer
  ports:
    - port: 5000
      targetPort: 5000
      protocol: TCP
      name: http
  selector:
    app: mlflow-deployment
```

중간중간 postgresql의 서버 ip:port / minio의 서버 ip:port를 잘 확인하고 배포 해주시면 됩니다.

## 확인하기

```bash
kubectl get all -n {mlflow namespace}
```

명령어를 통해 접속을 확인합니다!