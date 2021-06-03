### 部署ingress-controller
官方文档中，直接下载一个全量的部署文件，一键部署所需资源
https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.24.1/deploy/mandatory.yaml

mandatory.yaml 这一个yaml中包含了很多资源的创建，包括namespace，ConfigMap，role,ServiceAccount
等所有部署ingress-controller需要的资源。

在这个文件的deployment部分，镜像用了"quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.24.1"这个，
这个由于墙，在国内不好下载，我这里替换了其他人员贡献的另一个仓库“linuxhub/nginx-ingress-controller:0.24.1”.

这里由于我们使用Daemonset的方式部署，所以我们要改部分配置,注意加注释的部分为修改之后的。
```yaml
# 修改api版本及kind
# kind: Deployment
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
# 删除Replicas
# replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      # 选择对应标签的node
      nodeSelector:
        isIngress: "true"
      # 使用hostNetwork暴露服务
      hostNetwork: true
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.25.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
```
修改之后要给 worknode打上标签

```
kubectl label node node-xxx isIngress="true"
```
修改完之后执行apply, 检查服务和服务日志状态
```
# 执行部署
kubectl apply -f mandatory.yaml
# 检查状态
kubectl get po -n ingress-nginx -o wide
这里可以看到pod部署到了对应的worknode上

# 查看日志
kubectl logs -f --tail 500 nginx-ingress-conrtroller-xxx -n ingress-nginx
```
查看日志，发现nginx-ingress-controller有如下报错
```yaml
ng]string{},OwnerReferences:[],Finalizers:[],ClusterName:,Initializers:nil,ManagedFields:[],}, err services "ingress-nginx" not found
```
我们给ingrss-nginx加一个servcie的对象来消除这个错误日志
```
kubectl apply -f ingress-svc.yaml
```
致此，ingress-controller部署完毕了，接下来暴漏nginx-controller

### 暴漏nginx-controller
到nginx-controller 对应的node上，查看本地端口：
```
root@iZ2zeh4rsjnna82sqvlw6zZ:~# netstat -lntup|grep nginx
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:8181            0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:8181            0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:8181            0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:8181            0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:8181            0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:8181            0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:8181            0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:8181            0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      9705/nginx.conf
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      9705/nginx.conf
tcp6       0      0 :::10254                :::*                    LISTEN      9621/nginx-ingress-
```
由于配置hostnetwork，nginx已经在node主机本地监听端口80/443/8181端口。其中8181是nginx-controller默尔的一个default backend.
如果nginx是高可用的话，可以在多个node上部署，并在前面搭配一套LVS+keepalive 做负载均衡。


### 配置ingress资源
ingress-controller 部署完了之后，接下来创建具体的ingress对象和服务交互路由的。
注意这里配置了https，所以要先创建TLS secret
```
# 创建TLS secret
# 注意替换脚本里的${NAMESPACE} 变量为具体的的名称，这个namespace要和ingress资源在同一个命名空间下。
sh -x create_tls.sh
# 查看secret
kubectl get secret -n xxx
```
执行ingress资源对象
```
# 注意这里的ingress对象要和后端的服务创建在同一个命名空间下面
kubectl apply -f ingress-frontend.yaml
```
查看ingress

```yaml
kubectl get ing -n xxx
kubect describe ing -n xxx

Name:             test-ingress
Namespace:        test
Address:
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
TLS:
  abc-tls  terminates test-ingress.abc.com
Rules:
  Host                  Path  Backends
  ----                  ----  --------
  baas-demo.zhigui.com
                        /   bcs-frontend:80 (172.16.244.20:80)
Annotations:            kubernetes.io/ingress.class: nginx
                        nginx.ingress.kubernetes.io/ssl-passthrough: true
                        nginx.ingress.kubernetes.io/use-regex: true
Events:                 <none>
```
### 测试访问
浏览器里访问 http://test-ingress.abc.com

配置了tls之后， nginx-controller 会在动配置http->https redirect，可以在nginx.conf里看到具体的参数yaml
