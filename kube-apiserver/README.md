## kubernetes svc
kube-apiserver 扮演了整个Kubernetes集群管理的入口角色，负责对外暴露Kubernetes API。
为了能够让集群内部的应用与kube-apiserver交互，Kubernetes会在默认命名空间下创建一个名为kubernetes的服务。
```bash
kubectl get svc kubernetes -o wide
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE    SELECTOR
kubernetes   ClusterIP   172.21.0.1   <none>        443/TCP   198d   <none>
```

这也就是有时访问 kubernetes.svc.default:443 这个服务
通过这种方式集群内的应用或者系统主机就可以通过集群内部的DNS域名```kubernetes.default.svc```访问到kube-apiserver。
