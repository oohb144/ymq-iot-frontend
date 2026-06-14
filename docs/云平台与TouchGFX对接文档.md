# 智能羽毛球陪练系统 - 云平台与TouchGFX对接文档

## 一、云平台架构概览

```
┌─────────────────────────────────────────────────┐
│              云平台 (Web浏览器)                   │
│  ┌───────────────────────────────────────────┐  │
│  │  cloud_platform.html (单页应用)            │  │
│  │  - 登录页 / 控制台 / 设备管理 / 训练中心   │  │
│  │  - 球场定位 / 数据分析 / 告警日志          │  │
│  │  - 用户管理 / 系统设置                     │  │
│  └───────────────────┬───────────────────────┘  │
└──────────────────────┼──────────────────────────┘
                       │ MQTT / HTTP
            ┌──────────┴──────────┐
            │   MQTT Broker       │
            │   (如Mosquitto)     │
            └──────────┬──────────┘
                       │
       ┌───────────────┼───────────────┐
       │               │               │
  ┌────┴────┐    ┌─────┴────┐    ┌─────┴────┐
  │ STM32   │    │  STM32   │    │  STM32   │
  │ 设备#001│    │  设备#002 │    │  设备#003 │
  │TouchGFX │    │ TouchGFX │    │ TouchGFX │
  │ 800×480 │    │ 800×480  │    │ 800×480  │
  └─────────┘    └──────────┘    └──────────┘
```

---

## 二、云平台功能模块 (8个页面)

| 页面 | 功能 | 核心组件 |
|------|------|---------|
| 登录页 | 邮箱/密码登录 + 第三方OAuth | 登录表单、品牌展示 |
| 控制台 | 全局数据概览 | 5个KPI卡片、趋势折线图、模式饼图、环境数据、电机状态、告警列表 |
| 设备管理 | 设备列表与日志 | 3个设备卡片、运行日志表格、远程控制按钮 |
| 训练中心 | 实时训练监控 | 模式选择(3种)、参数滑块、开始/暂停/重置、实时数据曲线 |
| 球场定位 | 球场可视化 | 球场俯视图(含设备/球员/落点)、落点命中率统计、运动轨迹数据 |
| 数据分析 | 历史数据图表 | 4个汇总KPI、命中率趋势图、速度分布柱状图、训练历史表格 |
| 告警日志 | 异常管理 | 3个级别KPI、告警列表表格(严重/警告/信息) |
| 用户管理 | 用户CRUD | 用户列表表格(头像/等级/训练数据) |
| 系统设置 | 全局配置 | 默认参数滑块、功能开关、MQTT配置、系统信息 |

---

## 三、MQTT通信协议

### 设备 → 云平台 (上报数据)

| Topic | 数据格式 | 频率 | 说明 |
|-------|---------|------|------|
| `shuttlebot/{device_id}/status` | JSON | 5秒 | 设备在线状态 |
| `shuttlebot/{device_id}/env` | JSON | 10秒 | 环境数据(风速/温度/湿度/气压) |
| `shuttlebot/{device_id}/motor` | JSON | 5秒 | 电机状态(RPM/速度/运行时长/异常数) |
| `shuttlebot/{device_id}/training` | JSON | 训练时1秒 | 训练实时数据(模式/发球数/命中率) |
| `shuttlebot/{device_id}/position` | JSON | 训练时2秒 | 球场定位坐标 |
| `shuttlebot/{device_id}/alert` | JSON | 事件触发 | 告警信息 |

### 云平台 → 设备 (下发控制)

| Topic | 数据格式 | 说明 |
|-------|---------|------|
| `shuttlebot/{device_id}/cmd/mode` | JSON | 切换训练模式 |
| `shuttlebot/{device_id}/cmd/start` | JSON | 开始训练 |
| `shuttlebot/{device_id}/cmd/stop` | JSON | 停止训练 |
| `shuttlebot/{device_id}/cmd/params` | JSON | 更新参数(速度/间隔/时长) |
| `shuttlebot/{device_id}/cmd/config` | JSON | 系统配置更新 |

### JSON数据示例

```json
// 环境数据上报
{
  "ts": 1718264535,
  "wind_speed": 2.3,
  "temperature": 26.5,
  "humidity": 58,
  "pressure": 1013
}

// 电机状态上报
{
  "ts": 1718264535,
  "rpm": 2450,
  "ball_speed": 85,
  "runtime_seconds": 5025,
  "error_count": 0,
  "motor_temp": 45.2
}

// 训练数据上报
{
  "ts": 1718264535,
  "mode": "fixed_point",
  "serving_count": 156,
  "accuracy": 78.5,
  "duration_seconds": 2700
}

// 切换模式命令
{
  "cmd": "set_mode",
  "mode": "fixed_point",
  "params": {
    "speed": 85,
    "interval": 2.0,
    "duration": 45,
    "zone": "A"
  }
}
```

---

## 四、TouchGFX 设备端对接

### 4.1 WiFi连接模块

