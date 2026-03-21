# Nginx 配置示例集合

本目录包含多个 Nginx 配置文件，用于测试和验证 Nginx 拓扑分析工具的功能。

## 配置文件列表

### 1. gateway.conf - 公网入口网关
**用途**: 公网入口网关，接收用户请求并分发到不同的服务集群

**包含的 Upstream**:
- `web_cluster`: Web 应用集群（least_conn 负载均衡）
- `api_cluster`: API 服务集群（ip_hash 负载均衡）
- `static_cluster`: 静态资源集群

**预期链路关系**:
```
Client
  ↓
www.example.com:80/443 (gateway.conf)
  ↓ / → web_cluster
  ↓ /api/ → api_cluster  
  ↓ /static/ → static_cluster
```

---

### 2. internal_gateway.conf - 内部 API 网关
**用途**: 内部微服务网关，路由到各个微服务

**包含的 Upstream**:
- `auth_service`: 认证服务
- `user_service`: 用户服务
- `order_service`: 订单服务
- `payment_service`: 支付服务
- `product_service`: 商品服务

**预期链路关系**:
```
Client
  ↓
api-gateway.internal:80 (internal_gateway.conf)
  ↓ /api/auth → auth_service
  ↓ /api/users → user_service
  ↓ /api/orders → order_service
  ↓ /api/payments → payment_service
  ↓ /api/products → product_service
```

---

### 3. database_proxy.conf - 数据库代理层
**用途**: 数据库和缓存服务代理

**包含的 Upstream**:
- `mysql_primary`: MySQL 主库
- `mysql_replica`: MySQL 从库集群（least_conn）
- `redis_cluster`: Redis 集群
- `mongodb_cluster`: MongoDB 集群

**预期链路关系**:
```
Client
  ↓
db-proxy.internal:8080 (database_proxy.conf)
  ↓ /mysql/write → mysql_primary
  ↓ /mysql/read → mysql_replica
  ↓ /redis → redis_cluster
  ↓ /mongodb → mongodb_cluster
```

---

### 4. middleware.conf - 中间件网关
**用途**: Kafka、RabbitMQ、Elasticsearch 等中间件代理

**包含的 Upstream**:
- `kafka_broker`: Kafka 消息队列
- `rabbitmq_cluster`: RabbitMQ 集群
- `elasticsearch_cluster`: ES 集群
- `prometheus_server`: Prometheus 监控

**预期链路关系**:
```
Client
  ↓
middleware-gateway.internal:9091 (middleware.conf)
  ↓ /kafka → kafka_broker
  ↓ /rabbitmq → rabbitmq_cluster
  ↓ /elasticsearch → elasticsearch_cluster
  ↓ /metrics → prometheus_server
```

---

### 5. cdn_edge.conf - CDN 边缘节点
**用途**: CDN 边缘节点，缓存静态资源并代理到源站

**包含的 Upstream**:
- `origin_server`: 源站服务器
- `cdn_cache`: CDN 缓存服务器

**预期链路关系**:
```
Client
  ↓
cdn-edge-01.example.com:80/443 (cdn_edge.conf)
  ↓ /images/ → cdn_cache → origin_server
  ↓ /videos/ → origin_server
  ↓ /static/ → cdn_cache → origin_server
```

---

### 6. lb_entry.conf - 负载均衡入口
**用途**: 最外层负载均衡器，代理到内部 Nginx

**预期链路关系**:
```
Client
  ↓
load-balancer.example.com:80/443 (lb_entry.conf)
  ↓ → 10.0.1.100 (内部 Nginx)
```

---

### 7. microservices.conf - 微服务架构
**用途**: 微服务网关和内部服务

**包含的 Upstream**:
- `gateway_service`: API 网关
- `user_service_v1`: 用户服务
- `order_service_v1`: 订单服务
- `payment_gateway`: 支付网关
- `notification_service`: 通知服务

**预期链路关系**:
```
Client
  ↓
api.microservices.local:80 (microservices.conf)
  ↓ /api/v1/gateway → gateway_service
  ↓ /api/v1/users → user_service_v1
  ↓ /api/v1/orders → order_service_v1
  ↓ /api/v1/payments → payment_gateway
  ↓ /api/v1/notifications → notification_service
```

---

### 8. backup_cluster.conf - 备份和灾备集群
**用途**: 主备切换和灾难恢复

**包含的 Upstream**:
- `primary_web_cluster`: 主集群
- `backup_web_cluster`: 备用集群
- `disaster_recovery`: 灾备集群

