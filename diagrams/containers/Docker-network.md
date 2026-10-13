| 模式 | 说明 | 常见用途 |
|---|---|---|
| **bridge** | 默认模式。容器连接到 Docker 创建的虚拟网桥，通常通过 NAT 访问外部；同一用户自定义 bridge 上的容器可直接通信。 | 单机运行多个容器 |
| **host** | 容器直接使用宿主机的网络命名空间，没有独立容器 IP；端口直接占用宿主机端口。 | 对网络性能要求高，或需要使用宿主机网络的场景 |
| **none** | 不配置常规网络接口，容器基本处于隔离状态；仍有 loopback 接口。 | 需要自行配置网络的场景 |
| **container** | 与指定容器共享网络命名空间，包括 IP、端口和网络接口。 | Sidecar 或需要共享网络栈的容器 |
| **overlay** | 在多台 Docker 主机之间创建虚拟网络，让不同主机上的容器互通；通常用于 Swarm。 | Docker Swarm 多主机通信 |

docker run --network host nginx
docker run --network none alpine
docker run --network container:目标容器名 alpine
