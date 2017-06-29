此篇文章介紹如何使用外部的 NFS 服務作為 Kubernetes PersistentVolume

前置作業
=======

首先要確認有一個可以正常提供服務的 NFS server，以下為實驗環境：

```bash
$ showmount -e 10.102.70.123
Export list for 10.102.70.123:
/var/lib/libvirt/images/hdd/nfs-share *
```

- **NFS Server**: `10.102.70.123`

- **Remote Folder**: `/var/lib/libvirt/images/hdd/nfs-share`


----------------------------------


建立 StorageClass
================

不需要額外對 NFS PV 建立 StorageClass resource


----------------------------------


建立 PersistentVolume
====================

首先建立以下的 PV：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-nfs-pv
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteMany
  nfs:
    # Fix the IP and path to your NFS server
    server: 10.102.70.123
    path: "/var/lib/libvirt/images/hdd/nfs-share"
```

建立流程如下：

```bash
$ kubectl create -f https://raw.githubusercontent.com/godleon/learning_kubernetes/master/examples/volumes/nfs/pv.yaml
persistentvolume "my-nfs-pv" created

$ kubectl get pv
NAME        CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
my-nfs-pv   100Mi      RWX           Retain          Available                                      8s

# 檢視 PV 詳細資訊
$ kubectl describe pv my-nfs-pv
Name:		my-nfs-pv
Labels:		<none>
Annotations:	<none>
StorageClass:	
Status:		Available
Claim:		
Reclaim Policy:	Retain
Access Modes:	RWX
Capacity:	100Mi
Message:	
Source:
    Type:	NFS (an NFS mount that lasts the lifetime of a pod)
    Server:	10.102.70.123
    Path:	/var/lib/libvirt/images/hdd/nfs-share
    ReadOnly:	false
Events:		<none>
```


----------------------------------


建立 PersistentVolumeClaim
==========================

完成 PV 建立後，接著就是 PVC 了，PVC 的定義如下：

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
storage: 50Mi
```

實際建立流程如下：

```bash
# 建立 PersistentVolumeClaim
$ kubectl create -f https://raw.githubusercontent.com/godleon/learning_kubernetes/master/examples/volumes/nfs/nfs-pvc.yaml
persistentvolumeclaim "my-nfs-pvc" created

# 從 status 可看出已經與先前建立的 PV bound
$ kubectl get pvc 
NAME         STATUS    VOLUME      CAPACITY   ACCESSMODES   STORAGECLASS   AGE
my-nfs-pvc   Bound     my-nfs-pv   100Mi      RWX                          9s

# 檢視 PVC 詳細資訊
$ kubectl describe pvc my-nfs-pvc 
Name:		my-nfs-pvc
Namespace:	default
StorageClass:	
Status:		Bound
Volume:		my-nfs-pv
Labels:		<none>
Annotations:	pv.kubernetes.io/bind-completed=yes
		pv.kubernetes.io/bound-by-controller=yes
Capacity:	100Mi
Access Modes:	RWX
Events:		<none>
```


----------------------------------


建立 Web Service 連接 PersistentVolume
====================================

## 建立 Replication Controller

以下接著透過 ReplicationController 產生兩個 nginx web server pod，並與上面的 PVC 連結，使用下面的定義：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nfs-web
spec:
  replicas: 2
  selector:
    role: web-frontend
  template:
    metadata:
      labels:
        role: web-frontend
    spec:
      containers:
      - name: web
        image: nginx
        ports:
        - name: web
          containerPort: 80
        volumeMounts:
        - name: nfs
          mountPath: "/usr/share/nginx/html"
      volumes:
      - name: nfs
        persistentVolumeClaim:
          claimName: my-nfs-pvc
```

實際建立流程：

```bash
# 建立 replication controller
$ kubectl create -f https://raw.githubusercontent.com/godleon/learning_kubernetes/master/examples/volumes/nfs/web-front-rc.yaml
replicationcontroller "nfs-web" created

# 取得 replication controller 狀態
$ kubectl get rc
NAME      DESIRED   CURRENT   READY     AGE
nfs-web   2         2         2         10s

