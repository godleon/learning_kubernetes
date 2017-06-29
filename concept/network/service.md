此篇文章介紹 Kubernetes Service 的基本概念 & 使用方式

Overview
========

在 K8s cluster 中，pod 可能會因為某些原因停止運作 or 重新產生....等等，並非永遠可以維持運行的狀態，特別是透過 [Replication Controller](https://kubernetes.io/docs/user-guide/replication-controller) 介入時，為了維持使用者設定的 desired status，pod 會被動態的新增 or 刪除，每次取得的 IP 也都不會相同，在這樣的情況下，pod 在運作時的相依狀況要如何處理呢? 

在 K8s 中，給出了 **Service** 這個答案。

什麼是 Service? Service 可以視為一群 pod 的抽象層(透過 [Label](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors) 作為群組的依據)，隱藏的 pod 的連接細節，透過指定的 expose 的入口，與其他 K8s object 進行互動，藉由這樣的設計，讓 object 之間的 coupling 程度降低。

> 對於 K8s-native 的應用程式，K8s 提供了 Endpoint API 讓程式可以主動通知 pod 的變動；而 non-k8s-native 的應用程式，K8s 則提供了 virtual-IP-based bridge 來提供網路流量的導引

----------------------------------

建立 Service
===========

在建立 service 之前，先建立以下的 deployment：

```bash
$ cat <<EOF | kubectl create -f -
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: myweb-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: myweb
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
EOF
```

上面的 deployment 包含了幾個重點：

1. 使用 nginx container，每個 pod 所開放的 port 為 TCP 80

2. 3 份 replica，因此會產生 3 個 pod

3. Label 設定為 `app=MyWeb`

接著要設計一個可以用來銜接上面 pods 的 Service，可以使用以下定義：

```bash
# 建立 service
# 透過 Label selector 尋找 app=myweb 的 pod
# 產生一個 cluster IP，對外開啟 TCP 8080，將流量導至 pod TCP 80
$ cat <<EOF | kubectl create -f -
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: myweb
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
EOF

# 檢視 service 狀態
$ kubectl describe svc my-service 
Name:			my-service
Namespace:		default
Labels:			<none>
Annotations:		<none>
Selector:		app=myweb
Type:			ClusterIP
IP:			10.233.33.1
Port:			<unset>	8080/TCP
Endpoints:		10.233.103.196:80,10.233.103.74:80,10.233.109.75:80
Session Affinity:	None
Events:			<none>
```

從上面可以看出，K8s 額外為這個 service 產生了一個 cluster IP(**10.233.33.1**)，並透過 Label selector(**app=myweb**)，連結到了3個 pod(**10.233.103.196:80,10.233.103.74:80,10.233.109.75:80**)

> 此時 service 其實扮演的就是一個 load balancer 的角色，協助將網路流量分散到多個不同的 pod 上


接著啟用一個 busybox pod 進行驗證：

```bash
$ kubectl run busybox --image=busybox --restart=Never --tty -i --generator=run-pod/v1
/ # wget -qO- http://10.233.33.1:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
.....
</head>
<body>
<h1>Welcome to nginx!</h1>
....
</body>
</html>
```

> 也可以從任何一個 master/worker node 直接測試，但若是無法直接連線到 master/worker node，那就只能用類似上面的方式來測試

## 建立 multi-port service

使用 multi-port 是很平常的事情，例如同時在 web 上提供 HTTP & HTTPS 的服務，可以使用以下範例來定義 service：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
    selector:
      app: myweb
    ports:
      - name: http
        protocol: TCP
        port: 8080
        targetPort: 80
      - name: https
        protocol: TCP
        port: 10443
        targetPort: 443
```


----------------------------------


Virtual IPs and service proxies
================================

在 K8s 中，service 負責處理 Layer 4(TCP/UDP over IP) 的流量，而 K8s v1.1 之後，則有 **Ingress** API 負責處理 service 在 Layer 7(HTTP) 的工作。

每一個在 K8s cluster 的 node 都會執行一個 kube-proxy 的 process，負責為 ExternalName 以外的 object 提供 virtual IP 的服務，在 Kubernetes v1.2 之後，整個網路的動態調整變成由 iptables 服務來負責處理，如下圖：

![services-iptables-overview](http://www.webpaas.com/usr/uploads/2015/10/1817708548.png)

在此模式下，當 service 被建立後，iptables 服務會自動制定所需要的規則，為了就是將網路的流量導到 Label selector 所選取到的 pod 上，這樣的作法比起 v1.1 之前使用的 userspace proxy 效能更好。


----------------------------------


Publishing Services
===================

預設情況中，service 建立之後，只有在 K8s 內部才連的到(使用 **ClusterIP**)，若要將 service 提供給 K8s 外部供存取，就勢必得選擇其他的 service type，目前 K8s 提供以下幾種 service type: 

## ClusterIP

這是 K8s service 預設的 service type，藉由在 K8s 內部產生一個 virtual IP，並透過每個 node 上的 kube-proxy 搭配 iptables service 來達成 load balancer for pods 的目的。

## NodePort

NodePort 會在每個 node 上開啟相同 port number 供連結，因此使用者可以透過 `<NodeIP>:<NodePort>` 的方式訪問到位於 service 後端的 pod，預設的 port range 為 **30000-32767**。

我們可以透過以下指令定義一個 type 為 NodePort 的 service：

```bash
# 建立一個 type=NodePort 的 service
$ cat <<EOF | kubectl create -f -
kind: Service
apiVersion: v1
metadata:
  name: my-service-nodeport
spec:
  type: NodePort
  selector:
      app: myweb
  ports:
  - port: 8080
    targetPort: 80
    protocol: TCP
    name: http
EOF

# 檢視 service 細節
# 存取 service 的方式如下：
# 1. 在 K8s 內部與 10.233.0.121:8080 通訊
# 2. 在外部與 <Any Node's Public IP>:30909 通訊
$ kubectl describe svc my-service-nodeport 
Name:			my-service-nodeport
Namespace:		default
Labels:			<none>
Annotations:		<none>
Selector:		app=myweb
Type:			NodePort
IP:			10.233.0.121
Port:			http	8080/TCP
NodePort:		http	30909/TCP
Endpoints:		10.233.103.196:80,10.233.103.74:80,10.233.109.75:80
Session Affinity:	None
Events:			<none>
```

完成後我們就可以使用指定的 NodePort 進行連線：

```bash
# 其中 10.102.60.21 是 master node 的 public ip
# 也可以替換成其他 master node 或是 worker node
$ curl http://10.102.60.21:30909
<!DOCTYPE html>
<html>
.....
<body>
<h1>Welcome to nginx!</h1>
...
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## LoadBalancer

這必須要有 service provider(ex: AWS, GCP)來提供 Load Balancer 呼叫 & 建立的服務(也可以自己做)，以下是定義方式：

```yaml
# 內部的 cluster IP 為 10.0.171.239
# 呼叫 Load Balancer(位於 78.11.24.19) service 提供一個 virtual LB 來處理網路流量
# (不一定每一家 service provider 都支援直接對 LB IP 直接 request)
# 指定 146.148.47.155 為 virtual LB IP
kind: Service
apiVersion: v1
metadata:
  name: my-service-lb
spec:
  type: LoadBalancer
  selector:
    app: myweb
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
      nodePort: 30061
  clusterIP: 10.0.171.239
  loadBalancerIP: 78.11.24.19
status:
  loadBalancer:
    ingress:
      - ip: 146.148.47.155
```

## External IPs

如果 K8s cluster 中的 node 擁有可以從外部存取的其他 IP，就可以透過 external IP 的方式指定給 service。

> external IP 並非由 K8s 所管理，而是由 cluster manager 所負責

external IP 可藉由以下方式來定義：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service-extip
spec:
  selector:
    app: myweb
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
  externalIPs: 
    - 80.11.12.10
```


----------------------------------

References
==========

- [Concepts \| Kubernetes](https://kubernetes.io/docs/concepts/)

- [Services \| Kubernetes](https://kubernetes.io/docs/concepts/services-networking/service/)

- [探索Kubernetes的网络原理及方案](https://mp.weixin.qq.com/s/oPX8DW6Ek5gq0b9IH2cg8g)

- [标签 micro-service 下的文章 - Container Based Platform as a Service](http://webpaas.com/index.php/tag/micro-service/)
