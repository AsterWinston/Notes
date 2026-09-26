
# 1. OpenAPI 概述

Nacos 提供了完整的 REST API，支持所有核心功能的程序化操作。API 设计遵循 RESTful 风格，返回统一的 JSON 响应格式。

## 1.1 通用响应格式

```json
{
  "code": 200,
  "message": "success",
  "data": {}
}
```

## 1.2 通用返回码

| 状态码 | 含义 |
|--------|------|
| 200 | 成功 |
| 400 | 客户端请求错误 |
| 403 | 未授权（鉴权模式下） |
| 404 | 资源不存在 |
| 500 | 服务端内部错误 |

# 2. 命名服务 API

## 2.1 服务注册

```
POST /nacos/v1/ns/instance
```

**请求参数**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| serviceName | String | 是 | 服务名 |
| groupName | String | 否 | 分组名（默认 DEFAULT_GROUP） |
| ip | String | 是 | 实例 IP |
| port | int | 是 | 实例端口 |
| namespaceId | String | 否 | 命名空间 ID（默认 public） |
| weight | double | 否 | 权重（默认 1） |
| enabled | boolean | 否 | 是否启用（默认 true） |
| healthy | boolean | 否 | 是否健康（默认 true） |
| ephemeral | boolean | 否 | 是否临时实例（默认 true） |
| clusterName | String | 否 | 集群名（默认 DEFAULT） |
| metadata | String | 否 | 元数据 JSON 字符串 |

```bash
# 注册服务实例
curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'serviceName=order-service&ip=192.168.1.1&port=8080&weight=5'

# 注册带元数据的实例
curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance' \
  -d 'serviceName=order-service&ip=192.168.1.2&port=8080&metadata={"version":"v2","zone":"sh"}'
```

## 2.2 服务发现

```
GET /nacos/v1/ns/instance/list
```

**请求参数**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| serviceName | String | 是 | 服务名 |
| groupName | String | 否 | 分组名 |
| namespaceId | String | 否 | 命名空间 ID |
| clusters | String | 否 | 集群名，多集群逗号分隔 |
| healthyOnly | boolean | 否 | 是否只返回健康实例（默认 false） |

```bash
# 查询全部实例
curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=order-service'

# 只查询健康的
curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=order-service&healthyOnly=true'

# 查询指定集群
curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=order-service&clusters=SH'
```

**返回示例**：

```json
{
  "name": "DEFAULT_GROUP@@order-service",
  "groupName": "DEFAULT_GROUP",
  "clusters": "SH",
  "cacheMillis": 10000,
  "hosts": [
    {
      "instanceId": "192.168.1.1#8080#SH#order-service",
      "ip": "192.168.1.1",
      "port": 8080,
      "weight": 5.0,
      "healthy": true,
      "enabled": true,
      "ephemeral": true,
      "clusterName": "SH",
      "serviceName": "order-service",
      "metadata": {
        "version": "v2.0.0",
        "zone": "sh"
      },
      "instanceHeartBeatInterval": 5000,
      "lastBeat": 1700000000000
    }
  ],
  "cacheMillis": 10000,
  "lastRefTime": 1700000000000,
  "checksum": "abc123def456",
  "allIPs": false,
  "reachProtectionThreshold": false
}
```

## 2.3 服务注销

```
DELETE /nacos/v1/ns/instance
```

```bash
curl -X DELETE 'http://127.0.0.1:8848/nacos/v1/ns/instance' \
  -d 'serviceName=order-service&ip=192.168.1.1&port=8080'
```

## 2.4 实例健康检查心跳

```
PUT /nacos/v1/ns/instance/beat
```

```bash
curl -X PUT 'http://127.0.0.1:8848/nacos/v1/ns/instance/beat' \
  -d 'serviceName=order-service&beat={"ip":"192.168.1.1","port":8080,"cluster":"SH","weight":5}&ip=192.168.1.1&port=8080'
```

## 2.5 实例详情查询

```
GET /nacos/v1/ns/instance
```

```bash
curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instance?serviceName=order-service&ip=192.168.1.1&port=8080'
```

## 2.6 服务列表查询

```
GET /nacos/v1/ns/service/list
```

```bash
# 当前页，每页大小
curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/service/list?pageNo=1&pageSize=10'

# 按命名空间和分组过滤
curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/service/list?namespaceId=dev&groupName=DEFAULT_GROUP'
```

## 2.7 服务详情查询

```
GET /nacos/v1/ns/service
```

```bash
curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/service?serviceName=order-service'
```

## 2.8 更新实例

```
PUT /nacos/v1/ns/instance
```

```bash
# 更新实例权重和健康状态
curl -X PUT 'http://127.0.0.1:8848/nacos/v1/ns/instance' \
  -d 'serviceName=order-service&ip=192.168.1.1&port=8080&weight=10&enabled=false'
```

# 3. 配置管理 API

## 3.1 发布/更新配置

