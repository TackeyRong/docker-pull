# 删除 Kubernetes 命名空间的排查与解决方案

## 1. 检查命名空间的状态

使用以下命令查看命名空间状态：

```bash
kubectl get namespace <namespace-name>
```

如果命名空间处于 **`Terminating`** 状态，说明删除过程被阻塞，需进一步排查。

---

## 2. 查看命名空间中的资源

列出命名空间中的所有资源：

```bash
kubectl get all --namespace=<namespace-name>
```

如发现有资源未删除，尝试手动删除：

```bash
kubectl delete all --namespace=<namespace-name>
```

此外，还需检查 `CustomResourceDefinition` (CRD) 和 `Webhook` 等非标准资源：

```bash
kubectl get crd | grep <namespace-name>
kubectl delete crd <crd-name>

kubectl get mutatingwebhookconfiguration | grep <namespace-name>
kubectl delete mutatingwebhookconfiguration <webhook-name>
```

---

## 3. 检查并清理 Finalizers

### 3.1 查看 Finalizers

Kubernetes 的 `Finalizer` 会阻止资源直接删除。可以检查命名空间是否有未清理的 `Finalizer`：

```bash
kubectl get namespace <namespace-name> -o json | jq '.metadata.finalizers'
```

### 3.2 清理 Finalizers

清除 `Finalizer` 以允许命名空间删除：

```bash
kubectl patch namespace <namespace-name> -p '{"metadata":{"finalizers":[]}}' --type=merge
```

---

## 4. 强制删除命名空间

如果资源和 Finalizer 均已清理，仍无法删除，可以尝试强制删除命名空间：

```bash
kubectl delete namespace <namespace-name> --force --grace-period=0
```

---

## 5. 查看 API Server 日志

查看 Kubernetes API Server 的日志，排查是否有关于删除命名空间的详细错误信息：

```bash
journalctl -u kube-apiserver
```

---

## 6. 检查 RBAC 权限

确保当前用户或服务账户具有删除命名空间的权限：

```bash
kubectl auth can-i delete namespaces --as=<user>
```

如权限不足，可使用具有管理员权限的用户执行删除操作。

---

## 7. 检查是否有依赖的资源

某些资源可能与命名空间有绑定关系（如 `ClusterRoleBinding` 等），需要先删除这些关联资源。例如：

### 查看 `ClusterRoleBinding`

```bash
kubectl get clusterrolebinding | grep <namespace-name>
```

### 删除绑定资源

```bash
kubectl delete clusterrolebinding <binding-name>
```

---

## 8. 重启控制平面组件

如果删除命名空间过程长时间挂起，可以尝试重启 Kubernetes 控制平面的核心组件：

```bash
systemctl restart kube-apiserver
systemctl restart kube-controller-manager
```

---

## 9. 总结的清理脚本

以下是一个批量清理命名空间的脚本：

```bash
#!/bin/bash

NAMESPACE=<namespace-name>

# 1. 删除命名空间中的所有资源
kubectl delete all --all --namespace=$NAMESPACE

# 2. 删除 CRDs
CRDS=$(kubectl get crd | grep $NAMESPACE | awk '{print $1}')
for crd in $CRDS; do
    kubectl delete crd $crd
done

# 3. 删除 Webhook
WEBHOOKS=$(kubectl get mutatingwebhookconfiguration | grep $NAMESPACE | awk '{print $1}')
for webhook in $WEBHOOKS; do
    kubectl delete mutatingwebhookconfiguration $webhook
done

# 4. 清理 Finalizer
kubectl patch namespace $NAMESPACE -p '{"metadata":{"finalizers":[]}}' --type=merge

# 5. 强制删除命名空间
kubectl delete namespace $NAMESPACE --force --grace-period=0
```

保存脚本后，赋予执行权限并运行：

```bash
chmod +x delete-namespace.sh
./delete-namespace.sh
```

---

## 10. 其他可能的解决办法

### 检查依赖 APIService

查看是否有未正常注册的 `APIService` 依赖：

```bash
kubectl get apiservices.apiregistration.k8s.io | grep <namespace-name>
```

如发现残留的 `APIService`，使用以下命令删除：

```bash
kubectl delete apiservices.apiregistration.k8s.io <api-service-name>
```

---

## 11. 注意事项

- **强制删除命名空间时可能会导致资源泄漏**，如需要删除的资源未能正确回收。
- 删除命名空间前需确认无重要数据或依赖关系。
- 尽量在非生产环境中测试以上操作。

---

