此篇文章介紹 Kubernetes Ingress 的基本概念 & 使用方式

Overview
========

之前提到的 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) & [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 都會各自擁有一個 IP address 供存取，但這些 IP 僅有在 K8s cluster 內部才有辦法存取的到，但若要在 K8s cluster 上提供對外服務呢?

而目前可讓外部存取 K8s 上服務的方式主要有三種：

1. **Service NodePort**: 這方法會在每個 master & worker node 上都開啟指定的 port number (這樣其實造成不少資源浪費)

2. **Service LoadBalancer**: 只有在 GCP or AWS 這類的 public cloud 平台才有用的功能

3. **[Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) 就是被設計來處理這類的問題。**：即為此篇文章要談的主角


首先，在沒有 Ingress 之前，從外面到 K8s cluster 的流量會如下圖：

```
    internet
        |
  ------------
  [ Services ]
```

> 但在原有的模式下，如果是在 GCP or AWS 使用 K8s 的話，還可以搭配 LoadBalancer 的 service type 動態取得 LB 對外提供服務；但如果是自己架設 K8s，那就只能透過 Service NodePort 的方式讓使用者從外部存取運行在 K8s 上的服務。

所以 Ingress 在 K8s v1.1 之後就應運而生了，而 Ingress 其實就是一堆 rule 的集合，讓外面進來的網路流量可以正確的被導到後方的 Service，架構變成如下圖：

```
    internet
        |
   [ Ingress ]
   --|-----|--
   [ Services ]
```

在官網文件中提到 Ingress 可以負責以下工作：

- give services externally-reachable urls

- load balance traffic

- terminate SSL

- offer name based virtual hosting

看的出來可以做到的功能實在很多，非常值得好好研究一下。


----------------------------------

在使用 Ingress 之前
=================

看起來 Ingress 就是處理 inbound traffic 的銀彈，但直接在 YAML 中宣告 Ingress resource 並建立並不會有任何效果，必須搭配 [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-controllers) 才有辦法讓設定真正的生效。

> 目前 [K8s 官方 GitHub repository](https://github.com/kubernetes/ingress/tree/master/controllers) 中提供的則是以 nginx 來作為 Ingress controller。

但以下用 traefik 作為示範(**因為 traefik 與 K8s 的整合度很高，在 K8s 上 resource 的變化都可以自動反應到 Traefik 上而不需要額外的人工調整**)，首先要先選一個 node 作為 Load Balancer node，假設 node name 為 `k8s-worker01`(IP: `10.102.60.31`):

```bash
$ kubectl label node k8s-worker01 k8s-app=traefik-ingress-lb
```

建立使用 traefik 所需要的 role: 

```bash
$ cat <<EOF | kubectl create -f -
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - services
      - endpoints
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
EOF
```

接著建立 traefik ingress controller: 

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system

---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: traefik-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      hostNetwork: true
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: traefik
        name: traefik-ingress-lb
        resources:
          limits:
            cpu: 200m
            memory: 30Mi
          requests:
            cpu: 100m
            memory: 20Mi
        ports:
        - containerPort: 80
          hostPort: 80
        - containerPort: 8888
        args:
        - --web
        - --web.address=:8888
        - --kubernetes
        - --logLevel=DEBUG
      nodeSelector:
        k8s-app: traefik-ingress-lb
```

----------------------------------


佈署 Services & Ingress 範例 (Name-based)
=======================================

## traefik console (Web UI)

根據上面的示意圖，Ingress 的 inbound traffic 是要導到 Service 的，因此以下的範例將 ingress 連同 service 一同定義:

```yaml

apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: traefik-ui.10.102.60.31.nip.io
    http:
      paths:
      - backend:
          serviceName: traefik-web-ui
          servicePort: 80
```

接著透過瀏覽器瀏覽上面 `host` 關鍵字所指定的網址(**traefik-ui.10.102.60.31.nip.io**)，就可以看到如下圖所示的網頁：

![Kubernetes Ingress traefik](http://i.imgur.com/Kk2RfQk.png)


## Kubernetes Dashboard

雖然 kubectl 的功能很強大，但有個 dashboard 可以看會更容易讓資訊一目了然，可按照以下方式安裝並設定 Ingress resource。

首先先安裝最新版本的 Kubernetes dashboard 所需要的 ServiceAccount/ClusterRoleBinding/Deployment/Service：(需要 Kubernetes 1.6 以上)

> kubectl create -f https://git.io/kube-dashboard

若是 Kubernetes 1.5 或之前的版本，由於還未包含 RBAC 的功能在內，則使用下列命令安裝：

> kubectl create -f https://git.io/kube-dashboard-no-rbac

安裝好之後，來看看 K8s dashboard service 所使用的 port 為何：

```bash
# 取得 Kubernetes dashboard Service 的相關資訊：
# name: kubernetes-dashboard
# protocol & port: TCP/80
$ kubectl get svc --all-namespaces
NAMESPACE     NAME                   CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
.....
kube-system   kubernetes-dashboard   10.233.14.91    <none>        80/TCP           8m
.....
```

有了以上的資訊，我們可以宣告一個 traefik ingress resource 給 K8s dashboard 使用：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: k8s-dashboard
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: k8s-dashboard.10.102.60.31.nip.io
    http:
      paths:
      - backend:
          serviceName: kubernetes-dashboard
          servicePort: 80
``` 

當上面的 ingress resource 加入後，原本的 traefik console 就會變成下面的樣子：

![Imgur](http://i.imgur.com/zkuyBvm.png)

從 UI 就可以看出，K8s dashboard 的入口就位於 `k8s-dashboard.10.102.60.31.nip.io`。

當然也可以透過指令來查詢目前 Ingress resource 的狀態:

```bash
$ kubectl get ingress --all-namespaces
NAMESPACE     NAME             HOSTS                               ADDRESS   PORTS     AGE
kube-system   k8s-dashboard    k8s-dashboard.10.102.60.31.nip.io             80        21m
kube-system   traefik-web-ui   traefik-ui.10.102.60.31.nip.io                80        17h
```



References
==========

- [Concepts \| Kubernetes](https://kubernetes.io/docs/concepts/)

- [Ingress Resources \| Kubernetes](https://kubernetes.io/docs/concepts/services-networking/ingress/)

- [使用 NGINX 和 NGINX Plus 的 Ingress Controller 進行 Kubernetes 的負載均衡 - 歌穀穀](http://www.gegugu.com/2017/02/14/38925.html)

- [kubernetes 源碼分析之ingress（一）-趣讀](https://ifun01.com/8HAREFJ.html)

- [為什麼我不使用Kubernetes的Ingress - 每日頭條](https://kknews.cc/news/p5mgk48.html)

- [Accessing Kubernetes Pods from Outside of the Cluster - Ales Nosek - The Software Practitioner](http://alesnosek.com/blog/2017/02/14/accessing-kubernetes-pods-from-outside-of-the-cluster/)