```
POST /nacos/v1/cs/configs
```

```bash
curl -X POST 'http://127.0.0.1:8848/nacos/v1/cs/configs' \
  -d 'dataId=order-service-dev.yaml&group=DEFAULT_GROUP&content=server:\n  port: 8080\n\nspring:\n  datasource:\n    url: jdbc:mysql://localhost:3306/order_db'
```

## 3.2 获取配置

```
GET /nacos/v1/cs/configs
```

```bash
curl -X GET 'http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=order-service-dev.yaml&group=DEFAULT_GROUP'
```

## 3.3 删除配置

```
DELETE /nacos/v1/cs/configs
```

```bash
curl -X DELETE 'http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=order-service-dev.yaml&group=DEFAULT_GROUP'
```

## 3.4 监听配置变更（长轮询）

```
POST /nacos/v1/cs/configs/listener
```

```bash
# 格式: dataId^2group^2md5^2tenant^1
# %02 = ^2 (分隔符)
# %01 = ^1 (行分隔符)
curl -X POST 'http://127.0.0.1:8848/nacos/v1/cs/configs/listener' \
  -H 'Long-Pulling-Timeout: 30000' \
  -d 'Listening-Configs=order-service.yaml%02DEFAULT_GROUP%02d41d8cd98f00b204e9800998ecf8427e%02%01'
```

## 3.5 查询配置历史版本

```
GET /nacos/v1/cs/history
```

```bash
curl -X GET 'http://127.0.0.1:8848/nacos/v1/cs/history?dataId=order-service-dev.yaml&group=DEFAULT_GROUP&pageNo=1&pageSize=10'
```

## 3.6 查询配置详情（含历史版本）

```
GET /nacos/v1/cs/history/config
```

```bash
curl -X GET 'http://127.0.0.1:8848/nacos/v1/cs/history/config?nid=12345&dataId=order-service-dev.yaml&group=DEFAULT_GROUP'
```

# 4. 集群管理 API

## 4.1 获取集群节点

```
GET /nacos/v1/core/cluster/nodes
```

```bash
curl -X GET 'http://127.0.0.1:8848/nacos/v1/core/cluster/nodes'
```

**返回示例**：

```json
{
  "code": 200,
  "message": null,
  "data": [
    {
      "address": "10.0.0.1:8848",
      "state": "UP",
      "failAccessCnt": 0,
      "abilities": {
        "remoteAbility": {
          "supportRemoteConnection": true
        },
        "configAbility": {
          "supportRemoteMetrics": false
        }
      }
    }
  ]
}
```

## 4.2 查看自身节点

```
GET /nacos/v1/core/cluster/self
```

```bash
curl -X GET 'http://127.0.0.1:8848/nacos/v1/core/cluster/self'
```

## 4.3 更新集群节点

```
PUT /nacos/v1/core/cluster/nodes
```

```bash
curl -X PUT 'http://127.0.0.1:8848/nacos/v1/core/cluster/nodes' \
  -d 'addresses=10.0.0.1:8848,10.0.0.2:8848'
```

## 4.4 健康检查

```
GET /nacos/v1/console/health/readiness
GET /nacos/v1/console/health/liveness
```

# 5. 命名空间 API

## 5.1 查询命名空间列表

```
GET /nacos/v1/console/namespaces
```

```bash
curl -X GET 'http://127.0.0.1:8848/nacos/v1/console/namespaces'
```

## 5.2 创建命名空间

```
POST /nacos/v1/console/namespaces
```

```bash
curl -X POST 'http://127.0.0.1:8848/nacos/v1/console/namespaces' \
  -d 'customNamespaceId=dev-namespace&namespaceName=dev&namespaceDesc=Development'
```

## 5.3 修改/删除命名空间

```bash
# 修改
curl -X PUT 'http://127.0.0.1:8848/nacos/v1/console/namespaces' \
  -d 'namespaceId=dev-namespace&namespaceName=develop&namespaceDesc=dev'

# 删除
curl -X DELETE 'http://127.0.0.1:8848/nacos/v1/console/namespaces' \
  -d 'namespaceId=dev-namespace'
```

# 6. SDK 示例

## 6.1 Java SDK 使用指南

