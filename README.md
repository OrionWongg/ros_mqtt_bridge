# ROS2-MQTT Bridge Node

一个功能强大的ROS2与MQTT双向桥接工具，支持多话题同时桥接、图像传输、频率控制等高级特性。

## 📋 目录

- [功能特性](#功能特性)
- [系统架构](#系统架构)
- [快速开始](#快速开始)
- [配置说明](#配置说明)
- [使用示例](#使用示例)
- [工具集](#工具集)
- [常见问题](#常见问题)
- [开发指南](#开发指南)

## ✨ 功能特性

### 核心功能
- ✅ **双向桥接**：支持ROS→MQTT和MQTT→ROS双向消息转发
- ✅ **多话题并发**：同时桥接多个ROS话题到MQTT
- ✅ **动态消息类型**：自动解析和转换各种ROS消息类型
- ✅ **图像传输**：支持压缩图像的Base64编码传输
- ✅ **频率控制**：可配置消息发送频率，节省带宽
- ✅ **连接管理**：自动重连、心跳检测、连接状态监控

### 高级特性
- 📊 **统计信息**：实时统计消息数量、速率、数据量
- 🔄 **灵活配置**：YAML配置文件，支持热加载
- 🎯 **字段提取**：支持提取消息的特定字段或多字段组合
- 🔐 **安全认证**：支持MQTT用户名/密码认证
- 📈 **QoS控制**：支持MQTT三种QoS等级
- 🛠️ **工具完善**：提供监控、传输、验证等实用工具

## 🏗️ 系统架构

```
┌─────────────────────────────────────────────────────────┐
│                   ROS2 Environment                       │
├─────────────────────────────────────────────────────────┤
│  /image_raw_0  /gps/fix  /detections_0  /vlm_output    │
│       ↓           ↓            ↓            ↓           │
│  ┌──────────────────────────────────────────────────┐  │
│  │         Multi Bridge Manager Node                │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐        │  │
│  │  │ Bridge 1 │ │ Bridge 2 │ │ Bridge N │ ...    │  │
│  │  └──────────┘ └──────────┘ └──────────┘        │  │
│  │        ↓            ↓            ↓               │  │
│  │      MQTT Interface (paho-mqtt)                 │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                         ↓ ↑
              ┌──────────────────────┐
              │   MQTT Broker        │
              │  (120.24.79.108)     │
              └──────────────────────┘
                         ↓ ↑
┌─────────────────────────────────────────────────────────┐
│                  Remote Clients                          │
│  (Web Dashboard / Mobile App / Other ROS Systems)       │
└─────────────────────────────────────────────────────────┘
```

### 消息流转示例

```
ROS话题: /image_raw_0/compressed (30Hz)
    ↓
[频率限制: 5秒]
    ↓
[数据提取: data字段]
    ↓
[Base64编码]
    ↓
[JSON封装: 添加元数据]
    ↓
MQTT发布: ros2/image_compressed_0/data (0.2Hz)
    ↓
远程客户端接收并显示
```

## 🚀 快速开始

### 环境要求

- **ROS2**: Humble/Foxy/Galactic
- **Python**: 3.8+
- **操作系统**: Ubuntu 20.04/22.04

### 安装依赖

```bash
# ROS2包依赖（如果需要）
sudo apt install ros-${ROS_DISTRO}-sensor-msgs ros-${ROS_DISTRO}-std-msgs

# Python依赖
pip3 install paho-mqtt PyYAML
```

### 编译安装

```bash
# 进入工作空间
cd ~/ros2_ws

# 编译包
colcon build --packages-select ros_mqtt_bridge_node

# 加载环境
source install/setup.bash
```

### 启动桥接

```bash
# 使用launch文件启动（推荐）
ros2 launch ros_mqtt_bridge_node multi_bridge_manager.launch.py

# 或直接运行节点
ros2 run ros_mqtt_bridge_node multi_bridge_manager
```

### 验证运行

```bash
# 查看节点状态
ros2 node list | grep mqtt

# 查看统计信息
ros2 topic echo /ros_mqtt_bridge/statistics

# 查看日志
ros2 node info /multi_bridge_manager
```

## ⚙️ 配置说明

### 配置文件结构

主配置文件：`config/multi_bridge_config.yaml`

```yaml
# 全局MQTT配置
mqtt_global:
  broker_host: "120.24.79.108"    # MQTT服务器地址
  broker_port: 1883                # MQTT服务器端口
  topic_prefix: "ros2"             # MQTT主题前缀
  username: "zone"                 # MQTT用户名
  password: "NeverGiveUp"          # MQTT密码
  keepalive: 30                    # 心跳间隔（秒）
  qos: 1                           # 默认QoS等级

# 全局桥接配置
bridge_global:
  enable_statistics: true          # 启用统计信息
  enable_heartbeat: true           # 启用心跳
  stats_publish_interval: 30.0     # 统计发布间隔（秒）
  heartbeat_interval: 60.0         # 心跳间隔（秒）

# 桥接配置列表
bridges:
  - name: "Image_Bridge_0"
    description: "图像桥接"
    enabled: true
    ros_config:
      topic: "/image_raw_0/compressed"
      message_type: "sensor_msgs/CompressedImage"
      data_field: "data"
      queue_size: 5
      publish_interval: 5.0        # 发送间隔（秒）
    mqtt_config:
      topic_name: "image_compressed_0"
      topic_suffix: "data"
      qos: 1
      retain: false
    metadata:
      source_node: "car_1"
      frame_id: "camera_frame"
```

### 配置项详解

#### 1. MQTT全局配置 (`mqtt_global`)

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `broker_host` | string | - | MQTT服务器地址 |
| `broker_port` | int | 1883 | MQTT服务器端口 |
| `topic_prefix` | string | "ros2" | MQTT主题统一前缀 |
| `client_id_prefix` | string | "ros_mqtt_bridge" | 客户端ID前缀 |
| `username` | string | - | MQTT认证用户名 |
| `password` | string | - | MQTT认证密码 |
| `keepalive` | int | 30 | 心跳间隔（秒） |
| `qos` | int | 1 | 默认QoS等级（0/1/2） |

#### 2. 桥接全局配置 (`bridge_global`)

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `enable_statistics` | bool | true | 是否发布统计信息 |
| `enable_heartbeat` | bool | true | 是否发布心跳 |
| `message_history_size` | int | 50 | 消息历史记录大小 |
| `connection_check_interval` | float | 5.0 | 连接检查间隔（秒） |
| `stats_publish_interval` | float | 30.0 | 统计发布间隔（秒） |
| `heartbeat_interval` | float | 60.0 | 心跳发布间隔（秒） |

#### 3. 单个桥接配置 (`bridges[]`)

##### 基本配置

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `name` | string | ✅ | 桥接器唯一名称 |
| `description` | string | ❌ | 桥接器描述 |
| `enabled` | bool | ✅ | 是否启用 |
| `type` | string | ❌ | 桥接类型（ros_to_mqtt/mqtt_to_ros）|

##### ROS配置 (`ros_config`)

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `topic` | string | ✅ | ROS话题名称 |
| `message_type` | string | ✅ | ROS消息类型（如sensor_msgs/Image） |
| `data_field` | string | ✅ | 提取的数据字段 |
| `queue_size` | int | ❌ | 订阅队列大小（默认10） |
| `publish_interval` | float | ❌ | 发送间隔限制（秒），用于频率控制 |

**data_field 支持的格式：**
- 单字段：`"data"` - 提取msg.data
- 嵌套字段：`"pose.position.x"` - 提取msg.pose.position.x
- 多字段：`"latitude,longitude"` - 提取多个字段组成字典

##### MQTT配置 (`mqtt_config`)

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `topic_name` | string | ✅ | MQTT主题名称 |
| `topic_suffix` | string | ✅ | MQTT主题后缀 |
| `qos` | int | ❌ | QoS等级（0/1/2，默认1） |
| `retain` | bool | ❌ | 是否保留消息（默认false） |

**最终MQTT主题格式：** `{topic_prefix}/{topic_name}/{topic_suffix}`

示例：`ros2/image_compressed_0/data`

##### 元数据 (`metadata`)

| 参数 | 类型 | 说明 |
|------|------|------|
| `source_node` | string | 源节点标识（用于标识消息来源） |
| `frame_id` | string | 坐标系ID（可选） |

### 配置示例

#### 示例1：GPS数据桥接

```yaml
- name: "GPS_bridge"
  enabled: true
  ros_config:
    topic: "/gps/fix"
    message_type: "sensor_msgs/NavSatFix"
    data_field: "latitude,longitude"  # 只传输经纬度
    queue_size: 10
  mqtt_config:
    topic_name: "gps"
    topic_suffix: "fix"
    qos: 1
    retain: false
  metadata:
    source_node: "car_1"
```

**结果：**
- ROS话题：`/gps/fix`
- MQTT话题：`ros2/gps/fix`
- 数据格式：`{"latitude": 22.5, "longitude": 114.0, ...}`

#### 示例2：压缩图像桥接（带频率控制）

```yaml
- name: "Image_Bridge_0"
  enabled: true
  ros_config:
    topic: "/image_raw_0/compressed"
    message_type: "sensor_msgs/CompressedImage"
    data_field: "data"
    queue_size: 5
    publish_interval: 5.0  # 5秒发送一次
  mqtt_config:
    topic_name: "image_compressed_0"
    topic_suffix: "data"
    qos: 1
    retain: false
  metadata:
    source_node: "car_1"
```

**效果：**
- ROS发布频率：30Hz
- MQTT发送频率：0.2Hz（每5秒）
- 带宽节省：99.3%

#### 示例3：YOLO检测结果桥接

```yaml
- name: "YOLO_detection_bridge_0"
  enabled: true
  ros_config:
    topic: "/detections_0"
    message_type: "yolo_msgs/DetectionArray"
    data_field: "detections"
    queue_size: 5
  mqtt_config:
    topic_name: "yolo"
    topic_suffix: "detections_0"
    qos: 1
    retain: false
  metadata:
    source_node: "car_1"
```

#### 示例4：MQTT到ROS（事件触发）

```yaml
- name: "event_trigger_bridge"
  enabled: true
  type: mqtt_to_ros  # 反向桥接
  ros_config:
    topic: "/event_trigger"
    message_type: "std_msgs/String"
    data_field: "data"
    queue_size: 10
  mqtt_config:
    topic_name: "event_trigger"
    topic_suffix: "data"
    qos: 0
    retain: false
  metadata:
    source_node: "remote_controller"
```

## 📚 使用示例

### 1. 基本使用流程

```bash
# 1. 修改配置文件
nano ~/ros2_ws/src/ros_mqtt_bridge_node/config/multi_bridge_config.yaml

# 2. 编译（如果修改了代码）
cd ~/ros2_ws
colcon build --packages-select ros_mqtt_bridge_node

# 3. 加载环境
source install/setup.bash

# 4. 启动桥接
ros2 run ros_mqtt_bridge_node multi_bridge_manager




## 🔧 开发指南

### 添加新的桥接

1. **修改配置文件**

```yaml
bridges:
  - name: "My_New_Bridge"
    enabled: true
    ros_config:
      topic: "/my_topic"
      message_type: "std_msgs/String"
      data_field: "data"
      queue_size: 10
    mqtt_config:
      topic_name: "my_topic"
      topic_suffix: "data"
      qos: 1
      retain: false
    metadata:
      source_node: "my_node"
```

2. **重新加载配置**

```bash
# 重启节点即可，无需重新编译
ros2 launch ros_mqtt_bridge_node multi_bridge_manager.launch.py
```

### 支持新的消息类型

桥接器会自动解析ROS消息类型，无需额外配置。确保：

1. 消息类型包已安装
2. `message_type` 格式正确（`package_name/MessageType`）
3. `data_field` 对应消息中的实际字段

### 自定义消息处理

如需特殊处理，修改 `topic_bridge.py`：

```python
def _process_field_data(self, data, field_name: str, msg=None) -> Any:
    """处理字段数据"""
    # 添加自定义处理逻辑
    if field_name == "my_special_field":
        return custom_process(data)
    
    # 默认处理
    return data
```

### 调试技巧

```bash
# 1. 启用详细日志
ros2 run ros_mqtt_bridge_node multi_bridge_manager --ros-args --log-level debug

# 2. 监控ROS话题
ros2 topic echo /your_topic

# 3. 使用MQTT客户端测试
mosquitto_sub -h 120.24.79.108 -t "ros2/#" -u zone -P NeverGiveUp -v

# 4. 查看节点信息
ros2 node info /multi_bridge_manager
```

## 📦 项目结构

```
ros_mqtt_bridge_node/
├── config/
│   ├── multi_bridge_config.yaml      # 主配置文件
│   ├── zip_sender_config.yaml        # ZIP发送配置
│   └── zip_receiver_config.yaml      # ZIP接收配置
├── launch/
│   ├── multi_bridge_manager.launch.py  # 主启动文件
│   └── zip_transfer.launch.py          # 文件传输启动文件
├── ros_mqtt_bridge_node/
│   ├── __init__.py
│   ├── config_loader.py              # 配置加载器
│   ├── mqtt_interface.py             # MQTT接口
│   ├── topic_bridge.py               # 话题桥接器
│   ├── multi_bridge_manager.py       # 桥接管理器
│   ├── file_transfer_util.py         # 文件传输工具
│   ├── zip_sender.py                 # ZIP发送节点
│   └── zip_receiver.py               # ZIP接收节点
├── package.xml
├── setup.py
├── setup.cfg
└── README.md                         # 本文件
```


## 🤝 贡献指南

欢迎提交Issue和Pull Request！

### 开发环境搭建

```bash
git clone https://github.com/your-repo/ros_mqtt_bridge_node.git
cd ros_mqtt_bridge_node
# 按照快速开始部分进行编译
```

### 提交规范

- 遵循ROS2代码规范
- 添加适当的注释和文档
- 确保所有测试通过

## 📄 许可证

本项目采用 [MIT License](LICENSE)

## 👥 作者

- 开发者：OrionWongg,sc,Yesord


## 📮 联系方式

如有问题或建议，请提交Issue或联系开发团队。

---

**最后更新：** 2025-11-25