在TouchGFX的Model层中添加WiFi管理：

```cpp
// WiFiManager.hpp
class WiFiManager {
public:
    bool connect(const char* ssid, const char* password);
    bool isConnected();
    int getSignalStrength(); // RSSI
    const char* getIPAddress();
};
```

### 4.2 MQTT客户端模块

```cpp
// MQTTClient.hpp
class MQTTClient {
public:
    bool connect(const char* broker, int port, const char* deviceId);
    void publish(const char* topic, const char* jsonPayload);
    void subscribe(const char* topic, MessageCallback callback);
    void loop(); // 在FreeRTOS任务中周期调用
};
```

### 4.3 数据上报定时任务

```cpp
// 在FreeRTOS中创建定时任务
void sensorReportTask(void* param) {
    while(1) {
        // 读取传感器
        float windSpeed = readWindSensor();
        float temperature = readTempSensor();
        int humidity = readHumiditySensor();

        // 构建JSON
        char json[256];
        snprintf(json, sizeof(json),
            "{\"ts\":%lu,\"wind_speed\":%.1f,\"temperature\":%.1f,\"humidity\":%d}",
            getCurrentTimestamp(), windSpeed, temperature, humidity);

        // 发布到MQTT
        mqtt.publish("shuttlebot/SB2024001/env", json);

        vTaskDelay(pdMS_TO_TICKS(10000)); // 10秒上报一次
    }
}
```

### 4.4 云平台命令接收

```cpp
void onCommandReceived(const char* topic, const char* payload) {
    // 解析JSON命令
    JsonDocument doc;
    deserializeJson(doc, payload);

    if (strcmp(doc["cmd"], "set_mode") == 0) {
        const char* mode = doc["mode"];
        // 切换TouchGFX界面
        if (strcmp(mode, "fixed_point") == 0) {
            application().gotoTrainingModeScreen();
        } else if (strcmp(mode, "self_training") == 0) {
            application().gotoSelfTrainingScreen();
        }
    }
    else if (strcmp(doc["cmd"], "start") == 0) {
        startTraining(); // 启动电机
    }
    else if (strcmp(doc["cmd"], "stop") == 0) {
        stopTraining(); // 停止电机
    }
}
```

### 4.5 TouchGFX Model层集成

```cpp
// Model.hpp - 扩展数据模型
class Model {
public:
    // 传感器数据
    float windSpeed;
    float temperature;
    int humidity;
    int pressure;

    // 电机数据
    int motorRPM;
    int ballSpeed;
    int errorCount;

    // 训练数据
    const char* currentMode;
    int servingCount;
    float accuracy;

    // 云平台状态
    bool cloudConnected;
    int lastSyncTimestamp;

    // 方法
    void reportToCloud();
    void receiveFromCloud(const char* json);
};
```

---

## 五、数据同步策略

| 数据类型 | 设备→云 | 云→设备 | 同步方式 |
|---------|---------|---------|---------|
| 环境数据 | 每10秒 | - | 单向推送 |
| 电机状态 | 每5秒 | - | 单向推送 |
| 训练数据 | 训练时每秒 | - | 单向推送 |
| 球场定位 | 训练时每2秒 | - | 单向推送 |
| 训练模式 | - | 命令下发 | 双向同步 |
| 训练参数 | - | 命令下发 | 双向同步 |
| 开始/停止 | 状态上报 | 命令下发 | 双向同步 |
| 系统配置 | 确认回复 | 命令下发 | 请求-响应 |
| 告警事件 | 即时上报 | - | 单向推送 |

---

## 六、云平台文件结构

```
D:\student_code\esp\YMQ\
├── cloud_platform.html          # 云平台Web界面(已完成)
├── cloud_dashboard_concept.png  # 控制台概念效果图
├── cloud_login_concept.png      # 登录页概念效果图
└── docs/
    └── 云平台与TouchGFX对接文档.md  # 本文档
```

### 后续部署建议

**方案A: 本地部署 (推荐初期)**
- 云平台HTML直接放在ESP32的SPIFFS/LittleFS中
- 通过ESP32内置Web Server提供访问
- MQTT Broker也运行在ESP32上 (如使用ESP32作为Gateway)
- 访问地址: `http://192.168.x.x`

**方案B: 云服务器部署**
- 云平台HTML部署到云服务器 (如Vercel/Nginx)
- 独立的MQTT Broker (如EMQX/Mosquitto)
- ESP32/STM32设备通过WiFi连接MQTT Broker
- 支持多设备、多用户远程管理

---

## 七、新增建议功能

基于云平台能力，建议在后续版本中加入：

1. **训练回放** - 记录每次训练的球场轨迹动画，支持云平台回放
2. **AI教练建议** - 基于训练数据自动生成改进建议
3. **多设备对比** - 云平台上并排对比多台设备的训练数据
4. **训练计划** - 在云平台创建训练计划，下发到设备自动执行
5. **排行榜** - 多用户训练成绩排名
6. **OTA固件更新** - 通过云平台远程推送固件更新到STM32设备
