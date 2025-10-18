---
title: SpringCloud ZooKeeper 详解，以及与Go、Rust等非Java服务的集成
link: springcloud-zookeeper
date: 2023-10-24 08:41:51
tags:
  - Spring
  - ZooKeeper
---
ZooKeeper，是一个开源的分布式协调服务，不仅支持分布式选举、任务分配，还可以用于微服务的注册中心和配置中心。本文，我们将深入探讨ZooKeeper用做微服务注册中心的场景。

# ZooKeeper中的服务注册路径

SpringCloud ZooKeeper遵循特定的路径结构进行服务注册
```
/services/${spring.application.name}/${serviceId}
```
示例：
```
/services/provider-service/d87a3891-1173-45a0-bdfa-a1b60c71ef4e
```

`/services`和`/${spring.application.name}`是ZooKeeper中的永久节点，`/${serviceId}`是临时节点，当服务下线时，ZooKeeper会自动删除该节点。

注：当微服务的最后一个实例下线时，SpringCloud ZooKeeper框架会删除`/${spring.application.name}`节点。

# ZooKeeper中的服务注册数据

下面是一个典型的服务注册内容示例：

```json
{
    "name":"provider-service",
    "id":"d87a3891-1173-45a0-bdfa-a1b60c71ef4e",
    "address":"192.168.0.105",
    "port":8080,
    "sslPort":null,
    "payload":{
        "@class":"org.springframework.cloud.zookeeper.discovery.ZookeeperInstance",
        "id":"provider-service",
        "name":"provider-service",
        "metadata":{
            "instance_status":"UP"
        }
    },
    "registrationTimeUTC":1695401004882,
    "serviceType":"DYNAMIC",
    "uriSpec":{
        "parts":[
            {
                "value":"scheme",
                "variable":true
            },
            {
                "value":"://",
                "variable":false
            },
            {
                "value":"address",
                "variable":true
            },
            {
                "value":":",
                "variable":false
            },
            {
                "value":"port",
                "variable":true
            }
        ]
    }
}
```
其中，address、port和uriSpec是最核心的数据。uriSpec中的parts区分了哪些内容是可变的，哪些是固定的。

# SpringCloud 服务使用OpenFeign互相调用

一旦两个微服务都注册到了ZooKeeper，那么它们就可以通过OpenFeign互相调用了。简单的示例如下

## 服务提供者

### 创建SpringBoot项目

创建SpringBoot项目，并添加**spring-cloud-starter-zookeeper-discovery**和**spring-boot-starter-web**依赖。

### 配置application.yaml

```yaml
spring:
  application:
    name: provider-service
  cloud:
    zookeeper:
      connect-string: localhost:2181

server:
  port: 8082
```

### 注册到ZooKeeper

在启动类上添加`@EnableDiscoveryClient`注解。

### 创建一个简单的REST接口

```java
@RestController
public class ProviderController {
    @GetMapping("/hello")
    public String hello() {
        return "Hello from Provider Service!";
    }
}
```

## 服务消费者

### 创建SpringBoot项目

创建SpringBoot项目，并添加**spring-cloud-starter-zookeeper-discovery**、**spring-cloud-starter-openfeign**和**spring-boot-starter-web**依赖。

### 配置application.yaml

```yaml
spring:
  application:
    name: consumer-service
  cloud:
    zookeeper:
      connect-string: localhost:2181

server:
  port: 8081
```

### 注册到ZooKeeper

在启动类上添加`@EnableDiscoveryClient`注解。

### 创建一个REST接口，通过OpenFeign调用服务提供者

```java
@RestController
public class ConsumerController {
    
    @Autowired
    private ProviderClient providerClient;

    @GetMapping("/getHello")
    public String getHello() {
        return providerClient.hello();
    }
}
```

## 运行效果
```
curl localhost:8081/getHello -i
HTTP/1.1 200
Content-Type: text/plain;charset=UTF-8
Content-Length: 28
Date: Wed, 18 Oct 2023 02:40:57 GMT

Hello from Provider Service!
```

# 非Java服务在SpringCloud ZooKeeper中注册

可能有些读者乍一看觉得有点奇怪，为什么要在SpringCloud ZooKeeper中注册非Java服务呢？没有这个应用场景。

当然，这样的场景比较少，常见于大部分项目都是用SpringCloud开发，但有少部分项目因为种种原因，不得不使用其他语言开发，比如Go、Rust等。这时候，我们就需要在SpringCloud ZooKeeper中注册非Java服务了。

