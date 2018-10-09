

動態調整 pod number
=================

> oc scale rc myweb --replicas=2


Deployment 應用
==============

1. 是一種以抽象的方式定義 Pod & ReplicaSet 如何佈署的概念

2. 可透過檢查單一 deployment 的狀態來確認整體 wordload 是否佈署完成

3. 更新 Deployment 即可連同更新相對應的 Pod

4. 可快速 Rollback



Horizontal Pod Autoscaler (HPA)
===============================