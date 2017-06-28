此篇文章介紹 Kubernetes Volume, PersistentVolume, PersistentVolumeClaim 的基本概念 & 使用方式

Overview
========

由於在 container 上的資料是會隨著 container 的停止 or 刪除而不見，因此如何讓 container 運作時產生的資料可以持續保留著，一直都是在使用 container 時的一個非常重要的課題。

以往在只有使用 docker 的環境下，僅能使用 host path mapping 的方式將 host 的空間掛載到 container 上(或是透過 volume container 的方式分享)，而在最近的 docker v1.7 版本中也僅支援了弱弱的 volume driver，因此在資料保存上總是沒這麼方便。

而在 K8s 中則將此問題抽象化成 **Volume** resource，並支援很多種不同的 volume driver 藉此來提供統一的定義方式 & 強大的擴充彈性 & 多種不同的應用方式。

目前支援的 volume type 可參考[官網資料](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes)，以下會用以下幾種來示範：

- **emptyDir**

- **nfs**

- **iscsi**

- **rbd**


--------------------------------------------


Volume
======

以下用最簡單的例子來示範，使用 **emptyDir** 作為 Volume type 來進行 container 使用特定 volume 的宣告。

首先先介紹 **emptyDir** volume 的特性，**emptyDir** 會在 pod 建立時產生，但不會因為 container crash 而消失，但若是將 pod 移除，資料也會隨著消失。

> 可以設定 `emptyDir.medium = "Memory"` 讓資料存在於記憶體中(可用於需要快速處理資料的應用上)

此種 volume 僅能用來作為資料的暫存區，若是要長久保存資料，不建議用此 volume type，以下是建立 & 掛載 emptyDir volume 的定義方式：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: gcr.io/google_containers/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```


--------------------------------------------


PersistentVolume, PersistentVolumeClaim & StorageClass 概觀
===========================================================

管理 storage 一直是一個獨立的課題，在 Kubernetes 中使用了 PersistentVolume & PersistentVolume 兩種 resource，藉由提供標準的 API，讓 storage 的管理與應用抽象化。

`PersistentVolume`(簡稱 `PV`) resource 在概念上可以視為可讓 pod 掛載的 storage，後面的實作(NFS, iCSCI, RBD ... etc)都已經被標準的 API 隱藏，管理者可以透過標準的宣告方式佈署所需要的 storage resource；而 PersistentVolume 的 lifecycle 會跟著使用 PersistentVolume 的 pod 走，當所有 pod 都消失，PersistentVolume resource 也會跟著消失，但儲存在上面的資料依然會存在。

`PersistentVolumeClaim`(簡稱 `PVC`) resource 則是來自於使用者的 request，概念上類似 pod，只是 pod 使用 worker node 上的運算資源，可限定 CPU & Memory；而 PersistentVolumeClaim 使用的則是 PV，可限定 storage 的 size & access mode(read/write or read-only)。

為了讓 PersistentVolume 實際上可以支援各種不同的 storage，並提供各種不同的功能(例如：QoS, Backup policy ... 等等)，Kubernetes 透過 `StorageClass` resource 來將部份抽象化，讓使用者透過宣告不同 StorageClass 的方式就可以使用不同的 external storage。


--------------------------------------------


Volume & Claim 的生命周期
=======================

PV 在 K8s cluster 中屬於 resource，而 PVC 則是目標為 PV 的 request；而兩者的生命周期會歷經下面幾個階段：

## 1. Provisioning

PV 有兩種方式可以被 provision，`static` & `dynamic`:

### Static

在此種模式下，K8s cluster 管理者會預先建立好多個 PV 並與真實的 storage 相連接，以提供給使用者作使用。

### Dynamic

當使用者需要 volume resource 時，必須透過宣告 PVC 的方式來要求 resource，但若是當下沒有 static PV 可用時，此時 K8s cluster 就會嘗試透過動態的方式建立 PV。

而建立 PV 的細節取決於 StorageClass 的定義，當然要動態建立 PV 的話，必須在 K8s cluster 管理者已經預先準備好建立指定 StorageClass 連結的所有相關設定(例如要建立 NFS PV，當然預先要設定好 NFS server & 設定好 NFS 位址與掛載路徑...等等)。

## 2. Binding

K8s cluster 在運作過程中，會有個 control loop 持續的監控目前 cluster 上所擁有的 PV resource 以及隨時從 user 傳送過來的 PVC，當 PVC 被提出時，loop 就會尋找到合適的 static PV resource 後並與其進行 binding；若是當下沒有合適的 PV，就會藉由動態的方式產生並與其 binding。

但若是無法動態產生 PV resource 且 PVC 也一直無法找到合適的 PV 可連結，就會一直處於 **unbound** 的狀態，直到有合適的 PV resource 為止

> 實際上的狀況是，動態產生的 PV resource 並不一定會完全與 PVC 所要求的規格一模一樣，這取決於當初管理者設定 storage 是否有提供這樣的彈性

## 3. Using

此時 pod 會將 PVC 當作是 volume 使用，所有資料儲存的細節都已經被標準的 API 所隱藏；此外有些 volume type 支援同時給多個 pod 存取，這個部份就取決於 storage 是否支援這樣的方式，例如：NFS 可同時給多個 pod 存取，而 iSCSI or RBD 則無法。

當 PVC 與 PV 已經關聯，這個 PV resource 就會被與 PVC 相關連的 pod 所使用，一直到 pod 消失為止。


## 4. Releasing

當使用者已經不需要 PV resource 時可以將 PVC 刪除(**注意! 可以刪除的是 Claim 不是 Volume 本身**)，一旦當 PV 沒有與任何 PVC binding 在一起時，就會被判定為處於 releasing 狀態，但此時還無法接受與其他 PVC binding，因為前一個 PVC 可能還會因為 reclaim policy 的關係還有後續的工作需要完成。

## 5. Reclaiming

reclaim policy 是用來告知 PV 在 releasing 狀態時要執行的工作，目前有 3 個選項可以進行，分別是

- `Retained`: 允許 PV 再度被 reclaim  

- `Deleted`: 從 K8s cluster 中移除 PV resource，同時也刪除相對應在 storage 上的資料

- `Recycled`: K8s 會協助作基本清理，但也可由管理者自訂清除工作的 script，完成清除工作後再重新於 K8s cluster 釋出並接受新的 PVC


--------------------------------------------


支援的 PersistentVolume Type
===========================

此部份可參考[官網最新的支援列表](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes)。


--------------------------------------------


StorageClass
============

從上面的支援列表可以看出，Kubernetes 支援了相當多的 storage，但每個 storage 都有不同的連接方式 & 通訊細節，整合上的複雜程度可想而知。

因此 Kubernetes 提出了 `StorageClass` 的概念來將這些實作細節抽象化了，藉由設定 `provisioner` & `parameters` 兩個主要參數來與各種不同的 storage 連結，以下用一個簡單範例來說明：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/rbd
parameters:
  # Ceph monitors, comma delimited. This parameter is required.
  monitors: 10.16.153.105:6789
  # Ceph client ID that is capable of creating images in the pool. Default is “admin”
  adminId: kube
  # The namespace for adminSecret. Default is “default”
  adminSecretName: ceph-secret
  # Secret Name for adminId. This parameter is required. The provided secret must have type “kubernetes.io/rbd”
  adminSecretNamespace: kube-system
  # Ceph RBD pool. Default is “rbd”
  pool: kube
  # Ceph client ID that is used to map the RBD image. Default is the same as adminId
  userId: kube
  # The name of Ceph Secret for userId to map RBD image. It must exist in the same namespace as PVCs. This parameter is required. The provided secret must have type “kubernetes.io/rbd”
  userSecretName: ceph-secret-user
```