```java
// 1. 服务发现与注册
public class NamingSdkDemo {
    public static void main(String[] args) throws NacosException {
        Properties properties = new Properties();
        properties.setProperty("serverAddr", "127.0.0.1:8848");
        properties.setProperty("namespace", "dev");
        
        NamingService naming = NacosFactory.createNamingService(properties);
        
        // 注册
        naming.registerInstance("order-service", "192.168.1.100", 8080, "SH");
        
        // 发现
        List<Instance> instances = naming.getAllInstances("order-service");
        for (Instance instance : instances) {
            System.out.println(instance.getIp() + ":" + instance.getPort());
        }
        
        // 订阅
        naming.subscribe("order-service", event -> {
            System.out.println("Instances changed!");
        });
    }
}

// 2. 配置管理
public class ConfigSdkDemo {
    public static void main(String[] args) throws NacosException {
        Properties properties = new Properties();
        properties.setProperty("serverAddr", "127.0.0.1:8848");
        
        ConfigService configService = NacosFactory.createConfigService(properties);
        
        // 获取配置
        String config = configService.getConfig("order-service-dev.yaml", 
            "DEFAULT_GROUP", 5000);
        System.out.println(config);
        
        // 发布配置
        configService.publishConfig("order-service-dev.yaml", 
            "DEFAULT_GROUP", "server.port: 8080");
        
        // 监听变更
        configService.addListener("order-service-dev.yaml", 
            "DEFAULT_GROUP", new Listener() {
                @Override
                public Executor getExecutor() {
                    return null;
                }
                
                @Override
                public void receiveConfigInfo(String configInfo) {
                    System.out.println("Config updated: " + configInfo);
                }
            });
    }
}
```

## 6.2 Go SDK 示例

```go
package main

import (
    "fmt"
    "github.com/nacos-group/nacos-sdk-go/v2/clients"
    "github.com/nacos-group/nacos-sdk-go/v2/common/constant"
    "github.com/nacos-group/nacos-sdk-go/v2/vo"
)

func main() {
    // 创建客户端配置
    sc := []constant.ServerConfig{
        *constant.NewServerConfig("127.0.0.1", 8848),
    }
    cc := *constant.NewClientConfig(
        constant.WithNamespaceId("public"),
        constant.WithTimeoutMs(5000),
        constant.WithNotLoadCacheAtStart(true),
    )
    
    // 创建命名服务客户端
    namingClient, _ := clients.NewNamingClient(
        vo.NacosClientParam{
            ServerConfigs: sc,
            ClientConfig:  &cc,
        },
    )
    
    // 注册实例
    namingClient.RegisterInstance(vo.RegisterInstanceParam{
        ServiceName: "order-service",
        Ip:          "192.168.1.100",
        Port:        8080,
        Weight:      5,
        Enable:      true,
        Healthy:     true,
    })
    
    // 获取实例列表
    instances, _ := namingClient.SelectAllInstances(vo.SelectAllInstancesParam{
        ServiceName: "order-service",
    })
    
    for _, instance := range instances {
        fmt.Printf("%s:%d\n", instance.Ip, instance.Port)
    }
}
```

## 6.3 Python SDK 示例

```python
import nacos

# 创建客户端
SERVER_ADDRESSES = "127.0.0.1:8848"
NAMESPACE = "public"

client = nacos.NacosClient(SERVER_ADDRESSES, namespace=NAMESPACE)

# 注册服务
client.add_naming_instance(
    service_name="order-service",
    ip="192.168.1.100",
    port=8080,
    cluster_name="DEFAULT",
    weight=5.0,
    metadata={"version": "v2.0.0"}
)

# 获取配置
config = client.get_config(
    data_id="order-service-dev.yaml",
    group="DEFAULT_GROUP"
)
print(config)

# 发布配置
client.publish_config(
    data_id="order-service-dev.yaml",
    group="DEFAULT_GROUP",
    content="server.port: 8080"
)
```

# 7. API 鉴权

开启鉴权后，所有 API 请求需要携带 accessToken：

```bash
# 1. 登录获取 accessToken
curl -X POST 'http://127.0.0.1:8848/nacos/v1/auth/login' \
  -d 'username=nacos&password=nacos'

# 返回
# {"accessToken":"eyJhbGciOiJIUzI1NiJ9...","tokenTtl":18000,"globalAdmin":true}

# 2. 使用 accessToken 访问 API
curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=order-service&accessToken=eyJhbGci...'
```

# 8. 实用脚本

## 8.1 批量服务注册脚本

```bash
#!/bin/bash
# 批量注册服务实例

NACOS_URL="http://127.0.0.1:8848"
SERVICE_NAME="order-service"

for i in $(seq 1 10); do
    curl -X POST "${NACOS_URL}/nacos/v1/ns/instance" \
      -d "serviceName=${SERVICE_NAME}&ip=192.168.1.${i}&port=8080&weight=10"
    echo "Registered instance 192.168.1.${i}"
done
```

## 8.2 配置批量导出脚本

```bash
#!/bin/bash
# 导出指定命名空间下所有配置

NACOS_URL="http://127.0.0.1:8848"
NAMESPACE="dev"
ACCESS_TOKEN="your_token"

# 获取配置列表
page=1
while true; do
    response=$(curl -s "${NACOS_URL}/nacos/v1/cs/configs?search=accurate&dataId=&group=&appName=&config_tags=&pageNo=${page}&pageSize=100&tenant=${NAMESPACE}&accessToken=${ACCESS_TOKEN}")
    # 处理响应，导出配置...
    echo "$response"
    page=$((page + 1))
    # 检查是否还有下一页...
done
```
