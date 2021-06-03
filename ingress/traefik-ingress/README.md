### Traefik Ingress 介绍
traefilk ingress 和 nginx-ingress 一样也是一个统称的叫法，实际都是包含了两部分：
- ingress Controller
- ingress


Traefik Ingress 的部署一共有三种方式Daemonset,Deployment,LoadBalance
LoadBalance的方式一般在公有云的场景中使用。
那么对于Daemonset，Deployment的方式，我们该如何选择，以下是官方文档给出的一些建议：
- Deployment扩展性会好很多，Daemonset是每个节点一个pod。 Deployment相比会需要更少的副本
- Daemonset会自动扩展到新加入的节点上，然而Deployment 按需调度到新的节点上。
- Daemonset确保了一个节点上运行一个pod， Deployment如果想要确保一个节点上只运行一个pod，需要设置亲和性
- Daemonset可以使用**NET_BIND_SERVICE**功能，将允许它绑定到每个主机上的端口80/443。允许它绕过kube-proxy，减少流量跳跃。
- 如果不确定选择哪一个，那么从Daemonset开启。

还有如下的说法，可供参考
- 面向内部(internal)服务的traefik，建议可以使用deployment的方式
- 面向外部(external)服务的traefik，建议可以使用daemonset的方式


### 部署(以DaemonSet的方式部署)
#### 启用RBAC
我们需要向Traefik授予一些权限， 以访问集群中的一些Pod，Endpoint的Service。
我们将启用**ClusterRole** 和 **ClusterRoleBindings**资源。

Ps：如果集群命名空间不会动态变化， 并且traefik无需监控所有命名空间的时候，可以使用
命名空间级别的RoleBindings的小范围授权

为Traefik提供集群中的身份 traefik-sa.yaml

```shell
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
```
```shell
kubectl apply -f traefik-sa.yaml
```

创建一组有权限的ClusterRole，该权限将用于traefi 的 ServiceAccount,
通过这个权限， Traefik可以管理和监视集群中的资源， endpoint，service，secret
以及**extensions** apiGroups 下的ingresses资源。
```shell
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
```
```shell
kubectl apply -f traefik-rbac.yaml
```
#### 部署Traefik Controller到集群
注意，这里我使用的image:v2.2 版本，这里踩了不少坑，花了比较多的时间
最开始用的v1.6的版本，traefik-ingress-controller启动起来之后报如下错误：
```yaml
E0531 03:36:23.895361       1 reflector.go:205] github.com/containous/traefik/vendor/k8s.io/client-go/informers/factory.go:86: Failed to list *v1.Service: v1.ServiceList.Items: []v1.Service: v1.Service.ObjectMeta: v1.ObjectMeta.readObjectFieldAsBytes: expect : after object field, but found p, error found in #10 byte of ...|:{},"k:{\"port\":53,|..., bigger context ...|f:spec":{"f:clusterIP":{},"f:ports":{".":{},"k:{\"port\":53,\"protocol\":\"TCP\"}":{".":{},"f:name":|...
```
经过一番网络的帮助，看有笔者说1.7.26的版本修复了这个问题，升级到1.7.26之后，这个问题果然消失了。

而后一切，都很顺利，包括下面的代理到应用都可以成功访问，只是配置的https不能访问，后来升级到v2.2

** 各个版本之间在配置参数的的时候或者说启动的环境变量，变化比较大，更换时注意 **

```shell
        ports:
        - name: http
          containerPort: 80
          hostPort: 80
        - name: https
          containerPort: 443
          hostPort: 443
        - name: admin
          containerPort: 8080
```
这里在官方的基础上添加了443 port部分

```shell
        args: #这里都是global参数，对于所有的ingress都生效
        - --providers.kubernetesingress
        - --entrypoints.http.Address=:80
        - --entrypoints.http.http.redirections.entryPoint.to=https
        - --entrypoints.https.Address=:443
```
args 部分为ingress-contorller的全局配置，对于所在集群里的所有ingress都会生效，相当于traefik.toml

```shell
kubectl apply -f traefik-ds.yaml

# 检查一下新起来的pod的状态
kubectl get po -n kube-system|grep traefik
```
#### 部署后端应用
这里就是一个常见的deployment管控的pod，和一个默认的clusterip service服务的创建，下面通过ingrees去暴漏，使其有被外部访问的能力。
```shell
kubectl apply -f stilton-deployment.yaml
```
#### 创建TLS secret
```shell
kubectl create secret tls test-stilton  --cert=example.com.pem --key=example.com.key -n test
```
这里的secret要创建在下面和ingress同一个命名空间下
#### 为后端应用创建ingress资源
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: stilton-ingress
  namespace: test
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/router.entrypoints: http, https
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  rules:
  - host: test-stilton.example.com
    http:
      paths:
      - backend:
          serviceName: stilton
          servicePort: http
  tls:
   - secretName: test-stilton
```
注意这里的的ingress对象，要和后端的service在同一个命名空间，"namespace: test",
这里配置了tls，tls的证书以kubernetes的资源对象存储。
http自动跳转https的部分在上面有提到过是配置在了全局的 ingress-contorller里面

#### 配置DNS解析
将test-stilton.example.com域名映射到对应的公网地址，
这里的公网地址可以是worknode的公网地址, 也可以是kubernetes worknode 的LB地址，

#### test
http://test-stilton.example.com, 访问这个地址，会自动跳转至 https://test-stilton.example.com
打开浏览器的debug模式，可以看到301的redirect code。

### 部署(以Deployment的方式部署)

### 部署（以LoadBalance的方式部署)

Ref:
https://doc.traefik.io/traefik/v1.7/user-guide/kubernetes/
https://stackoverflow.com/questions/64581882/traefik-v2-2-ingress-on-kubernetes-http-and-https-cannot-co-exist
https://medium.com/kubernetes-tutorials/deploying-traefik-as-ingress-controller-for-your-kubernetes-cluster-b03a0672ae0c
https://traefik.io/blog/13-key-considerations-when-selecting-an-ingress-controller-for-kubernetes-d3e5d98ed8b7/
