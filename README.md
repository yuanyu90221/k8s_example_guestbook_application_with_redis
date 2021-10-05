# k8s_example_guestbook_application_with_redis

## 前言

今天這個章節將要來實作 [Deploying PHP Guestbook application with Redis](https://kubernetes.io/docs/tutorials/stateless-application/guestbook/) 這個任務

主要結構如下

一個 Redis instance 用來儲存 guestbook 的項目
多個網頁前端 instance 存取 Redis

## 佈署目標

1 建構一個 Redis leader

2 建構二個 Redis follower 作為 Redis leader 的 Replica

3 建立 guestbook 網頁前端

4 把 guestbook 對外開啟並且瀏覽

5 清除佈署

## 建構一個 Redis leader

### 建立 Redis leader Deployment

建立設定檔 redis-leader-deployment.yaml 如下：
```yaml=
# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-leader
  labels:
    app: redis
    role: leader
    tier: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
        role: leader
        tier: backend
    spec:
      containers:
      - name: leader
        image: "docker.io/redis:6.0.5"
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 6379
```

建立一個 Deployment 

設定名稱為 redis-leader

container image 選用 docker.io/redis:6.0.5

設定 resource cpu 使用 100m

設定 resource memory 使用 100Mi

設定 containerPort 為 6379 , 此為 container 對外的 Port

佈署指令如下：

```shell=
kubectl apply -f redis-leader-deployment.yaml
```

查看 deploy 的 Pod 使用以下指令:

```shell=
kubectl get pods
```

![](https://i.imgur.com/d9gNnw0.png)


查看 Pod 運行 log

```shell=
kubectl logs -f deployemnt/redis-leader
```

![](https://i.imgur.com/Yd0lIyl.png)

### 建立 Redis Leader Service

建立一個 Redis Leader Service 讓 guestbook 應用可以透過這個 Service 來儲存資料到 Redis

設定 redis-leader-service.yaml 如下

```yaml=
# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: v1
kind: Service
metadata:
  name: redis-leader
  labels:
    app: redis
    role: leader
    tier: backend
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
    role: leader
    tier: backend
```

建立一個 Service

設定名稱為 redis-leader

設定 selector 為篩選具有以下 label 的 Pod

條件為 app: redis, role: leader, tier: backend

設定 Service 的 Port 為 6379

設定目標 Pod 的 Port 為 6379

建構指令如下：

```shell=
kubectl apply -f redis-leader-service.yaml
```

查看 Service 狀況

```shell=
kubectl get service
```

![](https://i.imgur.com/PcYWkCZ.png)

## 建構二個 Redis follower 作為 Redis leader 的 Replica

### 建立 Redis follower Deployment

建構 redis-follower-deployment.yaml 如下：

```yaml=
# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-follower
  labels:
    app: redis
    role: follower
    tier: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
        role: follower
        tier: backend
    spec:
      containers:
      - name: follower
        image: gcr.io/google_samples/gb-redis-follower:v2
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 6379
```

設定一個 Deployment

設定名稱為 redis-follower

設定 replicas 為 2, 這個設定會建立兩個 Pod 在 ReplicaSet

設定 Pod 的 label 如下

```yaml=
app: redis
role: follower
tier: backend
```

設定 container image 使用 gcr.io/google_samples/gb-redis-follower:v2

設定 resource cpu 使用 100m

設定 resource memory 使用 100Mi

設定 containerPort 為 6379 , 此為 container 對外的 Port

佈署指令如下：

```shell=
kubectl apply -f redis-follower-deployment.yaml
```

查看 Pod 佈署狀態用以下指令

```shell=
kubectl get pods
```

![](https://i.imgur.com/4HZ04j1.png)

### 建立 Redis follower service

建立一個 Redis follower Service 讓 guestbook 應用可以透過這個 Service 讀取 Redis 資料

設定 redis-follower-service.yaml 如下

```yaml=
# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: v1
kind: Service
metadata:
  name: redis-follower
  labels:
    app: redis
    role: follower
    tier: backend
spec:
  ports:
    # the port that this service should serve on
  - port: 6379
  selector:
    app: redis
    role: follower
    tier: backend
```

建立一個 Service

設定名稱為 redis-follower

設定 selector 為篩選具有以下 label 的 Pod

條件為 app: redis, role: follower, tier: backend

設定 Service 的 Port 為 6379

建構指令如下：

```shell=
kubectl apply -f redis-follower-service.yaml
```

查看 Service 狀況

```shell=
kubectl get service
```

![](https://i.imgur.com/k5mHW3m.png)

## 建立 guestbook 網頁前端

guestbook 應用使用的是 PHP 作為 frontend, 透過 Redis follower Service 讀取資料, 透過 Redis Leader Service 寫入資料. 溝通的介面是 JSON, 而前端互動是用 jQuery-Ajax


### 建立 guestbook Deployment

建立 frontend-deployment.yaml 如下：
```yaml=
# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
        app: guestbook
        tier: frontend
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v5
        env:
        - name: GET_HOSTS_FROM
          value: "dns"
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 80
```

建立一個 Deployment

名稱設定為 frontend

設定 replias 為 3, 代表會建立 3 個 Pod

設定 selector 篩選 labels 符合條件如下：

```yaml=
app: guestbook
tier: frontend
```

設定 Pod 的 labels 如下

```yaml=
app: guestbook
tier: frontend
```

設定 container image 為 gcr.io/google_samples/gb-frontend:v5

設定環境參數 GET_HOSTS_FROM: "dns"

設定 resources cpu 為 100m

設定 resource memory 為 100Mi

設定 containerPort 為 80

佈署指令如下：

```shell=
kubectl apply -f frontend-deployment.yaml
```

查詢 Pod 狀況指令如下:

```shell=
kubectl get pods -l app=guestbook -l tier=frontend
```

![](https://i.imgur.com/ZF1QYXe.png)

### 建立 frontend Service

建立一個內部 Service 給 guestbook 

然後再使用 kubectl port-forward 指令讓這個 Service 能夠透過外部 IP 存取

建立 frontend-service.yaml 如下：

```yaml=
# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  # if your cluster supports it, uncomment the following to automatically create
  # an external load-balanced IP for the frontend service.
  # type: LoadBalancer
  #type: LoadBalancer
  ports:
    # the port that this service should serve on
  - port: 80
  selector:
    app: guestbook
    tier: frontend
```

建立一個 Service 

設定名稱為 frontend

設定 selector 篩選 labels 符合條件如下：

```yaml=
app: guestbook
tier: frontend
```

設定 Service 服務的 Port 為 80

建立佈署指令如下：

```shell=
kubectl apply -f frontend-service.yaml
```

查詢 Service 狀況

```shell=
kubectl get services
```

![](https://i.imgur.com/SWzIWvu.png)

## 把 guestbook 對外開啟並且瀏覽

透過 kubectl port-forward 指令

可以把內部服務開啟對外 Port

指令如下：

```shell=
kubectl port-forward svc/frontend 8080:80
```

指令內容為 把 service frontend 透過 host port 8080 對應到 Service port 80

接下來就可以透過 http://localhost:8080 從瀏覽器打開 guestbook

![](https://i.imgur.com/r3AHBqN.png)

### 拓展 frontend

scale up frontend 成為 5 個, 可以透過以下指令

```shell=
kubectl scale deployment frontend --replicas=5
```

執行完使用以下指令查看 Pod

```shell=
kubectl get pods
```

會發現 frontend pod 會從 3 個變成 5 個

![](https://i.imgur.com/bCBP395.png)


scale down frontend 成為 2 個, 可以透過以下指令

```shell=
kubectl scale deployment frontend --replicas=2
```

執行完使用以下指令查看 Pod

```shell=
kubectl get pods
```

會發現 frontend pod 會從 5 個變成 2 個

![](https://i.imgur.com/e0eN082.png)


### 清除佈署

```shell=
kubectl delete deployment -l app=redis
kubectl delete service -l app=redis
kubectl delete deployment frontend
kubectl delete service frontend
```
