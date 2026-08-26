





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