**预期链路关系**:
```
Client
  ↓
primary.example.com:80 (backup_cluster.conf)
  ↓ → primary_web_cluster (主集群)

Client
  ↓
backup.example.com:80 (backup_cluster.conf)
  ↓ → backup_web_cluster (备用集群)

Client
  ↓
dr.example.com:80 (backup_cluster.conf)
  ↓ → disaster_recovery (灾备集群)
```

---

### 9. complex_test.conf - 复杂测试配置
**用途**: 综合测试配置，包含多种负载均衡算法

**包含的 Upstream**:
- `app_cluster_alpha`: 应用集群 A（least_conn）
- `app_cluster_beta`: 应用集群 B（ip_hash）
- `database_write`: 数据库写库
- `database_read`: 数据库读库集群（round_robin）
- `cache_servers`: 缓存服务器集群

**预期链路关系**:
```
Client
  ↓
alpha.example.com:80 (complex_test.conf)
  ↓ / → app_cluster_alpha
  ↓ /beta/ → app_cluster_beta

Client
  ↓
beta.example.com:80 (complex_test.conf)
  ↓ / → app_cluster_beta

Client
  ↓
db-proxy.example.com:8080 (complex_test.conf)
  ↓ /write → database_write
  ↓ /read → database_read
  ↓ /cache → cache_servers
```

---

## 验证测试建议

### 测试 1: 基础功能测试
上传以下配置文件:
1. `gateway.conf`
2. `internal_gateway.conf`
3. `complex_test.conf`

**验证点**:
- ✅ 正确识别所有 upstream
- ✅ 正确识别所有 server 和 location
- ✅ 正确建立链路关系

### 测试 2: 多层转发测试
上传以下配置文件:
1. `lb_entry.conf` (外层)
2. `gateway.conf` (中间层)
3. `internal_gateway.conf` (内层)

**验证点**:
- ✅ 正确显示多层转发链路
- ✅ Client → LB → Gateway → Internal Gateway

### 测试 3: 完整微服务架构测试
上传所有配置文件:
1. `gateway.conf`
2. `internal_gateway.conf`
3. `database_proxy.conf`
4. `middleware.conf`
5. `microservices.conf`
6. `complex_test.conf`

**验证点**:
- ✅ 正确显示所有节点和链路
- ✅ 按层级清晰展示拓扑
- ✅ 链路关系正确连线

### 测试 4: 备份和灾备测试
上传以下配置文件:
1. `backup_cluster.conf`
2. `gateway.conf`

**验证点**:
- ✅ 识别 backup 节点
- ✅ 识别灾备链路

## 预期拓扑结构（全部上传）

```
Layer 0: Client
  ↓
Layer 1: 入口网关
  ├─ load-balancer.example.com
  ├─ www.example.com (gateway.conf)
  ├─ api-gateway.internal (internal_gateway.conf)
  ├─ db-proxy.internal (database_proxy.conf)
  ├─ middleware-gateway.internal (middleware.conf)
  ├─ api.microservices.local (microservices.conf)
  ├─ primary/backup/dr.example.com
  ├─ alpha/beta.example.com
  └─ cdn-edge-01.example.com
  ↓
Layer 2: Upstream 服务池
  ├─ web_cluster, api_cluster, static_cluster
  ├─ auth_service, user_service, order_service, payment_service, product_service
  ├─ mysql_primary/replica, redis_cluster, mongodb_cluster
  ├─ kafka_broker, rabbitmq_cluster, elasticsearch_cluster, prometheus_server
  ├─ gateway_service, user_service_v1, order_service_v1, payment_gateway, notification_service
  ├─ app_cluster_alpha/beta, database_write/read, cache_servers
  └─ origin_server, cdn_cache
  ↓
Layer 3: 后端服务器
  └─ 192.168.x.x, 10.0.x.x, 172.16.x.x, etc.
```

## 负载均衡算法覆盖

本配置集合覆盖了所有常见的负载均衡算法:

| 算法 | 配置文件 | Upstream |
|------|----------|-----------|
| round_robin | complex_test.conf | database_read |
| least_conn | gateway.conf, database_proxy.conf | web_cluster, mysql_replica |
| ip_hash | internal_gateway.conf, microservices.conf, complex_test.conf | api_cluster, gateway_service, app_cluster_beta |
| weight | 所有配置 | 多个 upstream |

## 特殊配置覆盖

- ✅ backup 节点: backup_cluster.conf
- ✅ SSL/HTTPS: gateway.conf, cdn_edge.conf
- ✅ 多端口监听: 所有配置文件
- ✅ proxy_next_upstream: backup_cluster.conf
- ✅ 缓存配置: cdn_edge.conf
