# Kubez-autoscaler Overview

`kubez-autoscaler` 通过为 `deployment` / `statefulset` 添加 `annotations` 的方式，自动管理维护对应 `HorizontalPodAutoscaler` 的生命周期.

### Installing

`kubez-autoscaler` 控制器的安装非常简单，通过 `kubectl`, 执行 `apply` 如下文件即可完成安装.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kubez
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kubez
rules:
  - apiGroups:
      - "*"
    resources:
      - horizontalpodautoscalers
      - deployments
      - statefulsets
    verbs:
      - get
      - watch
      - create
      - delete
      - update
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubez
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubez
subjects:
- kind: ServiceAccount
  name: kubez
  namespace: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    kubez.hpa.controller: kubez-autoscaler
  name: kubez-autoscaler-controller
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      kubez.hpa.controller: kubez-autoscaler
  template:
    metadata:
      labels:
        kubez.hpa.controller: kubez-autoscaler
    spec:
      containers:
        - image: jacky06/kubez-autoscaler-controller:v0.0.1
          command:
            - kubez-autoscaler-controller
            - --leader-elect=true
          imagePullPolicy: IfNotPresent
          name: kubez-autoscaler-controller
      serviceAccountName: kubez
```

然后通过 `kubectl get pod  -l kubez.hpa.controller=kubez-autoscaler` 能看到 `kubez-autoscaler` 已经启动成功.
```bash
NAME                                          READY   STATUS    RESTARTS   AGE
kubez-autoscaler-controller-dbc4bc4d8-hwpqz   1/1     Running   0          20s
kubez-autoscaler-controller-dbc4bc4d8-tqxrl   1/1     Running   0          20s
```

### Getting Started

在 `deployment` / `statefulset` 的 `annotations` 中添加所需注释即可自动创建对应的 `HPA`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    ...
    hpa.caoyingjunz.io/minReplicas: "2"  # MINPODS
    hpa.caoyingjunz.io/maxReplicas: "6"  # MAXPODS
    cpu.hpa.caoyingjunz.io/AverageUtilization: "60"  # TARGETS
    ...
  name: test1
  namespace: default
  ...
```

`kubez-autoscaler-controller` 会根据注释的变化，自动同步 `HPA` 的生命周期.

通过 `kubectl get hpa test1` 命令，可以看到 `deployment` / `statefulset` 关联的 `HPA` 被自动创建
```bash
NAME    REFERENCE          TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
test1   Deployment/test1   6% / 60%        1         2         1          5h29m
```
