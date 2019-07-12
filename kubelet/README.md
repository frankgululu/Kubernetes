# kubelet 默认的端口及其作用
- 10250--kubelet API: kubelet server 与apiserver通信的端口，定期请求apiserver获取自已所应当处理
  的任务，通过该端口可以访问获取Node资源以及状态
- 10255--readonly API：提供了pod和node信息，接口以只读形式暴露出去，访问该端口不需要认证和鉴权。
- 10248--health chec：通过该端口可以判断kubelet是否正常工作，通过kubelet的启动参数 ```--healthz-port```和 ```--healthz-bind-address```来指定监听的地址和端口。
- 4194---cAdvisor Listen：kubelet通过该端口可以获取到该节点的环境信息以及node上运行的容器状态等内容，访问http://localhost:4194 可以看到cAdvisor的管理界面，通过kubelet的启动参数 ```--cadvisor-port```可以指定启动的端口。