对于非JVM语言开发的服务，只需确保它们提供了Rest/HTTP接口并正确地注册到ZooKeeper，就可以被SpringCloud的Feign客户端所调用。

## Go服务在SpringCloud ZooKeeper

example代码组织：
```
├── consumer
│   └── consumer.go
├── go.mod
├── go.sum
└── provider
    └── provider.go
```

### Go服务提供者在SpringCloud ZooKeeper

注：该代码的质量为demo级别，实际生产环境需要更加严谨的代码，如重连机制、超时机制、更优秀的服务ID生成算法等。

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"time"

	"encoding/json"
	"github.com/gin-gonic/gin"
	"github.com/samuel/go-zookeeper/zk"
)

const (
	zkServers = "localhost:2181" // Zookeeper服务器地址
)

func main() {
	// 初始化gin框架
	r := gin.Default()

	// 添加一个简单的hello接口
	r.GET("/hello", func(c *gin.Context) {
		c.String(http.StatusOK, "Hello from Go service!")
	})

	// 注册服务到zookeeper
	registerToZookeeper()

	// 启动gin服务器
	r.Run(":8080")
}

func registerToZookeeper() {
	conn, _, err := zk.Connect([]string{zkServers}, time.Second*5)
	if err != nil {
		panic(err)
	}

	// 检查并创建父级路径
	ensurePathExists(conn, "/services")
	ensurePathExists(conn, "/services/provider-service")

	// 构建注册的数据
	data, _ := json.Marshal(map[string]interface{}{
		"name":        "provider-service",
		"address":     "127.0.0.1",
		"port":        8080,
		"sslPort":     nil,
		"payload":     map[string]interface{}{"@class": "org.springframework.cloud.zookeeper.discovery.ZookeeperInstance", "id": "provider-service", "name": "provider-service", "metadata": map[string]string{"instance_status": "UP"}},
		"serviceType": "DYNAMIC",
		"uriSpec": map[string]interface{}{
			"parts": []map[string]interface{}{
				{"value": "scheme", "variable": true},
				{"value": "://", "variable": false},
				{"value": "address", "variable": true},
				{"value": ":", "variable": false},
				{"value": "port", "variable": true},
			},
		},
	})

	// 在zookeeper中注册服务
	path := "/services/provider-service/" + generateServiceId()
	_, err = conn.Create(path, data, zk.FlagEphemeral, zk.WorldACL(zk.PermAll))
	if err != nil {
		log.Fatalf("register service error: %s", err)
	} else {
		log.Println(path)
	}
}

func ensurePathExists(conn *zk.Conn, path string) {
	exists, _, err := conn.Exists(path)
	if err != nil {
		log.Fatalf("check path error: %s", err)
	}
	if !exists {
		_, err := conn.Create(path, []byte{}, 0, zk.WorldACL(zk.PermAll))
		if err != nil {
			log.Fatalf("create path error: %s", err)
		}
	}
}

func generateServiceId() string {
	// 这里简化为使用当前时间生成ID，实际生产环境可能需要更复杂的算法
	return fmt.Sprintf("%d", time.Now().UnixNano())
}
```

调用效果

```
curl localhost:8081/getHello -i
HTTP/1.1 200
Content-Type: text/plain;charset=UTF-8
Content-Length: 28
Date: Wed, 18 Oct 2023 02:43:52 GMT

Hello from Go Service!
```

## Go服务消费者在SpringCloud ZooKeeper

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"time"

	"github.com/samuel/go-zookeeper/zk"
)

const (
	zkServers = "localhost:2181" // Zookeeper服务器地址
)

var conn *zk.Conn

func main() {
	// 初始化ZooKeeper连接
	initializeZookeeper()

	// 获取服务信息
	serviceInfo := getServiceInfo("/services/provider-service")
	fmt.Println("Fetched service info:", serviceInfo)

	port := int(serviceInfo["port"].(float64))

	resp, err := http.Get(fmt.Sprintf("http://%s:%d/hello", serviceInfo["address"], port))
	if err != nil {
		panic(err)
	}

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		panic(err)
	}

	fmt.Println(string(body))
}

func initializeZookeeper() {
	var err error
	conn, _, err = zk.Connect([]string{zkServers}, time.Second*5)
	if err != nil {
		log.Fatalf("Failed to connect to ZooKeeper: %s", err)
	}
}

func getServiceInfo(path string) map[string]interface{} {
	children, _, err := conn.Children(path)
	if err != nil {
		log.Fatalf("Failed to get children of %s: %s", path, err)
	}

	if len(children) == 0 {
		log.Fatalf("No services found under %s", path)
	}

	// 这里只获取第一个服务节点的信息作为示例，实际上可以根据负载均衡策略选择一个服务节点
	data, _, err := conn.Get(fmt.Sprintf("%s/%s", path, children[0]))
	if err != nil {
		log.Fatalf("Failed to get data of %s: %s", children[0], err)
	}

	var serviceInfo map[string]interface{}
	if err := json.Unmarshal(data, &serviceInfo); err != nil {
		log.Fatalf("Failed to unmarshal data: %s", err)
	}

	return serviceInfo
}
```

