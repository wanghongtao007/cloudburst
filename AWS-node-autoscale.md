在 AWS 上使用 IPI（Installer-Provisioned Infrastructure）部署的 OpenShift 集群天然集成了机器 API（Machine API）。要实现基于负载（Workload）的节点自动弹性扩缩，架构主要由 OpenShift 的 **ClusterAutoscaler** 和 **MachineAutoscaler** 两个 CRD (Custom Resource Definition) 配合完成。

无需编写底层的 Pod 监视脚本，OpenShift 自带的 Autoscaler 会监控因资源不足（CPU/Memory）而处于 `Pending` 状态的 Pod，并自动触发 AWS EC2 实例的创建；当节点空闲且资源利用率低于阈值时，自动缩容并回收节点。

### 方案整体架构

1. **ClusterAutoscaler（集群层级）**：全局唯一的资源，用于定义整个集群的扩缩容策略（如最大/最小节点总数、Pod 冷却等待时间、CPU/内存上限等）。
2. **MachineAutoscaler（机器组层级）**：针对指定的 `MachineSet` 设置副本数的上下限（`minReplicas` 和 `maxReplicas`）。
3. **测试应用 (Workload)**：部署带有明确 `resources.requests` 的 Deployment，通过调整副本数制造 Pending Pod 验证自动扩缩。

### POC 实现脚本与操作步骤

你可以直接登录到 OC 命令行终端执行以下操作：

#### 步骤 1：命令行登录集群

Bash

```
oc login https://api.ocp.2jwnl.sandbox1151.opentlc.com:6443 -u admin -p BEY0M6sr6Hb2sEev --insecure-skip-tls-verify
```

#### 步骤 2：部署集群全局 Autoscaler (ClusterAutoscaler)

创建 `cluster-autoscaler.yaml`：

YAML

```
apiVersion: autoscaling.openshift.io/v1
kind: ClusterAutoscaler
metadata:
  name: default
spec:
  podPriorityThreshold: -10
  resourceLimits:
    maxNodesTotal: 20
    cores:
      min: 8
      max: 512
    memory:
      min: 16
      max: 1024
  scaleDown:
    enabled: true
    delayAfterAdd: 10m
    delayAfterDelete: 5m
    delayAfterFailure: 3m
    unneededTime: 5m
```

应用该配置：

Bash

```
oc apply -f cluster-autoscaler.yaml
```

#### 步骤 3：为 AWS MachineSet 创建 MachineAutoscaler

自动扩缩需要绑定现有的 Worker 节点的 `MachineSet`。先查询集群当前的 MachineSet 名称：

Bash

```
oc get machineset -n openshift-machine-api
```

![image-20260810110210883](./autoscale-assets/image-20260810110210883.png)



选择某个 Worker MachineSet ，例如 `ocp-fn849-worker-eu-west-1a`，请根据实际命令行输出替换

可选：先修改该machineset，打标签

```
oc edit machineset ocp-2jwnl-worker-us-east-1a -n openshift-machine-api

####增加：
spec:
  template:
    spec:
      metadata:
        labels:
          node-role.kubernetes.io/autoscale-worker: "" # <--- 添加此标签
```



创建 `machine-autoscaler.yaml`：

YAML

```
apiVersion: autoscaling.openshift.io/v1beta1
kind: MachineAutoscaler
metadata:
  name: worker-eu-west-1a-autoscaler
  namespace: openshift-machine-api
spec:
  minReplicas: 1
  maxReplicas: 5
  scaleTargetRef:
    apiVersion: machine.openshift.io/v1beta1
    kind: MachineSet
    name: ocp-fn849-worker-eu-west-1a # <--- 此处替换为真实的 MachineSet 名称
```

应用配置：

Bash

```
oc apply -f machine-autoscaler.yaml
```

### POC 验证测试流程

#### 1. 创建测试项目并部署压测负载

自动扩缩的关键触发条件是 **Pod Request 明确且节点总资源不足导致 Pod 进入 Pending**。

Bash

```
# 创建测试项目
oc new-project autoscaling-poc

# 创建测试 Deployment (每个 Pod 申请 2 核 CPU, 4Gi 内存)
cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auto-scale-test
  namespace: autoscaling-poc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auto-scale-test
  template:
    metadata:
      labels:
        app: auto-scale-test
    spec:
      nodeSelector:
        node-role.kubernetes.io/autoscale-worker: ''
      containers:
      - name: pause
        image: k8s.gcr.io/pause:3.1
        resources:
          requests:
            cpu: "8000m"
            memory: "16Gi"
EOF
```



```
####负载增加的Deployment
cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auto-scale-test
  namespace: autoscaling-poc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auto-scale-test
  template:
    metadata:
      labels:
        app: auto-scale-test
    spec:
      nodeSelector:
        node-role.kubernetes.io/autoscale-worker: ''
      containers:
      - name: stress
        image: vish/stress
        args:
        - -cpus
        - "8"              # 真实吃满 2 核 CPU
        - -mem-total
        - "16Gi"          # 真实吃满 2G 内存
        resources:
          requests:
            cpu: "8000m"
            memory: "16Gi"
          limits:
            cpu: "8000m"
            memory: "16Gi"
EOF
```



#### 2. 触发扩容 (Scale Out)

手动将应用副本数从 1 大幅增加到 15，这会瞬间超出现有 Worker 节点的 CPU/内存容量 limit：

Bash

```
oc scale deployment auto-scale-test --replicas=10 -n autoscaling-poc
```

或者，通过压力负载的增加，自动实现扩缩

```
oc autoscale deployment auto-scale-test \
  -n autoscaling-poc \
  --min=1 \
  --max=10 \
  --cpu-percent=50
```



#### 3. 观察节点扩容状态

在新窗口运行以下命令观察自动扩容过程：

Bash

```
# 1. 观察集群节点状态 (约 3-5 分钟内 AWS 会新生成 EC2 实例并 join 集群)
oc get nodes -w

# 2. 观察 MachineSet 副本数自动增加
oc get machineset -n openshift-machine-api

# 3. 查看 ClusterAutoscaler 日志
oc logs -n openshift-machine-api -l cluster-autoscaler=default --tail=50 -f
```

#### 4. 触发缩容 (Scale In)

验证完节点自动拉起后，将副本数缩减回 1：

Bash

```
oc scale deployment auto-scale-test --replicas=1 -n autoscaling-poc
```

或者

```
oc delete HorizontalPodAutoscaler  auto-scale-test -n autoscaling-poc
oc scale deployment auto-scale-test --replicas=1 -n autoscaling-poc
```



*观察：* 在经过约 5-10 分钟（即 `unneededTime` 和冷却周期）后，集群将自动停止并注销多余的机器，节点数量恢复初始状态。