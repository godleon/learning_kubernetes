此篇文章嘗試從非常 high level 的角度來概觀 Kubernetes

Components
==========

K8s 是有很多個不同的元件所組成，以下用一張圖來介紹 K8s 的架構：

![K8s Architecture](http://www.imotif.net/wp-content/uploads/2016/11/Canvas-2.jpg)

從圖中可以看出在不同的 node 上面有著不同的元件分佈：

## Master node

- **API Server**: 身為 K8s 的 control plane，提供一組 REST APIs，透過 API 可使用 Kubernetes 上所提供的功能

- **Scheduler**: 用來決定 workload 要執行在哪一個 worker node 上

- **etcd**: 用來存放 K8s 運作資訊的 key/value 資料庫

- **Controller Manager**: 用來管理 K8s cluster 相關的 resource，分成4個部份，分別是 **Node Controller**(管理 worker nodes), **Replication Controller**(管理 replica set), **Endpoints Controller**(管理 endpoint 物件，可能由 service or pod resource 產生) 以及 **Service Account & Token Controllers**(管理認証相關事宜)

- **pod master**: 管理 pod 的生命周期

- **systemd**: 目前所有 K8s 的元件服務都是設計成由 systemd 管理

- **Supervisord**: 用來確保 kubelet & docker 持續的正常運作

- **docker**: container run time

- **fluentd**: 管理 log 之用


## Worker node

- **kubelet**: 主要作為 node agent，用來跟 master node 溝通，並監控被分配到該 node 上的 pod，主要負責以下工作：
  - 掛載 pod 所需要的 volume
  - 下載 pod secret
  - 運行 container
  - 定期進行 container 的健康檢查
  - 回報 pod 的狀態，必要時建立新的 pod

- **kube-proxy**: 用來維護網路的通訊以及封包的遞送

- **systemd**: 目前所有 K8s 的元件服務都是設計成由 systemd 管理

- **Supervisord**: 用來確保 kubelet & docker 持續的正常運作

- **docker**: container run time

- **fluentd**: 管理 log 之用


## client

- **kubectl**: 用來與 K8s cluster 互動的 CLI 工具


---------------------------------


Kubernetes Objects
==================

Kubernetes Objects 是一種用來用來描述 K8s 系統運作狀態的一種概念，舉例來說，Kubernetes Objects 可以被描述成：

- 有哪些 containerized application 正在運行?

- 承上，在哪些 worker nodes 上運行?

- 這些 application 可用的 resource 為何?

- 套用在這些 application 上的規則，例如：restart policy, fault-tolerance

每個 Object 都包含了 spec & status 兩個部份，其中 spec 是用來描述 object 要以什麼樣的形式(規格 or 內容)呈現，而 status 則是用來描述這個 object 希望由 K8s 所維持的狀態(**desired state**)。

以下舉個 deployment 的例子來說明：(可透過 YAML or JSON 呈現)

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

上是一個 deployment object 的定義，其中的 spec 部份則清楚的定義這個 deployment 的規格(使用 nginx:1.7.9 作為 image 來運行 container)；而 status 則是希望有 3 份 replica，因此 K8s 會協助維持 3 份 replica 的狀態。

在前一篇文章提到，在 K8s 上各式各樣(**kind**)的 resource，其實就是以一個一個 object 存在於 K8s ecosystem 中，而每個 object 都各自擁有自己的 spec & status。


## Name

每個在 K8s 上的 object 都可以用 Name or UID 來識別，而在同一時間，每一種 **kind(resource type)** 的 object name 不能重複，例如：不能有兩個 name 同為 nginx 的 pod。

而每個 resource 都可以透過 API 進行存取，以 pod 為例，resource url 可能像是 `/api/v1/pods/some-name`。當然，為 object 取名也是有學問的，若是要研究此部份可參考[官方文件](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/identifiers.md)。


## Namespace

假設有很多不同的使用者同時使用相同的 K8s cluster，K8s 提供了 `namespace` 讓一個 K8s cluster 上可驅分出多個不同的 virtual cluster。

大部分的 resource 都會隸屬於某個 namespace，不同 namespace 的 resource name 可以相同，每個使用者可以自訂一個自己的 namespace，並在此 namespace 中自訂自己的 resource。

> 有些特殊的 resource(e.g., node, persistent volume) 不屬於任何 namespace

而管理者也可以透過 [resource quota](https://kubernetes.io/docs/concepts/policy/resource-quotas/) 來定義每個 namespace 可使用的實體資源總量，亦可以 namespace 為單位設定 ccontrol policy。

而若是要在同一個 namespace 為不同的 resource 進行分門別類，則可搭配 [Label](https://kubernetes.io/docs/user-guide/labels) 來完成。

此外，namespace 其實也是 K8s resource 的一種，只是透過不同 resource 的搭配，可以達成資源隔離的效果：

```bash
$ kubectl get ns
NAME          STATUS    AGE
default       Active    4d
kube-system   Active    4d
```

當 K8s 安裝完成後，至少會有以上兩個 namespace；若沒有指定 namespace 的 resource 則會被歸類在 default；而 **kube-system** 這個 namespace 內的 resource 則都是屬於 K8s 系統本身所擁有的。


## Labels and Selectors

Label 是個 key/value 的組合，使用者可以隨意為 object 賦予附加上自訂的 label(一個 or 多個)，達成類似群組 or 分類的效果，並透過 selector 來進行 object 的選取。

以下是 Label 的範例：

- "**release**" : "stable", "**release**" : "canary"

- "**environment**" : "dev", "**environment**" : "qa", "**environment**" : "production"

- "**tier**" : "frontend", "**tier**" : "backend", "**tier**" : "cache"

- "**partition**" : "customerA", "**partition**" : "customerB"

- "**track**" : "daily", "**track**" : "weekly"

那 selector 如何使用呢? 以下也有幾個 cli 範例：

```bash
$ kubectl get pods -l environment=production,tier=frontend

$ kubectl get pods -l 'environment in (production),tier in (frontend)'

$ kubectl get pods -l 'environment in (production, qa)'

$ kubectl get pods -l 'environment,environment notin (frontend)'
```

若是透過 YAML 宣告：

```yaml
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
```

最後，也可以指定 schedule pod 到特定的 node。(使用 [nodeSelector](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/))



References
==========

- [Concepts \| Kubernetes](https://kubernetes.io/docs/concepts/)