# 檢視詳細資訊
$ kubectl describe rc nfs-web 
Name:		nfs-web
Namespace:	default
Selector:	role=web-frontend
Labels:		role=web-frontend
Annotations:	<none>
Replicas:	2 current / 2 desired
Pods Status:	2 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:	role=web-frontend
  Containers:
   web:
    Image:		nginx
    Port:		80/TCP
    Environment:	<none>
    Mounts:
      /usr/share/nginx/html from nfs (rw)
  Volumes:
   nfs:
    Type:	PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:	my-nfs-pvc
    ReadOnly:	false
Events:
  FirstSeen	LastSeen	Count	From			SubObjectPath	Type		Reason			Message
  ---------	--------	-----	----			-------------	--------	------			-------
  17s		17s		1	replication-controller			Normal		SuccessfulCreate	Created pod: nfs-web-3l0wb
  17s		17s		1	replication-controller			Normal		SuccessfulCreate	Created pod: nfs-web-7l3q8
```

## 建立 Service

接著要建立 Service 讓上面建立的兩個 pod 服務可供外部存取，此處使用 NodePort 的方式，以下是 Service 的定義：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: nfs-web
spec:
  type: NodePort
  ports:
    - protocol: TCP
      port: 80
      nodePort: 31080
  selector:
    role: web-frontend
```

實際流程如下：

```bash
# 建立 Service
$ kubectl create -f https://raw.githubusercontent.com/godleon/learning_kubernetes/master/examples/volumes/nfs/nfs-web-service.yaml
service "nfs-web" created

# 檢視 Service
$ kubectl get svc
NAME         CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   10.233.0.1     <none>        443/TCP        8d
nfs-web      10.233.40.95   <nodes>       80:31080/TCP   9s

# 檢視 Service 詳細資訊
$ kubectl describe svc nfs-web 
Name:			nfs-web
Namespace:		default
Labels:			<none>
Annotations:		<none>
Selector:		role=web-frontend
Type:			NodePort
IP:			10.233.40.95
Port:			<unset>	80/TCP
NodePort:		<unset>	31080/TCP
Endpoints:		10.233.103.201:80,10.233.109.80:80
Session Affinity:	None
Events:			<none>
```


----------------------------------


驗證 NFS PersistentVolume
========================

最後透過以下流程驗證：

1. 進入第一個 pod 並檢視 NFS mount status

2. 建立 web index page & 離開 pod

3. 進入第二個 pod 並檢視 NFS mount status

4. 檢查在第 1 個 pod 建立的 web index page 是否存在 & 離開 pod

5. 確認 web page 可連

```bash
# 檢視目前產生出來的 pod
$ kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
.....
nfs-web-3l0wb                       1/1       Running   0          16m
nfs-web-0rv9v                       1/1       Running   0          16m
....

# 進入 pod shell
$ kubectl exec -it nfs-web-3l0wb -- /bin/bash

# 檢視 NFS mount status
root@nfs-web-3l0wb:/# df -h
Filesystem                                           Size  Used Avail Use% Mounted on
....
10.102.70.123:/var/lib/libvirt/images/hdd/nfs-share  2.0T   95G  1.8T   5% /usr/share/nginx/html
...

# 建立 web index page
root@nfs-web-3l0wb:/# echo "<h1>Hello NFS PV and PVC</h1>" > /usr/share/nginx/htm
# 離開 pod shell
root@nfs-web-3l0wb:/# exit
exit

# 登入另外一個 pod shell，檢視資料是否存在
$ kubectl exec -it nfs-web-0rv9v -- /bin/bash
# 檢視 NFS mount
root@nfs-web-0rv9v:/# df -h
Filesystem                                           Size  Used Avail Use% Mounted on
....
10.102.70.123:/var/lib/libvirt/images/hdd/nfs-share  2.0T   95G  1.8T   5% /usr/share/nginx/html
...
# 檢視由另外一個 pod 產生的檔案
root@nfs-web-0rv9v:/# cat /usr/share/nginx/html/index.html 
<h1>Hello NFS PV and PVC</h1>
# 離開 pod shell
root@nfs-web-0rv9v:/# exit
exit

# 透過 node port 檢視網頁是否可正常回應
$ curl http://10.102.60.21:31080
<h1>Hello NFS PV and PVC</h1>
```


----------------------------------


References
==========

- [kubernetes/examples/volumes/nfs at master · kubernetes/kubernetes](https://github.com/kubernetes/kubernetes/tree/master/examples/volumes/nfs)