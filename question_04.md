四、有一个简单的三层 Web 应用，包含前端（frontend）、中间层（backend），和数据库（database）三部分。这些应用都已经打包成 Docker 镜像，现在需要部署在 Kubernetes 中。
该集群是一个全新部署的集群，除了 kube-proxy, CoreDNS, CNI (Calico) 外没有部署任何应用。请根据下面的要求分别提供部署所需的所有 yaml 定义及部署说明，可以使用 helm 等工具。

具体要求如下：

    1. 前端服务（frontend）：
     - 需要通过外部域名：frontend.example.com访问并支持 https，可以self-signed
     - 支持自动扩缩容，根据 CPU (80%) 使用率在 2 到 10 个 Pod 之间调整实例数量
     - 该 Pod 需要配置 health check, HTTP 协议 9090 端口

    2. 中间层服务（backend）：
    - 仅供前端服务调用，不能对外部暴露
    - 支持通过环境变量来配置对 database 的访问，ENV KEY: DB_HOST/DB_NAME/DB_USER/DB_PASS...
    - 为了保证高可用性，至少要有 3个Pod 同时存活，同时需要 **尽可能** 避免多个 Pod 运行在同一个 Node 上

    3. 数据库服务（database）：
    - 数据库只允许中间层服务访问，不能通过 ClusterIP 之外的方式暴露
    - 需要考虑持久化问题，数据存储在 /var/lib/mysql 目录下, 确保 Pod 崩溃或重建后数据不丢失


Solution:

helm chart 文件位于: [./helm-chart/web-app](./helm-charts/web-app/)
- 首先安装 cert manager: `kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.3/cert-manager.yaml`
- 再安装服务 helm chart： `helm install web-app ./helm-charts/web-app`