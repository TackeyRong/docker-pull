在 Kubernetes 中，可以使用 `kubectl` 命令将节点设置为不可调度（unschedulable）。这通常用于维护节点或不希望新的 Pod 被调度到该节点上。你可以通过设置节点的 `taint` 或修改节点的调度状态来完成此操作。

### 1. **将节点标记为不可调度（unschedulable）**

如果你想禁止 Kubernetes 调度新的 Pod 到指定节点上，可以使用以下命令将节点设置为不可调度：

```bash
kubectl taint nodes <node-name> node.kubernetes.io/unschedulable=true:NoSchedule
```

- `<node-name>`：你想设置为不可调度的节点名称。
- `node.kubernetes.io/unschedulable=true:NoSchedule` 是一个 taint，它告诉 Kubernetes 不要将新的 Pod 调度到这个节点。

### 2. **将节点重新设置为可调度**

如果你想将节点恢复为可调度状态，可以使用以下命令删除 `taint`：

```bash
kubectl taint nodes <node-name> node.kubernetes.io/unschedulable:NoSchedule-
```

这样 Kubernetes 将允许 Pod 在该节点上调度。

### 3. **直接修改节点的调度状态**

你还可以直接通过修改节点的 `spec.unschedulable` 字段来设置节点不可调度。

```bash
kubectl cordon <node-name>
```

- `cordon`：将节点标记为不可调度。
- `uncordon`：将节点恢复为可调度。

例如，执行以下命令将节点恢复为可调度状态：

```bash
kubectl uncordon <node-name>
```

### 示例

- **将节点设置为不可调度：**

```bash
kubectl cordon node1
```

- **将节点恢复为可调度：**

```bash
kubectl uncordon node1
```

### 4. **检查节点的调度状态**

可以通过以下命令查看节点是否可调度：

```bash
kubectl get nodes
```

在输出中，`STATUS` 字段为 `SchedulingDisabled` 表示该节点不可调度，正常状态下为 `Ready`。

### 总结

- `kubectl taint nodes <node-name> node.kubernetes.io/unschedulable=true:NoSchedule` 用于将节点标记为不可调度。
- `kubectl cordon <node-name>` 也是将节点设置为不可调度。
- `kubectl uncordon <node-name>` 将节点恢复为可调度状态。
