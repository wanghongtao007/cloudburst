





```
#  export CLUSTER_DOMAIN=apps.ocp.2jwnl.sandbox1151.opentlc.com
#  oc login --server=https://api.${CLUSTER_DOMAIN##apps.}:6443 -u admin -p BEY0M6sr6Hb2sEev
# oc project user1-toolings

# echo https://$(oc get route argocd-server --template='{{ .spec.host }}' -n user1-toolings)

https://argocd-server-user1-toolings.apps.ocp.2jwnl.sandbox1151.opentlc.com


# oc get secret argocd-cluster -n user1-toolings -o jsonpath='{.data.admin\.password}' | base64 -d; echo ""
AJqbc7u61P0MGasYwnVoxBhvtNle9HLj


# argocd login argocd-server-user1-toolings.apps.ocp.2jwnl.sandbox1151.opentlc.com   --username admin   --password AJqbc7u61P0MGasYwnVoxBhvtNle9HLj   --insecure   --grpc-web   --skip-test-tls
'admin:login' logged in successfully
Context 'argocd-server-user1-toolings.apps.ocp.2jwnl.sandbox1151.opentlc.com' updated



# 1. 在本地 Kubeconfig 中先配置好两个集群的 Context
oc login --server=https://api.iynoipaf.eastus.aroapp.io:6443 -u admin -p MjcwMDM5
oc config get-contexts


# 3. 将目标集群添加至 Argo CD（这会在目标集群自动创建 service account 并回传 Token）
argocd cluster add <target-cluster-context-name> --name cluster-ai501
argocd cluster add user1-toolings/api-ocp-2jwnl-sandbox1151-opentlc-com:6443/admin --name cluster-ai501
argocd cluster rm cluster-ai501

argocd cluster add default/api-iynoipaf-eastus-aroapp-io:6443/admin --name cluster-azure

```



```
# 1. 为本地默认集群打标签
oc label secret argocd-default-cluster-config -n user1-toolings vendor=OpenShift

# 2. 为远端 ARO 集群打标签
oc label secret cluster-api.iynoipaf.eastus.aroapp.io-1895598084 -n user1-toolings vendor=OpenShift
```



创建ApplicationSet



```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-cluster-app
  namespace: user1-toolings
spec:
  generators:
    - clusters:
        # 使用标签过滤要部署的目标 OpenShift 集群
        selector:
          matchLabels:
            vendor: OpenShift  # 或者直接匹配集群 name
  template:
    metadata:
      name: 'my-app-{{name}}' # 自动根据集群名称生成如 my-app-cluster-a
    spec:
      project: default
      source:
        repoURL: 'https://rhoai-genaiops.github.io/genaiops-helmcharts/'
        targetRevision: HEAD
        path: manifests/openshift/ # 应用清单所在路径（支持 Kustomize 或 Helm）
      destination:
        server: '{{server}}'      # 动态注入集群 API Server URL
        namespace: my-app-ns       # 目标集群的命名空间
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```



```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-cluster-app
  namespace: user1-toolings
spec:
  generators:
    - clusters:
        # 使用标签过滤要部署的目标 OpenShift 集群
        selector:
          matchLabels:
            vendor: OpenShift  # 或者直接匹配集群 name
  template:
    metadata:
      name: 'my-app-{{name}}' # 自动根据集群名称生成如 my-app-cluster-a
    spec:
      project: default
      source:
        repoURL: 'https://github.com/wanghongtao007/openshift-gitops-getting-started'
        targetRevision: HEAD
        path: app # 应用清单所在路径（支持 Kustomize 或 Helm）
      destination:
        server: '{{server}}'      # 动态注入集群 API Server URL
        namespace: my-app-ns       # 目标集群的命名空间
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```



```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-cluster-app
  namespace: user1-toolings
spec:
  generators:
    - clusters:
        # 匹配标签带 vendor: OpenShift 的目标集群
        selector:
          matchLabels:
            vendor: OpenShift
  template:
    metadata:
      name: 'my-app-{{name}}'
    spec:
      project: default
      source:
        repoURL: 'https://github.com/wanghongtao007/cloudburst'
        targetRevision: HEAD
        path: app # 对应 Git 仓库中包含 auto-scale-test-deployment.yaml 的目录路径
      destination:
        server: '{{server}}'
        namespace: autoscaling-poc # 统一部署到目标命名空间 autoscaling-poc
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```



```
##查找集群标签
oc get secrets -n user1-toolings -l argocd.argoproj.io/secret-type=cluster

# 1. 为本地默认集群打标签
oc label secret argocd-default-cluster-config -n user1-toolings name=in-cluster

# 2. 为远端 ARO 集群打标签
oc label secret cluster-api.iynoipaf.eastus.aroapp.io-1895598084 -n user1-toolings name=cluster-azure

oc label node aro-cluster-277rm-clrtj-worker-eastus1-tpnlr node-role.kubernetes.io/autoscale-worker=''
```

deployment

```
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
        - "8"
        - -mem-total
        - "16Gi"
        resources:
          requests:
            cpu: "8000m"
            memory: "16Gi"
          limits:
            cpu: "8000m"
            memory: "16Gi"
```



```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-cluster-app
  namespace: user1-toolings
spec:
  generators:
    # 集群 1：例如生产集群/主集群，配置 8 副本
    - clusters:
        selector:
          matchLabels:
            name: in-cluster # 或直接匹配 name: in-cluster
        values:
          replicas: "4"
    
    # 集群 2：例如测试集群/从集群，配置 2 副本
    - clusters:
        selector:
          matchLabels:
            name: cluster-azure    # 或直接匹配具体集群名
        values:
          replicas: "1"

  template:
    metadata:
      name: 'my-app-{{name}}'
    spec:
      project: default
      source:
        repoURL: 'https://github.com/wanghongtao007/cloudburst'
        targetRevision: HEAD
        path: app
        # 核心：通过 Kustomize 的 overrides 动态覆写副本数
        kustomize:
          patches:
            - target:
                kind: Deployment
                name: auto-scale-test
              patch: |-
                - op: replace
                  path: /spec/replicas
                  value: {{values.replicas}}
      destination:
        server: '{{server}}'
        namespace: autoscaling-poc
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

