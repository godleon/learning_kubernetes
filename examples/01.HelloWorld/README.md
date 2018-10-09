
權限問題
======

## 方法 1

```bash
$ oc edit scc anyuid
```

並在 `user` section 中加入設定：

> system:serviceaccount:<project-name>:default


## 方法 2

或是直接輸入下列指令：

> oc adm policy add-scc-to-user anyuid -z default -n <project-name>