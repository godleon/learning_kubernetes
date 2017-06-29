相關組件
=======

以下是一個 Kubernetes Cluster 運作的示意圖：

![Kubernetes Concepts](http://read.html5.qq.com/image?src=forum&q=5&r=0&imgflag=7&imageUrl=http://mmbiz.qpic.cn/mmbiz/A1HKVXsfHNlp7AL9YFV5qLicaaGmBKIJGOvBacjP6Zs7PlWNysKbXPV44Adkv31w7lQICeUYzUJCWBXwIEW13Aw/0?wx_fmt=png)

## Pod

同一個 pod 的 container 共享 network namespace，因此可以透過 localhost 互通，一般生命周期短暫不長。

而為了確保 container 所使用的資料可以保存，每個 pod 都會使用外部的 volume 作為資料儲存用。

## Label

Kubernetes 中的 Label 跟一般的 Label 不太一樣，是以 key-value 的型式的存在，並用來標記 pod 之用，藉此來定義 pod 的屬性。

當 pod 有了屬性之後，就可以透過 selector 來選擇特定的 pod，來與 **Service** or **Replication Controller** 進行連結。

## Replication Controller

Replication Controller 的用途在於確保任何時間內都會有 **<font color='red'>指定數量</font>** 的 pod copy 執行中。

> Replication Controller 與 pod 的關係是一對一的（不是指 copy），意思是若是不同的 pod 也有多個 copy 的需求，則會有另外一個 Replication Controller 來協助處理

Replication Controller 的運作行為如下：

- 監控 pod 的運作狀況，若是沒有 response，就會自動換掉，確保可以保持一開始所指定的 copy 數量

- 若沒回應的 pod 突然回應，但新的 pod 又產生出來時，Replication Controller 就會自動其中一個終止

- 運作中加大 copy 的數量，Replication Controller 會立刻啟動新的 pod

- 減少 copy 的數量後的行為同上

> 從上面可以知道，要建立 Replication Controller，必須要指定 `pod` & `label` 兩種資訊

## Service

Service 是為了讓 pod 可以被存取而抽象出來的一個概念，為了達到此目的，需要具備幾個條件：

1. DNS 服務 (沒人會想用 IP 存取服務)

2. 網路服務 (kube-proxy 在 pod 上層提供了一個 Load Balance 的服務)

![Kubernetes - Service](http://read.html5.qq.com/image?src=forum&q=5&r=0&imgflag=7&imageUrl=http://mmbiz.qpic.cn/mmbiz/A1HKVXsfHNlp7AL9YFV5qLicaaGmBKIJG8kpzP7WgncG37ZvEvexJG79icfzTHbTXxvO3n1NrmHZcEtNUO50Wv5w/0?wx_fmt=gif)

假設要讓 `backend` 可以被 `frontend` 存取，完整的運作流程大概如下：

1. backend Service 會建立一個可以提供 frontend 存取的 DNS 服務，假設名稱為 `backend-service`

2. kube-proxy 會在 backend pods 上層建立一個 load balancer

3. frontend 要存取 backend，先詢問 DNS，接著會取得 load balancer 的 ip，並進行存取

4. kube-proxy 會根據 load balancer 的分流規則，將存取流量分散到不同的 backend pods

> 還有種特別的 Kubernetes Service 稱為 `LoadBalancer`，適合作為 external load balancer(特別是 for Web server traffic)

## Node

可以是實體機器 or VM，作為 Kubernetes worker，以前稱為 Minion，每一台內部都會安裝以下元件：

- `Kublet`：Master Agent

- `Kube-proxy`：用來連接 service 與 pod 的網路

- `Docker`(or `Rocket`)：Conainer Technology (可自選)

## Kubernetes Master

Kubernetes Master 中提供 REST API service，以及產生 Replication Controller 的服務。

-------------------------------------------------------------------

References
==========

- [十分鐘帶你理解Kubernetes核心概念 : 歌穀穀](http://www.gegugu.com/2016/01/06/9845.html)

- [Kubernetes基本要素介绍 - DockOne.io](http://dockone.io/article/1260)

- [Kubernetes基本要素介绍（下） - DockOne.io](http://dockone.io/article/1281)

- [Kubernetes系统架构简介](http://www.infoq.com/cn/articles/Kubernetes-system-architecture-introduction)

- [Kubernetes - Using Multiple Clusters](http://kubernetes.io/docs/admin/multi-cluster/)

## VM in container

- [Sebastien Goasguen: Running VMs in Docker Containers via Kubernetes](http://sebgoa.blogspot.tw/2015/05/running-vms-in-docker-containers-via.html)

- [rancher/vm: Package and Run Virtual Machines as Docker Containers](https://github.com/rancher/vm)