## Rust服务在SpringCloud ZooKeeper

example代码组织：
```
├── Cargo.lock
├── Cargo.toml
└── src
    └── bin
        ├── consumer.rs
        └── provider.rs
```

### Rust服务提供者在SpringCloud ZooKeeper

```rust
use std::collections::HashMap;
use std::time::Duration;
use serde_json::Value;
use warp::Filter;
use zookeeper::{Acl, CreateMode, WatchedEvent, Watcher, ZooKeeper};

static ZK_SERVERS: &str = "localhost:2181";

static mut ZK_CONN: Option<ZooKeeper> = None;

struct LoggingWatcher;
impl Watcher for LoggingWatcher {
    fn handle(&self, e: WatchedEvent) {
        println!("WatchedEvent: {:?}", e);
    }
}

#[tokio::main]
async fn main() {
    let hello = warp::path!("hello").map(|| warp::reply::html("Hello from Rust service!"));
    register_to_zookeeper().await;

    warp::serve(hello).run(([127, 0, 0, 1], 8083)).await;
}

async fn register_to_zookeeper() {
    unsafe {
        ZK_CONN = Some(ZooKeeper::connect(ZK_SERVERS, Duration::from_secs(5), LoggingWatcher).unwrap());
        let zk = ZK_CONN.as_ref().unwrap();

        let path = "/services/provider-service";
        if zk.exists(path, false).unwrap().is_none() {
            zk.create(path, vec![], Acl::open_unsafe().clone(), CreateMode::Persistent).unwrap();
        }

        let service_data = get_service_data();
        let service_path = format!("{}/{}", path, generate_service_id());
        zk.create(&service_path, service_data, Acl::open_unsafe().clone(), CreateMode::Ephemeral).unwrap();
    }
}

fn get_service_data() -> Vec<u8> {
    let mut data: HashMap<&str, Value> = HashMap::new();
    data.insert("name", serde_json::Value::String("provider-service".to_string()));
    data.insert("address", serde_json::Value::String("127.0.0.1".to_string()));
    data.insert("port", serde_json::Value::Number(8083.into()));
    serde_json::to_vec(&data).unwrap()
}

fn generate_service_id() -> String {
    format!("{}", chrono::Utc::now().timestamp_nanos())
}
```

### Rust服务消费者在SpringCloud ZooKeeper

```rust
use std::collections::HashMap;
use std::time::Duration;
use zookeeper::{WatchedEvent, Watcher, ZooKeeper};
use reqwest;
use serde_json::Value;

static ZK_SERVERS: &str = "localhost:2181";

struct LoggingWatcher;
impl Watcher for LoggingWatcher {
    fn handle(&self, e: WatchedEvent) {
        println!("WatchedEvent: {:?}", e);
    }
}

#[tokio::main]
async fn main() {
    let provider_data = fetch_provider_data_from_zookeeper().await;
    let response = request_provider(&provider_data).await;
    println!("Response from provider: {}", response);
}

async fn fetch_provider_data_from_zookeeper() -> HashMap<String, Value> {
    let zk = ZooKeeper::connect(ZK_SERVERS, Duration::from_secs(5), LoggingWatcher).unwrap();

    let children = zk.get_children("/services/provider-service", false).unwrap();
    if children.is_empty() {
        panic!("No provider services found!");
    }

    // For simplicity, we just take the first child (i.e., service instance). 
    // In a real-world scenario, load balancing strategies would determine which service instance to use.
    let data = zk.get_data(&format!("/services/provider-service/{}", children[0]), false).unwrap();
    serde_json::from_slice(&data.0).unwrap()
}

async fn request_provider(provider_data: &HashMap<String, Value>) -> String {
    let address = provider_data.get("address").unwrap().as_str().unwrap();
    let port = provider_data.get("port").unwrap().as_i64().unwrap();
    let url = format!("http://{}:{}/hello", address, port);

    let response = reqwest::get(&url).await.unwrap();
    response.text().await.unwrap()
}
```