從上面的 Ceph RBD 看的出還需要額外定義一個 **secret** resource，定義的範例如下：

> kubectl create secret generic ceph-secret --type="kubernetes.io/rbd" --from-literal=key='QVFEQ1pMdFhPUnQrSmhBQUFYaERWNHJsZ3BsMmNjcDR6RFZST0E9PQ==' --namespace=kube-system

詳細 secret 的概念 & 使用方法可參考[官網文章](https://kubernetes.io/docs/concepts/configuration/secret/)。

## Provisioner

透過 `provisioner` 關鍵字設定。

透過此設定指定不同的 volume plugin 來建立 PersistentVolume，針對不同的 volume plugin 有不同的宣告方式，詳細的設定方法可參考[官方文件的範例](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#provisioner)。

## Parameters

透過 `parameters` 關鍵字設定。

很顯而易見的，parameters 的部份會緊密的與上方所宣告的 volume plugin 關聯，因此在設定 parameters 要注意每個不同的 volume plugin 所支援的選項有那些。


--------------------------------------------


PersistentVolume
================

接著要來了解如何在 K8s cluster 中宣告 PV resource 並使用，以下是一個簡單範例：

```yaml
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv0003
    # Mount Options
    #annotations:
    #  volume.beta.kubernetes.io/mount-options: "discard"
  spec:
    # Capacity
    capacity:
      storage: 5Gi
    # Access Modes
    accessModes:
      - ReadWriteOnce
    # Reclaim Policy
    persistentVolumeReclaimPolicy: Recycle
    # Class
    storageClassName: slow
    nfs:
      path: /tmp
      server: 172.17.0.2
```

從上面的 PV 定義，原本常見的 apiVersion, kind, metadata .... 等等就不額外解釋，以下就針對 PV 專有的 spec 來說明：

## Capacity

透過 `capacity` 關鍵字使用。

基本上每個 PV 都會宣告有多少儲存容量可用，上面的範例使用 `Gi` 關鍵字來表示 GiB；而其他不同單位 & 不同關鍵字的用法，可參考 [GitHub 上的說明](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/resources.md)。

> 目前關於 storage 的部份僅有提供 size 的管控，未來預計會提供像是 IOPS, throughput 的管控

## Access Modes

透過 `accessModes` 關鍵字使用。

目前 K8s 提供的 access mode 有以下三種：

- `ReadWriteOnce`: 僅可提供一個 pod 掛載與讀寫

- `ReadOnlyMany`: 可提供多個 pod 掛載與讀取，但只能有一個 pod 可以寫入

- `ReadWriteMany`： 可提供多個 pod 同時掛載與讀寫

基本上，access mode 的支援會與所使用的 storage 有關係，例如 NFS 支援 ReadWriteMany，但 Ceph RBD 就只能支援 ReadOnlyMany，詳細的支援列表資訊可參考[官網文件](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)。

## Class

透過 `storageClassName` 關鍵字使用。

每個 PV 可以擁有一個 class，透過 `storageClassName` 關鍵字指定；而每個 PVC 在宣告時也會指定 class，唯有當 PV 與 PVC 的 **storageClassName** 相同時，兩者才會進行 binding。
> 若是沒有設定 storageClassName 的 PV，就只能跟沒有設定 storageClassName 的 PVC 進行 binding



## Reclaim Policy

透過 `persistentVolumeReclaimPolicy` 關鍵字使用。

目前支援三種 reclaim policy: 

- `Retain`: 手動進行 reclaim

- `Recycle`: 執行基本清理工作 (`rm -rf /thevolume/*`)

- `Delete`: 實際對應的 storage 上的資料也會同時被移除

> 目前僅有 **NFS** & **HostPath** 支援 Recycle，其他像是 AWS EBS, GCE PD, Azure Disk & Cinder volumes 則是支援 Delete

## Phase

由於每個 PV 都有其生命周期，因此 PV 都會處於下面 4 個狀態的其中一個：

- **Available**: 可用，尚未與任何 PVC binding

- **Bound**: 已經與 PVC binding

- **Released**:  與其 binding 的 PVC 已經刪除，但尚未進行 reclaim

- **Failed**: automatic reclaimation 失敗會處於此狀態

## Mount Options

在 `metadata` section 中透過 `volume.beta.kubernetes.io/mount-options` key 加入。

可設定在實際掛載 PV resource 時，加入額外自訂的 mount option，但並非所有 volume type 都支援 mount option，詳情可參考[官方文件支援清單](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#mount-options)。


--------------------------------------------


PersistentVolumeClaim
=====================

接著來認識 PVC 在 K8s cluster 中要如何定義，以下是個簡單範例：

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim
spec:
  # Access Modes
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  # Class
  storageClassName: slow
  # Selector
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

從上面的定義可以看出，PVC 包含了以下幾個部份：

## Access Modes

透過 `accessModes` 關鍵字使用。

這部份跟 PV 中的定義的 access mode 其實是相同的。

## Resources

透過 `resources` 關鍵字使用。

在此處會指定需要多少的 storage resource，這會成為 K8s cluster control loop 在尋找 PV 配對時的依據之一。

## Selector

透過 `selector` 關鍵字使用。

藉由 K8s 原生提供的 [Label selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) 針對 resource 進一步的進行過濾並使用；上面的範例使用了 **matchLabels** & **matchExpressions** 兩種過濾條件來進行雙重過濾。

## Class

透過 `storageClassName` 關鍵字使用。

設定了 storageClassName 之後，PVC 就僅能與擁有相同 class 的 PV 進行 binding。

此外還需要注意若是有設定 [DefaultStorageClass](https://kubernetes.io/docs/admin/admission-controllers/#defaultstorageclass)，若此設定為 true，則未設定 storageClassName 的 PVC 就只能與 storageClassName 為預設值的 PV 作 binding。


--------------------------------------------


Claims As Volumes
=================

實際上 pod 在使用 PV resource 是必須透過 PVC 的，因此除了 PV & PVC 的定義之外，還要額外宣告 Volume & PVC 相關連的定義才可正常運作，其中的關係大概如下圖：

> `PersistentVolume` <--> `PersistentVolumeClaim` <--> `Volume` <--> `Pod`

以下是個簡單範例：

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: dockerfile/nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

透過上面範例可看出，在 Volume 的部份透過 `persistentVolumeClaim` 關鍵字與指定的 PVC 作連結，而若是 K8s cluster 有找到與 PVC 對應的 PV，就可以進而讓這個 pod 使用 PV resource。

> 必須注意的是，每個 PVC 都會隸屬於特定的 namespace，並非屬於 cluster 層級


--------------------------------------------


References
==========

- [Concepts \| Kubernetes](https://kubernetes.io/docs/concepts/)

- [Persistent Volumes \| Kubernetes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

- [Labels and Selectors \| Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)

- [Secrets \| Kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/)