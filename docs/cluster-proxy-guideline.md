# Cluster Proxy Service Access 配置指南

基于 https://github.com/stolostron/cluster-proxy-addon/tree/main/pkg/serviceproxy/readme.md

## 环境信息

- **Hub Cluster**: obs-hub-of-hubs-aws-418-sno-c82l2
- **Managed Cluster**: cluster1
- **Cluster Proxy URL**: cluster-proxy-user.apps.obs-hub-of-hubs-aws-418-sno-c82l2.scale.red-chesterfield.com

---

## 场景 1: 使用 Service Account 访问 Managed Cluster Services

### 目标
在 hub cluster 上创建 service account，通过 ClusterPermission 授权该 service account 访问 cluster1 上 `open-cluster-management-agent-addon` namespace 中的 services。

### 步骤 1.1: 创建 Service Account

```bash
# 创建 namespace
oc create namespace test

# 创建 service account
oc create serviceaccount test-sa -n test

# 为 service account 创建 token secret (Kubernetes 1.24+)
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: test-sa-token
  namespace: test
  annotations:
    kubernetes.io/service-account.name: test-sa
type: kubernetes.io/service-account-token
EOF

# 等待 token 生成
sleep 3

# 验证创建
oc get sa test-sa -n test
oc get secret test-sa-token -n test
```

### 步骤 1.2: 创建 ClusterPermission

创建 ClusterPermission 资源，授权 service account 访问 cluster1 上的 services。

```bash
cat <<EOF | oc apply -f -
apiVersion: rbac.open-cluster-management.io/v1alpha1
kind: ClusterPermission
metadata:
  name: test-services
  namespace: cluster1
spec:
  roles:
  - namespace: open-cluster-management-agent-addon
    rules:
    - apiGroups: [""]
      resources: ["services"]
      verbs: ["get", "list"]
  roleBindings:
  - namespace: open-cluster-management-agent-addon
    roleRef:
      kind: Role
    subject:
      apiGroup: rbac.authorization.k8s.io
      kind: User
      name: cluster:hub:system:serviceaccount:test:test-sa
EOF
```

**重要**: Service Account 的用户名必须使用格式 `cluster:hub:system:serviceaccount:<namespace>:<sa-name>`

### 步骤 1.3: 验证 ManifestWork 创建

```bash
# 检查 ManifestWork 是否创建
oc get manifestwork -n cluster1 | grep test-services

# 检查 ManifestWork 状态
oc get manifestwork -n cluster1 -o json | \
  jq -r '.items[] | select(.metadata.name | startswith("test-services")) |
  "\(.metadata.name): Applied=\(.status.conditions[] | select(.type=="Applied") | .status), Available=\(.status.conditions[] | select(.type=="Available") | .status)"'

# 查看 ManifestWork 详细配置（验证 Role 和 RoleBinding）
oc get manifestwork -n cluster1 -o name | grep test-services | head -1 | \
  xargs -I {} oc get {} -n cluster1 -o yaml
```

**验证结果**:
```
test-services-f01aa: Applied=True, Available=True
```

ManifestWork 内容显示在 cluster1 上创建了：
- Role: `test-services` (namespace: open-cluster-management-agent-addon)
- RoleBinding: `test-services` (绑定到 user: cluster:hub:system:serviceaccount:test:test-sa)

### 步骤 1.4: 测试访问

```bash
# 获取 service account token
SA_TOKEN=$(oc get secret test-sa-token -n test -o jsonpath='{.data.token}' | base64 -d)

# 设置 cluster-proxy URL
CLUSTER_PROXY_URL="cluster-proxy-user.apps.obs-hub-of-hubs-aws-418-sno-c82l2.scale.red-chesterfield.com"

# 测试访问 cluster1 上的 services
curl -k -H "Authorization: Bearer $SA_TOKEN" \
  "https://$CLUSTER_PROXY_URL/cluster1/api/v1/namespaces/open-cluster-management-agent-addon/services"
```

