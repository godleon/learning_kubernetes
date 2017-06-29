設定 kubeconfig
==============

以下為一個 kubeconfig 設定檔的範例：

```yaml
# 預設所使用的 context
current-context: federal-context

apiVersion: v1

kind: Config

preferences:
  colors: true

# cluster & certificate 資訊 (可能會有很多個)
clusters:
- cluster:
    api-version: v1
    certificate-authority: /coreos-bootcfg/assets/tls/ca.pem
    server: https://10.102.2.21
  name: coreos-cluster

# 使用者資訊(可能會有很多個)
users:
- name: default-admin
  user:
    client-certificate: /coreos-bootcfg/assets/tls/admin.pem
    client-key: /coreos-bootcfg/assets/tls/admin-key.pem

# 為上述 cluster, user, namespace 的組合，使用者必須根據自己的使用環境設定正確的 context
# 選擇搭配正確的 cluster, user & namespace 資訊
contexts:
- context:
    cluster: coreos-cluster
#    namespace: chisel-ns
    user: default-admin
  name: federal-context
```

設定完畢後，透過 `kubectl config view` 可以檢視目前 kubeconfig 的設定是否正確。

--------------------------------------------------------

kubeconfig 載入 & 合併規則
=======================

kubectl 在決定要向哪個 Kubernetes cluster 存取前，必須確定使用者的設定，大致流程如下：

## 取得設定檔內容

若是環境複雜的使用者，kubeconfig 的選擇需要注意一下，所使用的規則順序如下：(越前面越優先使用)

1. 透過 `kubectl config` 所指定的參數 (不合併設定)

2. 環境變數 `$KUBECONFIG` 所指定的 kubeconfig 檔案位置 (可合併設定)
> 透過 `$KUBECONFIG` 可以一次指定多個設定檔，但若是出現相同的設定，後面檔案的設定會被忽略不用

3. 使用 `~/.kube/config` 設定檔內容 (不合併設定)

## 決定 context

接著 kubectl 會在全部設定中尋找 context 設定，基本上來源有兩種：

1. 使用 cli 參數 `context` 帶入

2. 從 kubeconfig 設定檔中的 `current-context` 取得

> kubectl 在此階段可接受未設定 context

## 決定 cluster & user

cluster & user 的資料來源同樣也是有兩種：

1. 從 cli 的 `cluster` & `user` 參數帶入

2. 若上個步驟取得 context 資訊，則會使用 context 所指定的 cluster & user

> kubectl 在此階段可接受未取得 cluster & user 資訊

## 確定 cluster 資訊

cluster 的存取資訊可以從以下管道取得：

1. cli 中的參數 `server`、`api-version`、`certificate-authority`、`insecure-skip-tls-verify`

2. kubeconfig 設定檔

> 此時若是連 server 資訊都無法提供，kubectl 就會回報錯誤

## 確定 user 資訊

user 資訊取得同樣也是跟 cluster 資訊類似

1. 從 cli 中取得，若 cli 未指定則會尋找 kubeconfig 設定檔

2. 尋找 `client-certificate`、`client-key`、`username`、`password`、`token` ... 等參數

> 使用者認證的方式只能有一種，若是遇到設定衝突，kubectl 就會回報錯誤

--------------------------------------------------------

驗證存取 Kubernetes Cluster
========================

當 kubeconfig 設定完成後，可用以下指令驗證 Kubernetes cluster 存取是否正常：

```bash
[root@centos7 ~]# kubectl cluster-info

[root@centos7 ~]# kubectl config view
```

--------------------------------------------------------

References
==========

- [Kubernetes - kubectl config](http://kubernetes.io/docs/user-guide/kubectl/kubectl_config/)

- [kubectl config set-cluster](http://kubernetes.io/docs/user-guide/kubectl/kubectl_config_set-cluster/)

- [kubectl config set-credentials](http://kubernetes.io/docs/user-guide/kubectl/kubectl_config_set-credentials/)

- [kubectl config set-context](http://kubernetes.io/docs/user-guide/kubectl/kubectl_config_set-context/)

- [kubectl config use-context](http://kubernetes.io/docs/user-guide/kubectl/kubectl_config_use-context/)