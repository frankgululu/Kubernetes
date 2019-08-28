# Kubernetes
记录在使用Kubernetes当中的盲点
- Configmap 动态跟新命令
```bash
kubectl create configmap foo --from-file foo.properties -o yaml --dry-run | kubectl replace -f -
```