### ❌ 实际结果（问题）

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```

**问题分析**:
- ClusterPermission 已成功创建
- ManifestWork 显示 Applied=True, Available=True
- cluster1 上的 Role 和 RoleBinding 已正确创建
- 但通过 cluster-proxy 访问时返回 401 Unauthorized

### ✅ 期望结果

应该返回 services 列表，例如：

```json
{
  "kind": "ServiceList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "..."
  },
  "items": [
    {
      "metadata": {
        "name": "cluster-proxy-service-proxy",
        "namespace": "open-cluster-management-agent-addon"
      },
      "spec": {
        "ports": [...],
        "selector": {...}
      }
    }
  ]
}
```

---

## 场景 2: 使用 OpenShift User 访问 Managed Cluster Pods

### 目标
创建 OpenShift 用户（使用 HTPasswd），通过 ClusterPermission 授权该用户访问 cluster1 上 `open-cluster-management-agent-addon` namespace 中的 pods。

### 步骤 2.1: 创建 OpenShift User

使用 HTPasswd 身份提供者创建用户。

```bash
# 创建 htpasswd 文件
htpasswd -c -B -b /tmp/htpasswd testuser password123

# 创建 secret
oc create secret generic htpass-secret \
  --from-file=htpasswd=/tmp/htpasswd \
  -n openshift-config --dry-run=client -o yaml | oc apply -f -

# 配置 OAuth 使用 HTPasswd
oc get oauth cluster -o json | \
jq '.spec.identityProviders += [{
  "name": "htpasswd_provider",
  "mappingMethod": "claim",
  "type": "HTPasswd",
  "htpasswd": {
    "fileData": {
      "name": "htpass-secret"
    }
  }
}]' | oc apply -f -

# 等待 OAuth pods 重启
echo "Waiting for OAuth pods to restart..."
sleep 30

# 验证用户可以登录
oc login -u testuser -p password123 --insecure-skip-tls-verify=true

# 切换回 admin
oc login -u kube:admin --insecure-skip-tls-verify=true

# 验证用户创建成功
oc get user testuser
```

**验证结果**:
```
NAME       UID                                    FULL NAME   IDENTITIES
testuser   8cbf0b70-16ab-4ad8-862e-d1ccfe905250               htpasswd_provider:testuser
```

### 步骤 2.2: 创建 ClusterPermission

创建 ClusterPermission 资源，授权 testuser 访问 cluster1 上的 pods。

```bash
cat <<EOF | oc apply -f -
apiVersion: rbac.open-cluster-management.io/v1alpha1
kind: ClusterPermission
metadata:
  name: test-pods
  namespace: cluster1
spec:
  roles:
  - namespace: open-cluster-management-agent-addon
    rules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["get", "list"]
  roleBindings:
  - namespace: open-cluster-management-agent-addon
    roleRef:
      kind: Role
    subject:
      apiGroup: rbac.authorization.k8s.io
      kind: User
      name: testuser
EOF
```

### 步骤 2.3: 验证 ManifestWork 创建

```bash
# 检查 ManifestWork 是否创建
oc get manifestwork -n cluster1 | grep test-pods

# 检查 ManifestWork 状态
oc get manifestwork -n cluster1 -o json | \
  jq -r '.items[] | select(.metadata.name | startswith("test-pods")) |
  "\(.metadata.name): Applied=\(.status.conditions[] | select(.type=="Applied") | .status), Available=\(.status.conditions[] | select(.type=="Available") | .status)"'

# 查看 ManifestWork 详细配置
oc get manifestwork -n cluster1 -o name | grep test-pods | head -1 | \
  xargs -I {} oc get {} -n cluster1 -o yaml
```

**验证结果**:
```
test-pods-373a9: Applied=True, Available=True
```

ManifestWork 内容显示在 cluster1 上创建了：
- Role: `test-pods` (namespace: open-cluster-management-agent-addon)
- RoleBinding: `test-pods` (绑定到 user: testuser)

### 步骤 2.4: 测试访问

```bash
# 登录为 testuser 获取 token
oc login -u testuser -p password123 --insecure-skip-tls-verify=true
USER_TOKEN=$(oc whoami -t)

# 切换回 admin
oc login -u kube:admin --insecure-skip-tls-verify=true

# 设置 cluster-proxy URL
CLUSTER_PROXY_URL="cluster-proxy-user.apps.obs-hub-of-hubs-aws-418-sno-c82l2.scale.red-chesterfield.com"

# 测试访问 cluster1 上的 pods
curl -k -H "Authorization: Bearer $USER_TOKEN" \
  "https://$CLUSTER_PROXY_URL/cluster1/api/v1/namespaces/open-cluster-management-agent-addon/pods"
```

### ❌ 实际结果（问题）

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```

**问题分析**:
- OpenShift User (testuser) 已成功创建
- ClusterPermission 已成功创建
- ManifestWork 显示 Applied=True, Available=True
- cluster1 上的 Role 和 RoleBinding 已正确创建
- 但通过 cluster-proxy 访问时返回 401 Unauthorized

