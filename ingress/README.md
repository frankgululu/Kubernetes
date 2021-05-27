### ingress的用途
对于kubernetes集群外部的访问，可以使用NodePort和LoadBalance，为什么还需要ingress呢？

NodePort,对于测试环境使用还可以，当有几十上百的服务在集群中运行时，NodePort的端口管理是灾难。

LoadBalance方式受限于云平台，且通常在云平台部署ELB还需要额外的费用。

因此，k8s提供了一种集群维度的暴漏服务的方式，ingress。它通过独立的ingress对象来制定请求转发
的规则， 把请求路由到一个或多个service中。

ingress规则是很灵活的，可以根据不同域名，不同path转发请求到不同的service，并支持https/http.

### ingress 与 ingress-controller
通常说的ingress，其实分为两个概念， ingress和ingress-controller
- ingress对象
指的是k8s中的一个api对象，一般用yaml配置。作用是定义请求如何转发到service的规则。
- ingress-controller
具体实现反向代理及负载均衡的程序， 对ingress定义的规则进行解析，根据配置来实现请求到service的转发。

简单来说， ingress-controller才是负责具体转发的组件，通过各种方式将它暴漏在
集群入口，外部对集群的请求流量会先到ingress-controller, 而ingress对象是告诉ingress-controller 该如何转发请求，比如哪些域名哪些path要
转发到哪些服务等。 


#### ingress-controller
ingress-controller并不是k8s自带的组件，实际上ingress-controller只是一个统称，
用户可以选择不同的ingress-controller实现。目前，由k8s 维护的ingree-controller只有google云
的GCE与ingress-nginx两个，其他有很多第三方维护的ingress-controller。

不管哪一种ingress-controller，实现的机制都是大同小异，只是在配置上有差异。
一般来说， ingress-controller的形式是一个pod, 里面跑着demon程序和反向代理程序。daemon 负责不断监控集群的变化，根据ingress对象生成配置并应用新配置到反向代理，比如nginx-controller就是动态生成nginx配置，动态更新upstream，并在需要的时候reload程序应用新配置。
#### ingress
ingress是一个API对象，和其他对象一样，通过yaml文件来配置。
ingress通过http或https暴漏集群内部service，给service提供外部URL，负载均衡，SSL/TLS能力以及
基于host的反向代理。ingress要依靠ingress-controller来具体实现以上功能。

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  namespace: test
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  tls:
  - hosts:
    - test-ingress.abc.com
    secretName: abc-tls
  rules:
  - host: test-ingress.abc.com
    http:
      paths:
      - path: /
        backend:
          serviceName: frontend
          servicePort: 80
  - host: test-api.abc.com
    http:
    paths:
    - path: /
      backend:
        serviceName: api
        serviePort: 8081
  - host: test-api.abc.com
    http:
    paths:
    - path: /image/*
      backend:
        serviceName: image
        serviePort: 8080

```
与其他k8s对象一样，ingress配置也包含了apiVersion，kind,metadat,spec等关键字段。其中spec字段中，tls用于定义https密钥，证书。rule用于指定请求路由规则。

这里值得关注的是metadata.annotations字段。 在ingress配置中，annotations很重要。
前面说ingress-controller有很多不同的实现， 而不同ingress-controller就可以根据"kubernetes.io/ingress.class:"来判断需要使用哪个ingress配置。 同时，不同的ingress-controller也有对应的annotations配置，用于定义一些参数。
如上面配置的"nginx.ingress.kubernetes.io/use-regex: "true"",最终是生成nginx配置中，会采用location ~来表示正则匹配。 

### ingress部署
ingress的部署，需要考虑两个方面：
1. ingress-controller是作为pod运行的，是以什么方式部署比较好。Deployment or DaemonSet
2. inngress解决了把如何请求路由到集群内部，那么自己怎么暴漏给外部比较好。

下面介绍几种常见的部署和暴漏方式。

#### Daemonset+HostWork+nodeSelector(这里也要部署service)
用DaemonSet结合nodeselector来部署ingress-controller到特定的node上，然后使用HostNetwork直接
把该pod与宿主机node的网络打通，直接使用宿主机的80/443端口就能访问服务， 这时，ingress-controller所在node机器类似与传统架构的边缘节点。 改方式请求链路组简单，性能相对NodePort模式更好。
缺点时由于直接利用宿主机节点的网络和端口，一个node只能部署一个ingress-controller pod.
比较适合大并发的生产环境使用。 

具体的部署文档见[DaemonSet+HostNetwork+nodeSelector][deploy/daemonset_hostnetwork/installation_guide.md]  

#### Deployment+NodePort 模式的Service

具体的部署文档见

#### Deployment+LoadBalance模式的Service

具体的部署文档见
