當 Kubernetes 中(以下簡稱 k8s) 安裝完成後，接著就是要開始與其互動，此篇文章介紹透過 kubectl (k8s CLI tool) 與 k8s 進行互動。  


配置 kubeconfig
==============

[kubectl 的安裝這邊就省略，可以參考[官網的介紹](https://kubernetes.io/docs/tasks/kubectl/install/)。(其實就是把檔案下載下來，加入可執行權限，最後再將檔案放到 $PATH 目錄中即可)

安裝完 kubectl 後可以檢查版本：

```bash
# 目前所使用的版本為 v1.6.4
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"6", GitVersion:"v1.6.4", GitCommit:"d6f433224538d4f9ca2f7ae19b252e6fcb66a3ae", GitTreeState:"clean", BuildDate:"2017-05-19T18:44:27Z", GoVersion:"go1.7.5", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"6", GitVersion:"v1.6.4+coreos.0", GitCommit:"8996efde382d88f0baef1f015ae801488fcad8c4", GitTreeState:"clean", BuildDate:"2017-05-19T21:11:20Z", GoVersion:"go1.7.5", Compiler:"gc", Platform:"linux/amd64"}
```

接著要配置 kubeconfig，如此一來 kubectl 才會知道要進行存取的 k8s cluster 位於何處，以下是範例檔：(**~/.kube/config**)

```yaml
current-context: default-context

apiVersion: v1

kind: Config

preferences:
  colors: true

clusters:
- name: k8s-cluster-centos7
  cluster:
    api-version: v1
    certificate-authority: ca.pem
    server: https://10.102.60.10:1060
- name: k8s-cluster-ubuntu1604
  cluster:
    api-version: v1
    certificate-authority: ubuntu1604/ca.pem
    server: https://10.102.60.10:1062

users:
- name: root_centos7
  user:
    client-certificate: admin.pem
    client-key: admin-key.pem
- name: root_ubuntu1604
  user:
    client-certificate: ubuntu1604/admin.pem
    client-key: ubuntu1604/admin-key.pem

contexts:
- name: default-context
  context:
    cluster: k8s-cluster-centos7
    namespace: default
    user: root_centos7

- name: context-ubuntu1604
  context:
    cluster: k8s-cluster-ubuntu1604
    namespace: default
    user: root_ubuntu1604
```

我們可透過 kubeconfig 了解以下資訊：

- 存取的 k8s cluster 進入點為 **context**，透過設定 `current-context` 可以指定 kubectl 操作時預設的 context

- 使用者可以自訂多組的 cluster/user/context 並隨意搭配出不同的 context 組合，藉此存取多組 k8s cluster

- 上面的範例包含了兩組 context，分別是 `default-context` & `context-ubuntu1604`

- 若要使用 kubectl 直接對不同的 context 進行存取，可在命令中加上 `--context [CONTEXT_NAME]` 來動態變更，只要相對應的憑證有設定正確即可



查詢 Kubernetes 資源
==================

在 Kubernetes 中(以下簡稱 k8s)，所有的組件都是以 resource 的角度來看待，藉由各式各樣 resource 的組合搭配出我們想要透過 Kubernetes 所提供的服務。

## 查詢 resource 類型

而 k8s 到底有哪些 resource 可用? 可用下列指令來查詢：

```bash
$ kubectl get
You must specify the type of resource to get. Valid resource types include:

    * all
    * certificatesigningrequests (aka 'csr')
    * clusters (valid only for federation apiservers)
    * clusterrolebindings
    * clusterroles
    * componentstatuses (aka 'cs')
    * configmaps (aka 'cm')
    * daemonsets (aka 'ds')
    * deployments (aka 'deploy')
    * endpoints (aka 'ep')
    * events (aka 'ev')
    * horizontalpodautoscalers (aka 'hpa')
    * ingresses (aka 'ing')
    * jobs
    * limitranges (aka 'limits')
    * namespaces (aka 'ns')
    * networkpolicies
    * nodes (aka 'no')
    * persistentvolumeclaims (aka 'pvc')
    * persistentvolumes (aka 'pv')
    * pods (aka 'po')
    * poddisruptionbudgets (aka 'pdb')
    * podsecuritypolicies (aka 'psp')
    * podtemplates
    * replicasets (aka 'rs')
    * replicationcontrollers (aka 'rc')
    * resourcequotas (aka 'quota')
    * rolebindings
    * roles
    * secrets
    * serviceaccounts (aka 'sa')
    * services (aka 'svc')
    * statefulsets
    * storageclasses
    * thirdpartyresources
....
``` 

上面的列表即為可透過 kubectl 管理操作的 resource。

## 查詢目前的節點狀況

```bash
$ kubectl get node --show-labels
NAME           STATUS                     AGE       VERSION           LABELS
k8s-master01   Ready,SchedulingDisabled   19h       v1.6.4+coreos.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=k8s-master01,node-role.kubernetes.io/master=true
k8s-master02   Ready,SchedulingDisabled   19h       v1.6.4+coreos.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=k8s-master02,node-role.kubernetes.io/master=true
k8s-master03   Ready,SchedulingDisabled   19h       v1.6.4+coreos.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=k8s-master03,node-role.kubernetes.io/master=true
k8s-worker01   Ready                      19h       v1.6.4+coreos.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=k8s-worker01,node-role.kubernetes.io/node=true
k8s-worker02   Ready                      19h       v1.6.4+coreos.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=k8s-worker02,node-role.kubernetes.io/node=true
k8s-worker03   Ready                      19h       v1.6.4+coreos.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=k8s-worker03,node-role.kubernetes.io/node=true
```


## 查詢全部 resource

```bash
$ kubectl get all
NAME                                       READY     STATUS    RESTARTS   AGE
po/elasticsearch-logging-v1-6wg70          1/1       Running   0          18h
po/elasticsearch-logging-v1-xks3r          1/1       Running   0          18h
po/fluentd-es-v1.22-4g8x7                  1/1       Running   1          18h
.....
po/kibana-logging-3605065340-t6djz         1/1       Running   0          18h
po/kube-apiserver-k8s-master01             1/1       Running   1          18h
.....
po/kube-controller-manager-k8s-master01    1/1       Running   1          18h
.....
po/kube-proxy-k8s-master01                 1/1       Running   1          18h
.....
po/kube-scheduler-k8s-master01             1/1       Running   1          18h
.....
po/kubedns-3639568078-z1469                3/3       Running   3          18h
po/kubedns-autoscaler-2999057513-23s0d     1/1       Running   0          18h
po/kubernetes-dashboard-2039414953-nxzj1   1/1       Running   0          18h

NAME                          DESIRED   CURRENT   READY     AGE
rc/elasticsearch-logging-v1   2         2         2         18h

NAME                        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
svc/elasticsearch-logging   10.233.49.95    <none>        9200/TCP        18h
svc/kibana-logging          10.233.30.124   <none>        5601/TCP        18h
svc/kubedns                 10.233.0.3      <none>        53/UDP,53/TCP   18h
svc/kubernetes-dashboard    10.233.1.10     <none>        80/TCP          18h

NAME                          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/kibana-logging         1         1         1            1           18h
deploy/kubedns                1         1         1            1           18h
deploy/kubedns-autoscaler     1         1         1            1           18h
deploy/kubernetes-dashboard   1         1         1            1           18h

NAME                                 DESIRED   CURRENT   READY     AGE
rs/kibana-logging-3605065340         1         1         1         18h
rs/kubedns-3639568078                1         1         1         18h
rs/kubedns-autoscaler-2999057513     1         1         1         18h
rs/kubernetes-dashboard-2039414953   1         1         1         18h
```

在 k8s cluster 剛安裝完成時，其實已經安裝好很多不同的 pod/service/deployment 來提供整體的服務，從上面的結果可以看出使用了以下 resource:

1. [pods (aka 'po')](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)

2. [replicationcontrollers (aka 'rc')](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)

3. [services (aka 'svc')](https://kubernetes.io/docs/concepts/services-networking/service/)

4. [deployments (aka 'deploy')](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

5. [replicasets (aka 'rs')](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)

還有很多其他的 resource，可參考[官網的 Kubernetes Concepts](https://kubernetes.io/docs/concepts/) 取得更多資訊。

> 也可以加上 `-o wide` 來看到最詳細的輸出資訊

## 查詢指定的 resource

```bash
# 僅查詢 service & deployment 兩種 resource
$ kubectl get svc,deploy -o wide
NAME                        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE       SELECTOR
svc/elasticsearch-logging   10.233.49.95    <none>        9200/TCP        19h       k8s-app=elasticsearch-logging
svc/kibana-logging          10.233.30.124   <none>        5601/TCP        19h       k8s-app=kibana-logging
svc/kubedns                 10.233.0.3      <none>        53/UDP,53/TCP   19h       k8s-app=kubedns
svc/kubernetes-dashboard    10.233.1.10     <none>        80/TCP          19h       k8s-app=kubernetes-dashboard

NAME                          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINER(S)              IMAGE(S)                                                                                                                                    SELECTOR
deploy/kibana-logging         1         1         1            1           19h       kibana-logging            gcr.io/google_containers/kibana:v4.6.1                                                                                                      k8s-app=kibana-logging
deploy/kubedns                1         1         1            1           19h       kubedns,dnsmasq,healthz   gcr.io/google_containers/kubedns-amd64:1.9,gcr.io/google_containers/kube-dnsmasq-amd64:1.3,gcr.io/google_containers/exechealthz-amd64:1.1   k8s-app=kubedns,version=v19
deploy/kubedns-autoscaler     1         1         1            1           19h       autoscaler                gcr.io/google_containers/cluster-proportional-autoscaler-amd64:1.1.1                                                                        k8s-app=kubedns-autoscaler
deploy/kubernetes-dashboard   1         1         1            1           19h       kubernetes-dashboard      gcr.io/google_containers/kubernetes-dashboard-amd64:v1.6.1                                                                                  k8s-app=kubernetes-dashboard
```



Pod 操作
=======

在 Kubernetes 中，pod 是最小的佈署單位，可能由多個 container 一起組成，有以下幾點特性：

- 共享同樣的 IP 位址及 Port space
- 共享儲存空間 (volumes)
- 耦合性較高之應用

## deploy a pod

以下用個很簡單的範例做示範，使用以下設定檔進行佈署

```bash
$ curl https://raw.githubusercontent.com/kubernetes/kubernetes.github.io/master/docs/user-guide/walkthrough/pod-nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
    ports:
    - containerPort: 80

$ kubectl create -f https://raw.githubusercontent.com/kubernetes/kubernetes.github.io/master/docs/user-guide/walkthrough/pod-nginx.yaml
pod "nginx" created
```

## 檢視 pod 佈署狀態

```bash
# 可以看到名稱為 nginx 的 pod 已經順利產生
$ kubectl get pods -o wide
NAME                                    READY     STATUS    RESTARTS   AGE       IP               NODE
.....
nginx                                   1/1       Running   0          43s       10.233.109.68    k8s-worker02
```

## 檢視 pod 的細節

```bash
$ kubectl describe po nginx
Name:		nginx
Namespace:	kube-system
Node:		k8s-worker02/10.102.60.32
Start Time:	Fri, 16 Jun 2017 11:36:14 +0800
Labels:		<none>
Annotations:	<none>
Status:		Running
IP:		10.233.109.68
Controllers:	<none>
Containers:
  nginx:
    Container ID:	docker://4ae78737ffd73f193ca635b492d4592abcb3e221f3deef70c043645f88dbf415
    Image:		nginx:1.7.9
    Image ID:		docker-pullable://nginx@sha256:e3456c851a152494c3e4ff5fcc26f240206abac0c9d794affb40e0714846c451
    Port:		80/TCP
    State:		Running
      Started:		Fri, 16 Jun 2017 11:36:44 +0800
    Ready:		True
    Restart Count:	0
    Environment:	<none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-b4904 (ro)
Conditions:
  Type		Status
  Initialized 	True 
  Ready 	True 
  PodScheduled 	True 
Volumes:
  default-token-b4904:
    Type:	Secret (a volume populated by a Secret)
    SecretName:	default-token-b4904
    Optional:	false
QoS Class:	BestEffort
Node-Selectors:	<none>
Tolerations:	<none>
Events:
  FirstSeen	LastSeen	Count	From			SubObjectPath		Type		Reason		Message
  ---------	--------	-----	----			-------------		--------	------		-------
  30m		30m		1	default-scheduler				Normal		Scheduled	Successfully assigned nginx to k8s-worker02
  30m		30m		1	kubelet, k8s-worker02	spec.containers{nginx}	Normal		Pulling		pulling image "nginx:1.7.9"
  29m		29m		1	kubelet, k8s-worker02	spec.containers{nginx}	Normal		Pulled		Successfully pulled image "nginx:1.7.9"
  29m		29m		1	kubelet, k8s-worker02	spec.containers{nginx}	Normal		Created		Created container with docker id 4ae78737ffd7; Security:[seccomp=unconfined]
  29m		29m		1	kubelet, k8s-worker02	spec.containers{nginx}	Normal		Started		Started container with docker id 4ae78737ffd7
```

從上面可以看出 nginx container 所在的 IP 為 **10.233.109.68**，但這個 IP 我們從外面無法直接連到，必須從 master or worker nodes 才有辦法連到；而若是不想透過 master or worker nodes 來連線(**不建議這麼做**)，可以簡單啟動一個 busybox 的 container 並在裏面測試，例如：

```bash
# 啟動 busybox 做測試
$ kubectl run busybox --image=busybox --restart=Never --tty -i --generator=run-pod/v1 --env "NGINX_POD_IP=$(kubectl get pod nginx -o go-template='{{.status.podIP}}')" 

# 檢視 nginx pod IP 是否有作為環境變數傳入
/ # echo $NGINX_POD_IP
10.233.109.68

# 查詢 web 服務是否正確啟動
/ # wget -qO- http://$NGINX_POD_IP
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

# 離開 container 
/ # exit

# 移除 pod
godleon@godleon-E5570 ~ $ kubectl delete pod busybox
pod "busybox" deleted
```


Volumes 操作
===========

剛開始使用 container 都會知道，當 container 發生了 failure/restart/delete 後，裡面的所有資料也就跟著 container 不見了，所以在 docker 裏面可以把 host 的目錄掛進 container 中，
或是透過 volume container 的方式來保存資料。

而在 k8s 中也有相同的機制，稱為 [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)，我們可以透過定義 persistent volume 給 pod 使用，而當 pod 消失後，存在於 persistent volume 中的資料依舊會存在。

而使用 persistent volume 需要額外宣告 volume resource 的方式進行，以下用一個簡單的範例來說明：

```bash
$ curl https://raw.githubusercontent.com/kubernetes/kubernetes.github.io/master/docs/user-guide/walkthrough/pod-redis.yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis
    volumeMounts:
    - name: redis-persistent-storage
      mountPath: /data/redis
  volumes:
  - name: redis-persistent-storage
    emptyDir: {}

$ kubectl create -f https://raw.githubusercontent.com/kubernetes/kubernetes.github.io/master/docs/user-guide/walkthrough/pod-redis.yaml
```

更多 volume 的設定方式可參考[官方文件](https://kubernetes.io/docs/concepts/storage/volumes/)。


使用 Label 作為識別
=================

```bash
# 透過 Label 可以賦予每個 resource 不同的屬性(key/value)
# 未來就可透過 Label 來篩選 resource 來進行不同的處理
$ curl https://raw.githubusercontent.com/kubernetes/kubernetes.github.io/master/docs/user-guide/walkthrough/pod-nginx-with-label.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80

$ kubectl create -f https://raw.githubusercontent.com/kubernetes/kubernetes.github.io/master/docs/user-guide/walkthrough/pod-nginx-with-label.yaml
pod "nginx" created

# 透過 Label 進行查詢
$ kubectl get pods -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
nginx     1/1       Running   0          15s
```


使用 Deployment
==============

一般系統在執行，不會只有單一個 pod 在運作，很多時候是好幾個 container，形成好幾組 pod，相互聯繫而成，而 `deployment` 正是可以描述一個完整的系統的單位，此外，還可以指定 replica 的數量，確保 k8s cluster 上會有一定數量的 pod 提供服務。

以下用一個簡單的範例來說明：

```bash
$ curl https://raw.githubusercontent.com/kubernetes/kubernetes.github.io/master/docs/concepts/workloads/controllers/nginx-deployment.yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3   # 指定 replica 的數量為 3
  template:
    metadata:
      labels:
        app: nginx  # 設定 label，未來可透過 label 進行 resource 的搜尋
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80

$ kubectl create -f https://raw.githubusercontent.com/kubernetes/kubernetes.github.io/master/docs/concepts/workloads/controllers/nginx-deployment.yaml
deployment "nginx-deployment" created

$ kubectl get pod -l app=nginx
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-4234284026-5tt05   1/1       Running   0          24s
nginx-deployment-4234284026-6qfdn   1/1       Running   0          24s
nginx-deployment-4234284026-ckjgx   1/1       Running   0          24s

$ kubectl get deploy -l app=nginx
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           3m

# 檢視 deployment 的細節
$ kubectl describe deployment nginx-deployment 
Name:			nginx-deployment
Namespace:		kube-system
CreationTimestamp:	Mon, 19 Jun 2017 14:28:05 +0800
Labels:			app=nginx
Annotations:		deployment.kubernetes.io/revision=1
Selector:		app=nginx
Replicas:		3 desired | 3 updated | 3 total | 3 available | 0 unavailable # 有三份 replica
StrategyType:		RollingUpdate
MinReadySeconds:	0
RollingUpdateStrategy:	25% max unavailable, 25% max surge
Pod Template:
  Labels:	app=nginx # Label 設定
  Containers:
   nginx:
    Image:		nginx:1.7.9 # 目前使用的 container image 版本
    Port:		80/TCP
    Environment:	<none>
    Mounts:		<none>
  Volumes:		<none>
Conditions:
  Type		Status	Reason
  ----		------	------
  Available 	True	MinimumReplicasAvailable
  Progressing 	True	NewReplicaSetAvailable
OldReplicaSets:	<none>
NewReplicaSet:	nginx-deployment-4234284026 (3/3 replicas created)
Events:
  FirstSeen	LastSeen	Count	From			SubObjectPath	Type		Reason			Message
  ---------	--------	-----	----			-------------	--------	------			-------
  8m		8m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-4234284026 to 3
```

接著嘗試把 container image 從 `nginx:1.7.9` 改為 `nginx:1.9.1`

```bash
# 也可以使用 "kubectl edit deployment/nginx-deployment" 以編輯 YAML 的方式達成更新
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
deployment "nginx-deployment" image updated

# 重新檢視 deployment 的細節
$ kubectl describe deployment nginx-deployment 
Name:			nginx-deployment
Namespace:		kube-system
CreationTimestamp:	Mon, 19 Jun 2017 14:28:05 +0800
Labels:			app=nginx
Annotations:		deployment.kubernetes.io/revision=2
Selector:		app=nginx
Replicas:		3 desired | 3 updated | 3 total | 3 available | 0 unavailable # 同樣有三份 replica
StrategyType:		RollingUpdate
MinReadySeconds:	0
RollingUpdateStrategy:	25% max unavailable, 25% max surge
Pod Template:
  Labels:	app=nginx
  Containers:
   nginx:
    Image:		nginx:1.9.1   #container image 的版本已經變更為 nginx:1.9.1
    Port:		80/TCP
    Environment:	<none>
    Mounts:		<none>
  Volumes:		<none>
Conditions:
  Type		Status	Reason
  ----		------	------
  Available 	True	MinimumReplicasAvailable
  Progressing 	True	NewReplicaSetAvailable
OldReplicaSets:	<none>
NewReplicaSet:	nginx-deployment-3646295028 (3/3 replicas created)
# 可以看出重新使用 nginx:1.9.1 產生了一個新的 deployment 並 scale up 至 3
# 而原本使用 nginx:1.7.9 的 deployment 則被逐漸 scale down 至 0
Events:
  FirstSeen	LastSeen	Count	From			SubObjectPath	Type		Reason			Message
  ---------	--------	-----	----			-------------	--------	------			-------
  11m		11m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-4234284026 to 3
  1m		1m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-3646295028 to 1
  46s		46s		1	deployment-controller			Normal		ScalingReplicaSet	Scaled down replica set nginx-deployment-4234284026 to 2
  46s		46s		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-3646295028 to 2
  31s		31s		1	deployment-controller			Normal		ScalingReplicaSet	Scaled down replica set nginx-deployment-4234284026 to 1
  31s		31s		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-3646295028 to 3
  30s		30s		1	deployment-controller			Normal		ScalingReplicaSet	Scaled down replica set nginx-deployment-4234284026 to 0
```

最後移除 deployment：

```bash
$ kubectl delete deployment nginx-deployment 
deployment "nginx-deployment" deleted
```


使用 Service
===========

有了 **Deployment** 來描述完整系統 & replica 的配置之後，接著就是連線問題了! 而 `Service` 正是 k8s 中為 replica set 中的多組 pod 提供負載平衡的用途，藉此形成單一且不變的 endpoint 供使用。(透過 Label 的方式)

> 在這邊需要注意的是，**Service 是透過 selector(使用 Label) 提供 load balancer 的服務給 replica set of Pods，而 Deployment 同樣也有 Label 的定義，但跟 Service selector 無關**

以下用一個簡單範例說明：

```bash
$ curl https://raw.githubusercontent.com/kubernetes/kubernetes.github.io/master/docs/user-guide/walkthrough/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  ports:
  - port: 8000 # 指定 service 對外提供的 endpoint port number = 8000
    targetPort: 80
    protocol: TCP
  # 透過 selector 的方式指定進行負載平衡的 Pods
  selector:
    app: nginx


$ kubectl create -f https://raw.githubusercontent.com/kubernetes/kubernetes.github.io/master/docs/user-guide/walkthrough/service.yaml
service "nginx-service" created

$ kubectl describe svc nginx-service 
Name:			nginx-service
Namespace:		kube-system
Labels:			<none>
Annotations:		<none>
Selector:		app=nginx
Type:			ClusterIP
IP:			10.233.14.234
Port:			<unset>	8000/TCP
Endpoints:		10.233.103.205:80,10.233.109.76:80,10.233.109.77:80
Session Affinity:	None
Events:			<none>
```

因為目前從外面還無法直接連線到內部，因此我們可以透過取得一個 busybox console 來確認 service 是否運作正確：

```bash
# 取得 service 所產生的單一 endpoint
$ export SERVICE_IP=$(kubectl get service nginx-service -o go-template='{{.spec.clusterIP}}')

# 取得 service 所使用的 port (其實就是上面 YAML 定義中的 8000)
$ export SERVICE_PORT=$(kubectl get service nginx-service -o go-template='{{(index .spec.ports 0).port}}')

$ echo "$SERVICE_IP:$SERVICE_PORT"

# service 所導向的服務原本就是 HTTP，因此透過連接 http://$SERVICE_IP:$SERVICE_PORT 來檢查結果
$ kubectl run busybox  --generator=run-pod/v1 --image=busybox --restart=Never --tty -i --env "SERVICE_IP=$SERVICE_IP,SERVICE_PORT=$SERVICE_PORT"
u@busybox$ wget -qO- http://$SERVICE_IP:$SERVICE_PORT # Run in the busybox container
u@busybox$ exit # Exit the busybox container

# 確認無誤之後就可以刪除 busybox pod 了
$ kubectl delete pod busybox # Clean up the pod we created with "kubectl run"
```


為 pod 額外自訂 Health Checking
==============================

 在 k8s 中，當 pod 中的 container 因為某些原因掛掉後，k8s 會自動幫使用者重新啟動一個新的代替，既然這樣，為什麼還需要額外為 pod 自動 Health Checking 呢?

 原因是因為 k8s 所監控的僅僅是在非常低的層級，只有當 container process 已經不運作了，才會進行修復的動作；但有時候可能因為程式沒有寫好，導致於系統處於 dead lock 的狀態，此時 container process 也還會是 running 狀態，因此 k8s 不會有任何動作，當然系統此時也無法提供正常服務。

 因此 k8s 提供了以下三種方式可以讓使用者自訂 Health Checking 的條件，確保服務可以持續正常運作:

1. **[HTTP Health Checks](https://kubernetes.io/docs/user-guide/liveness/)**: 透過持續的 polling HTTP 的服務取得 response code 來確認服務是否正常

2. **[Container Exec](https://kubernetes.io/docs/user-guide/liveness/)**: 透過在 container 中執行特定指令，若是回傳 status 0 則表示服務正常

3. **TCP Socket**: 檢查 tcp 特定 port number 是否回應來判斷服務是否正常

以下是一個 HTTP health check 的範例：

```bash
$ curl https://raw.githubusercontent.com/kubernetes/kubernetes.github.io/master/docs/user-guide/walkthrough/pod-with-http-healthcheck.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-http-healthcheck
spec:
  containers:
  - name: nginx
    image: nginx
    # 使用 livenessProbe 關鍵字宣告自訂的 health checking
    livenessProbe:
      # 透過 HTTP GET 的方式持續監控特定位址是否可正常回應
      httpGet:
        path: /_status/healthz
        port: 80
      # 第一次進行 polling 前的 delay(second)
      initialDelaySeconds: 30
      timeoutSeconds: 1
    ports:
    - containerPort: 80
```

而下面這個則是 tcp socket health check 的範例：

```bash
$ curl https://raw.githubusercontent.com/kubernetes/kubernetes.github.io/master/docs/user-guide/walkthrough/pod-with-tcp-socket-healthcheck.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-tcp-socket-healthcheck
spec:
  containers:
  - name: redis
    image: redis
    # 使用 livenessProbe 關鍵字宣告自訂的 health checking
    livenessProbe:
      # 指定使用 tcp socket health checking
      tcpSocket:
        port: 6379
      # 第一次進行 polling 前的 delay(second)
      initialDelaySeconds: 30
      timeoutSeconds: 1
    ports:
    - containerPort: 6379
```

References
==========

- [Concepts \| Kubernetes](https://kubernetes.io/docs/concepts/)

- [Kubernetes 最小部署單位 Pod](https://tachingchen.com/tw/blog/Kubernetes-Pod/)

- [GKE 系列教學 (二) – 簡介Pod的網路機制 \| GCP 雲端服務頂級合作夥伴 LIVEhouse.in](https://blog.gcp.expert/gke-k8s-pod-network/)

- [kubectl Cheat Sheet \| Kubernetes](https://kubernetes.io/docs/user-guide/kubectl-cheatsheet/)

- [Persistent Volumes \| Kubernetes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

- [Volumes \| Kubernetes](https://kubernetes.io/docs/concepts/storage/volumes/)

- [Deployments \| Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)