### ✅ 期望结果

应该返回 pods 列表，例如：

```json
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "..."
  },
  "items": [
    {
      "metadata": {
        "name": "klusterlet-addon-workmgr-...",
        "namespace": "open-cluster-management-agent-addon"
      },
      "spec": {
        "containers": [...],
        "nodeName": "..."
      },
      "status": {
        "phase": "Running"
      }
    }
  ]
}
```

---

## 完整自动化测试脚本

一次性执行两个场景的完整脚本：

```bash
#!/bin/bash
set -e

echo "======================================================================"
echo "Cluster Proxy Service Access Test"
echo "======================================================================"

CLUSTER_PROXY_URL="cluster-proxy-user.apps.obs-hub-of-hubs-aws-418-sno-c82l2.scale.red-chesterfield.com"

# ============================================================================
# Scenario 1: Service Account -> Services
# ============================================================================
echo -e "\n[Scenario 1] Testing Service Account Access to Services"
echo "========================================================================"

echo -e "\n[Step 1.1] Creating Service Account..."
oc create namespace test --dry-run=client -o yaml | oc apply -f -
oc create serviceaccount test-sa -n test --dry-run=client -o yaml | oc apply -f -

cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: test-sa-token
  namespace: test
  annotations:
    kubernetes.io/service-account.name: test-sa
type: kubernetes.io/service-account-token
EOF

sleep 3
oc get sa test-sa -n test

echo -e "\n[Step 1.2] Creating ClusterPermission for Service Account..."
cat <<EOF | oc apply -f -
apiVersion: rbac.open-cluster-management.io/v1alpha1
kind: ClusterPermission
metadata:
  name: test-services
  namespace: cluster1
spec:
  roles:
  - namespace: open-cluster-management-agent-addon
    rules:
    - apiGroups: [""]
      resources: ["services"]
      verbs: ["get", "list"]
  roleBindings:
  - namespace: open-cluster-management-agent-addon
    roleRef:
      kind: Role
    subject:
      apiGroup: rbac.authorization.k8s.io
      kind: User
      name: cluster:hub:system:serviceaccount:test:test-sa
EOF

echo -e "\n[Step 1.3] Verifying ManifestWork..."
sleep 10
oc get manifestwork -n cluster1 | grep test-services

oc get manifestwork -n cluster1 -o json | \
  jq -r '.items[] | select(.metadata.name | startswith("test-services")) |
  "\(.metadata.name): Applied=\(.status.conditions[] | select(.type=="Applied") | .status), Available=\(.status.conditions[] | select(.type=="Available") | .status)"'

echo -e "\n[Step 1.4] Testing Service Account Access..."
SA_TOKEN=$(oc get secret test-sa-token -n test -o jsonpath='{.data.token}' | base64 -d)

echo "URL: https://$CLUSTER_PROXY_URL/cluster1/api/v1/namespaces/open-cluster-management-agent-addon/services"
echo -e "\nActual Result:"
curl -k -H "Authorization: Bearer $SA_TOKEN" \
  "https://$CLUSTER_PROXY_URL/cluster1/api/v1/namespaces/open-cluster-management-agent-addon/services" 2>/dev/null | jq .

# ============================================================================
# Scenario 2: OpenShift User -> Pods
# ============================================================================
echo -e "\n\n[Scenario 2] Testing OpenShift User Access to Pods"
echo "========================================================================"

echo -e "\n[Step 2.1] Creating OpenShift User (testuser)..."
htpasswd -c -B -b /tmp/htpasswd testuser password123

oc create secret generic htpass-secret \
  --from-file=htpasswd=/tmp/htpasswd \
  -n openshift-config --dry-run=client -o yaml | oc apply -f -

oc get oauth cluster -o json | \
jq '.spec.identityProviders += [{
  "name": "htpasswd_provider",
  "mappingMethod": "claim",
  "type": "HTPasswd",
  "htpasswd": {
    "fileData": {
      "name": "htpass-secret"
    }
  }
}]' | oc apply -f - 2>&1 | grep -v "Warning"

echo "Waiting for OAuth to reconcile (30 seconds)..."
sleep 30

oc login -u testuser -p password123 --insecure-skip-tls-verify=true > /dev/null 2>&1
oc login -u kube:admin --insecure-skip-tls-verify=true > /dev/null 2>&1
oc get user testuser

echo -e "\n[Step 2.2] Creating ClusterPermission for User..."
cat <<EOF | oc apply -f -
apiVersion: rbac.open-cluster-management.io/v1alpha1
kind: ClusterPermission
metadata:
  name: test-pods
  namespace: cluster1
spec:
  roles:
  - namespace: open-cluster-management-agent-addon
    rules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["get", "list"]
  roleBindings:
  - namespace: open-cluster-management-agent-addon
    roleRef:
      kind: Role
    subject:
      apiGroup: rbac.authorization.k8s.io
      kind: User
      name: testuser
EOF

echo -e "\n[Step 2.3] Verifying ManifestWork..."
sleep 10
oc get manifestwork -n cluster1 | grep test-pods

oc get manifestwork -n cluster1 -o json | \
  jq -r '.items[] | select(.metadata.name | startswith("test-pods")) |
  "\(.metadata.name): Applied=\(.status.conditions[] | select(.type=="Applied") | .status), Available=\(.status.conditions[] | select(.type=="Available") | .status)"'

echo -e "\n[Step 2.4] Testing User Access..."
oc login -u testuser -p password123 --insecure-skip-tls-verify=true > /dev/null 2>&1
USER_TOKEN=$(oc whoami -t)
oc login -u kube:admin --insecure-skip-tls-verify=true > /dev/null 2>&1

echo "URL: https://$CLUSTER_PROXY_URL/cluster1/api/v1/namespaces/open-cluster-management-agent-addon/pods"
echo -e "\nActual Result:"
curl -k -H "Authorization: Bearer $USER_TOKEN" \
  "https://$CLUSTER_PROXY_URL/cluster1/api/v1/namespaces/open-cluster-management-agent-addon/pods" 2>/dev/null | jq .

echo -e "\n======================================================================"
echo "Test Complete!"
echo "======================================================================"
```

