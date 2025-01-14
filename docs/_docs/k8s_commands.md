---
title: Kubernetes commands
permalink: /docs/k8s_commands/
redirect_from: /docs/k8s_commands.md
---


## List PODs running on an specific node

```shell
kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName=<node_name>
```

## List Taints of all nodes

```shell
kubectl describe nodes | grep Taint
```

## Restart pod

```shell
kubectl rollout restart daemonset/deployment/statefulset <daemonset/deployment/statefulset>
```