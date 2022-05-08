# Redis 在 Docker 中的部署问题
#Redis 

目前，Redis 集群不支持 NAT 环境以及重新映射 IP 地址或 TCP 端口的一般环境。

因此 Redis 无法与 Docker 默认的桥接模式兼容，如果要在 Docker 中部署 Redis 集群，则需要使用主机模式，即容器启动时携带 `--net=host` 选项。