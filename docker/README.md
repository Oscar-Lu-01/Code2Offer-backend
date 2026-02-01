# Code2Offer 部署文档

---

## 🛠 1. 快速启动指南

### 1.1 基础环境
* 已安装 Docker 和 Docker Compose
* 端口 `3306, 6379, 8848, 9200, 5601, 2379, 8858, 8111` 未被占用

### 1.2 目录结构

```
docker/
├── docker-compose-infra.yml     # 1. 基础设施层 (MySQL, Redis, Etcd, Nacos, ES)
├── docker-compose-monitor.yml   # 2. 监控管理层 (Kibana, Sentinel)
├── docker-compose-worker.yml    # 3. 业务计算层 (Hotkey Worker)
├── Dockerfiles/                 # 自定义 Dockerfile
│   ├── Dockerfile-Elasticsearch
│   └── Dockerfile-hotkey-worker
├── jar/                         # 存放编译好的 Jar 包
│   └── hotkey-worker.jar
└── sql/                         # 初始化脚本目录
    ├── init.sql                 # MySQL 自动执行
    └── post_es_mapping.json     # ES 索引模板脚本
```

### 1.3 启动命令
```bash
# 1. 启动基础设施层 (数据持久化层)
docker compose -f docker-compose-infra.yml up -d

# 2. 启动监控管理 (可视面板)
docker compose -f docker-compose-monitor.yml up -d --build

# 3. 启动弹性扩容hotkey-worker
# 3.1 构建并启动单节点 Worker
docker compose -f docker-compose-worker.yml up -d --build
# 3.2 查看 Worker 是否成功注册到 Etcd
docker exec -it etcd etcdctl get "" --prefix
# 3.3 自行弹性扩容
docker compose -f docker-compose-worker.yml up -d --scale hotkey-worker=3
```



## ⚙️2. 服务端口速查表

| **服务名称**      | **外部访问端口** | **容器内通信地址**   | **说明**                               |
| ----------------- | ---------------- | -------------------- | -------------------------------------- |
| **MySQL**         | `3306`           | `mysql:3306`         | 用户: root, 密码: 123456               |
| **Redis**         | `6379`           | `redis:6379`         | 开启 AOF 持久化                        |
| **Nacos**         | `8848/9848`      | `nacos:8848`         | 控制台: http://localhost:8848/nacos    |
| **ES**            | `9200`           | `elasticsearch:9200` | 已集成 IK 中文分词插件                 |
| **Kibana**        | `5601`           | `kibana:5601`        | ES 控制台： http://106.13.184.117:5601 |
| **Etcd**          | `2379`           | `etcd:2379`          | 无需密码验证                           |
| **Sentinel**      | `8858`           | `sentinel:8858`      | 流量治理控制台                         |
| **Hotkey Worker** | `8111`           | `hotkey-worker:8111` | 热点探测计算节点                       |



## 🏗 3. 自定义镜像说明

### Elasticsearch (自定义 IK 分词)

- **配置文件**: `Dockerfile-Elasticsearch`
- **说明**: 构建时会自动下载并安装对应版本的 IK 分词插件。如需更换版本，请修改 Dockerfile 中的下载 URL。

### Hotkey Worker (Java 应用)

- **配置文件**: `Dockerfile-hotkey-worker`
- **构建逻辑**: 该服务会将本地编译好的 Jar 包打包进镜像。
- **注意**: 当你的 Java 代码发生变更并重新打出 Jar 包后，必须运行以下命令生效：