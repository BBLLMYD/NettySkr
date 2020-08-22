## 轻量RPC通信

一个完备的 RPC 框架在实现远程调用的基础上通常还有着健康监测、服务治理等方面的相关设施，以及易扩展的良好设计。

这里主要是以学习为目的，围绕通信的部分实践一个轻量级的 RPC 框架。


- [1. 基础实现](https://github.com/BBLLMYD/netty-stroll#1-%E5%9F%BA%E7%A1%80%E5%AE%9E%E7%8E%B0)
- [2. 说明](https://github.com/BBLLMYD/netty-stroll#2-%E8%AF%B4%E6%98%8E)
- [3. 应用示例](https://github.com/BBLLMYD/netty-stroll#3-%E5%BA%94%E7%94%A8%E7%A4%BA%E4%BE%8B)

---

### 1. 基础实现

- **传输 & 协议**

    采用 TCP 协议为通信基础，基于 Netty 自定义数据包格式，心跳机制维持 TCP 单一长链接

- **注册中心**

    缺省基于 ZooKeeper 实现服务注册和服务发现，可扩展

- **序列化**

    默认基于 ProtoStuff 实现序列化机制，可扩展
    
- **负载均衡**

    默认 Random 访问，可扩展
    
    ...
    
---

### 2. 说明

- **signal-base** 
    
    封装了上述提到的 RPC 各**基础组件和扩展点**；同时将需要发布的上层接口放在 common.service 包下发布
        
<div align=center><img src="https://github.com/BBLLMYD/netty-stroll/blob/master/other/baset.png?raw=true" width="423" alt="RPC基础模式" ></div>

- **signal-front** 

    基于 Netty 实现简易版独立的 HttpServer，读
    同时引入 base 包提供作为 RPC Client 端的基础向下游发起调用
    ```
    /** 通过注解和继承 指定path和输入输出类型，扩展接口无需关注通信细节 */
    @HandlerTag(path = "/front") 
    public class TransHandler extends Handler<RequestInfo, ResponseInfo> {
        @Override
        public ResponseInfo handle(RequestInfo requestInfo) {
            ...
            return respInstance;
        }
    }
    ```
- signal-core 、 signal-route 、 signal-data

    引入 base 包、配置 config-rpc.properties 后通过 @RpcServiceTag 发布注册服务，使用 RpcClient.create(XService.class) 远程调用服务，同时可以在 Client 端自行扩展负载均衡策略
    
---

### 3. 应用示例

- **Step-1：** 没有扩展注册中心情况下，服务通信默认需要提前安装启动ZooKeeper；

- **Step-2：** 确认配置信息，如果ZK正常启动缺省值即可用；

config-rpc.properties
```
registration.address=host:port          # 注册中心 host:port
server.address=port                     # 当前服务占用的端口
server.basePackage=com.skr.xxx          # 递归扫描配置包下的服务提供者并注册服务

# 一些可选扩展配置

# 序列化：        自定义实现com.skr.signal.base.rpc.letter.serialize.SignalSerializable接口，显示配置后会替换，默认采用 ProtoStuff 
# serializer.impl=com.xxx    
           
# 负载均衡算法：   自定义实现com.skr.signal.base.registry.LoadPolicy接口，默认随机 
# loadPolicy.impl=com.xxx

# 服务发现/注册：  自定义实现com.skr.signal.base.registry.ServiceDiscover/ServiceRegistry接口，默认采用 Zookeeper 
# serviceDiscover.impl=com.xxx
# serviceRegistry.impl=com.xxx

...
```

- **Step-3：** 启动 DataMain、RouteMain、CoreMain、FrontMain；

- **Step-4：** 向前置（front）节点发送Http请求

```
curl  -X POST --data '{"traceId":"traceId","businessId":"businessId","requestKey":"requestKey"}' http://127.0.0.1:9001/front
```
Response：
```
{"answer":"[core][route][data]requestKey[data][route][core]"}
```

--- 

<br>
<div align=center><img src="https://github.com/BBLLMYD/netty-stroll/blob/master/other/img.png?raw=true" width="706" alt="应用模型示例" ></div>
<br>




