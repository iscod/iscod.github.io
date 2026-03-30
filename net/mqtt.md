# MQTT 协议：从窄带遥测到物联网消息

1999 年前后，在**带宽贵、链路不稳、设备省电**需求突出的场景（如通过卫星监控长输管道）下，**MQTT（Message Queuing Telemetry Transport）**被设计出来。它用**发布/订阅**替代典型 Web 的「一问一答」，在弱网与嵌入式环境里长期占据主流。

若把 HTTP 比作「每次带大量头信息的请求」，MQTT 更像「长连接上的轻量信令 + 按需推送」。下文按架构、报文、QoS、实战与高级特性展开。

---

## 一、核心架构：发布/订阅与 Broker

MQTT 不采用传统 **请求–响应** 为主模型，而是 **Publish/Subscribe**，中间经 **Broker（代理）** 转发，带来几类解耦：

1. **空间解耦**：发布者与订阅者不必知道对方地址，甚至不必感知对方是否存在。
2. **时间解耦**：是否能在订阅者离线时「攒消息」，取决于 **Broker 能力**、**会话是否持久**（如 `clean session` / MQTT 5 的会话过期间隔等），**并非**所有部署默认都会无限缓存。
3. **同步解耦**：发布调用通常不必阻塞等待对端业务处理完成（具体仍与 QoS、API 同步/异步有关），有利于节省终端 CPU 与功耗。

---

## 二、为何省带宽：二进制与短报文

早期卫星链路按比特计费，协议设计强调**紧凑**。

### 1. 固定报头：控制字节 + 剩余长度

- **第 1 字节**：报文类型（CONNECT、PUBLISH 等）及标志（QoS、Retain、DUP 等）。
- **剩余长度（Remaining Length）**：**变长编码**，占 **1～4 字节**，表示**当前报文**里固定头之后还有多少字节。

因此「固定报头」常见是 **2～5 字节**量级；只有在剩余长度单字节可表示时，才是**最少 2 字节**。若与「仅返回少量文本」的 HTTP 响应比，**头部开销通常小得多**，小包场景下的有效载荷占比往往明显更高。

### 2. 主题（Topic）层级与通配符

类似路径的层级字符串，便于按业务划分命名空间：

- **单层通配符 `+`**：匹配一层。例：`sensor/+/temperature` 匹配任意中间层房间下的 `temperature`。
- **多层通配符 `#`**：匹配当前层及以后所有层级，且**须出现在主题过滤器末尾**。例：`sensor/#` 匹配 `sensor` 下任意子主题。

> 通配符仅用于**订阅端**的主题过滤器；**发布**时主题名必须是具体字符串，不能带 `+` / `#`。

---

## 三、服务质量（QoS）

三种 QoS 在**发送端与 Broker、Broker 与订阅端**之间分别按约定执行；**端到端**是否「不丢不重」还依赖会话、持久化与业务幂等。

| 等级 | 语义（单跳常见理解） | 往返代价 | 典型用途 |
| --- | --- | --- | --- |
| **QoS 0** | 至多一次（Fire and Forget） | 低 | 可容忍偶发丢失的周期性遥测 |
| **QoS 1** | 至少一次（需 PUBACK 等确认） | 中 | **最常用**；可能重复，业务宜幂等 |
| **QoS 2** | 协议层通过四步握手保证**该跳不重复** | 高 | 开锁、关键指令等；成本高，需评估是否真有必要 |

---

## 四、报文结构概览

一条 MQTT 控制报文通常由三部分组成（视类型而定）：

1. **固定报头**：控制字节 + 剩余长度（必选）。
2. **可变报头**：如主题名、报文标识符等（按报文类型可选）。
3. **有效载荷（Payload）**：应用数据（部分报文可为空）。

![MQTT 固定报头示意](https://iscod.github.io/images/mqtt_1.png)

上图若展示 **PUBLISH** 且 QoS 为 1，则在发布方向会看到 **QoS 1** 标志；随后由接收方按规范回复 **PUBACK**（与图示中的确认帧对应），以体现「至少一次」语义。

![MQTT 消息负载示意](https://iscod.github.io/images/mqtt_2.png)

**Payload** 对内容为**透明字节流**：可用 JSON、CBOR、Protobuf 或自定义二进制；工业场景里常用紧凑二进制以进一步省流量与解析成本。

---

## 五、示例：用 Python 发布一条消息

以下使用 **paho-mqtt**。若你使用 **2.x**，需指定 `callback_api_version`；**1.x** 可去掉该参数。

```python
import paho.mqtt.client as mqtt

# paho-mqtt 2.x
client = mqtt.Client(
    mqtt.CallbackAPIVersion.VERSION2,
    client_id="Pipeline_Sensor_01",
)

# paho-mqtt 1.x 可写为：mqtt.Client(client_id="Pipeline_Sensor_01")

client.connect("broker.emqx.io", 1883, keepalive=60)
client.publish("pipeline/pressure", payload="52.5 PSI", qos=1)
print("已发布")
client.disconnect()
```

生产环境建议：**TLS（常见端口 8883）**、**认证（用户名密码或证书）**、**自有 Broker**，勿长期依赖公共测试 Broker。

---

## 六、遗嘱消息与保留消息

1. **遗嘱（Last Will and Testament, LWT）**  
   连接时在 Broker 登记「异常断开时代发的主题与载荷」。当连接**非干净断开**达到判定条件时，Broker 发布该消息，便于上位机快速感知离线（具体行为与 Keep Alive、Broker 实现有关）。

2. **保留消息（Retained）**  
   Broker 对每个主题可保留**最新一条**带 retain 标志的发布；新订阅者订阅后立即收到该保留消息，便于拿到「当前状态」而非等下一轮上报。

---

## 七、小结：适用边界

相对以文本头为主的短 HTTP 交互，MQTT 的常见优势包括：

1. **解析成本低**：二进制头 + 固定结构，适合 MCU。
2. **长连接推送**：由 Broker 向订阅方下发，适合指令与事件。
3. **头开销小**：在窄带、按量计费链路上更友好。

它特别适合 **嵌入式、弱网、移动网络、海量连接** 的消息场景；若已是 REST/浏览器同源生态、强依赖无状态短请求，则仍可能以 HTTP 为主。**是否选用 MQTT，应看连接模型、QoS、安全与运维栈是否匹配项目。**

---

## 参考（扩展阅读）

- [OASIS MQTT 标准](https://www.oasis-open.org/standard/mqtt/)
- [Eclipse Paho Python 客户端](https://eclipse.dev/paho/index.php?page=clients/python/index.php)