---

## 故障排查

### 两个场景都返回 401 Unauthorized

**可能的原因**:

1. **cluster-proxy 配置问题**
   ```bash
   # 检查 ManagedProxyServiceResolver 配置
   oc get managedproxyserviceresolvers.proxy.open-cluster-management.io service-proxy -o yaml

   # 检查 cluster1 是否在正确的 ManagedClusterSet
   oc get managedcluster cluster1 --show-labels
   ```

2. **cluster-proxy-addon 未正确配置或运行**
   ```bash
   # 检查 addon 状态
   oc get managedclusteraddon cluster-proxy -n cluster1 -o yaml

   # 检查 addon pods
   oc get pods -n multicluster-engine | grep cluster-proxy

   # 查看 user-server 日志
   oc logs -n multicluster-engine <pod-name> -c user-server --tail=100
   ```

3. **身份验证流程问题**
   ```bash
   # 使用 verbose 模式查看详细请求信息
   curl -k -v -H "Authorization: Bearer $SA_TOKEN" \
     "https://$CLUSTER_PROXY_URL/cluster1/api/v1/namespaces/open-cluster-management-agent-addon/services" 2>&1 | head -50
   ```

4. **RBAC 在 cluster1 上未生效**（如果可以直接访问 cluster1）
   ```bash
   # 切换到 cluster1
   oc config use-context <cluster1-context>

   # 验证 Role 和 RoleBinding
   oc get role test-services -n open-cluster-management-agent-addon -o yaml
   oc get rolebinding test-services -n open-cluster-management-agent-addon -o yaml
   oc get role test-pods -n open-cluster-management-agent-addon -o yaml
   oc get rolebinding test-pods -n open-cluster-management-agent-addon -o yaml
   ```

---

## 清理资源

```bash
# 删除 ClusterPermissions
oc delete clusterpermission test-services -n cluster1
oc delete clusterpermission test-pods -n cluster1

# 删除 service account 和 namespace
oc delete namespace test

# 删除 user
oc delete user testuser
oc delete identity htpasswd_provider:testuser

# 删除 htpasswd secret
oc delete secret htpass-secret -n openshift-config

# 移除 htpasswd identity provider
oc edit oauth cluster
# 在编辑器中删除 spec.identityProviders 中的 htpasswd_provider 条目
```

---

## 参考资料

- [Cluster Proxy Service Proxy Guide](https://github.com/stolostron/cluster-proxy-addon/tree/main/pkg/serviceproxy/readme.md)
- [Open Cluster Management ClusterPermission API](https://open-cluster-management.io/concepts/clusterpermission/)
