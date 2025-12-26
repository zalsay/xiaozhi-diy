# 连接自定义 MCP 服务器计划

本文档详细说明如何将 Xiaozhi ESP32 设备连接到自定义的 MCP 服务器（而非官方服务器）。

## 1. 原理说明

Xiaozhi 设备启动后，会执行以下流程：
1. **OTA 版本检查**: 访问配置的 `OTA_URL`（默认为官方地址）。
2. **获取配置**: OTA 服务器返回 JSON 响应，其中包含 `mqtt` 或 `websocket` 的连接信息。
3. **建立连接**: 设备根据获取的配置信息，初始化对应的协议并连接到 MCP 服务器。
4. **握手**: 连接建立后，设备发送 `hello` 消息，声明支持 `mcp` 功能。

因此，连接自定义服务器的核心是**修改 OTA 检查的流程，让设备获取到指向自定义服务器的配置**。

## 2. 操作方案 A：修改 OTA URL (推荐)

最标准的方法是搭建一个简单的 HTTP 服务作为"伪 OTA 服务器"，并配置设备指向它。

### 步骤 1: 搭建配置服务器
你需要部署一个简单的 HTTP 接口（可以是静态 JSON 文件，也可以是简单的 Python/Node 服务），该接口需返回如下格式的 JSON：

**WebSocket 方式 (推荐):**
```json
{
  "firmware": {
    "version": "1.0.0",
    "url": ""
  },
  "websocket": {
    "url": "ws://192.168.1.100:8000/ws",
    "token": "your-auth-token-if-needed",
    "version": 1
  }
}
```

**MQTT 方式:**
```json
{
  "firmware": {
    "version": "1.0.0",
    "url": ""
  },
  "mqtt": {
    "endpoint": "192.168.1.100:1883",
    "client_id": "xiaozhi-dev",
    "username": "user",
    "password": "pass",
    "publish_topic": "xiaozhi/packet",
    "keepalive": 60
  }
}
```

### 步骤 2: 修改设备 OTA URL 配置
在编译固件前，通过 menuconfig 修改默认的 OTA URL。

1. 在终端运行配置命令：
   ```bash
   idf.py menuconfig
   ```
2. 导航至：
   `Xiaozhi Assistant` -> `Default OTA URL`
3. 将 URL 修改为你的配置服务器地址，例如：
   `http://192.168.1.100:8000/ota_check`
4. 保存并退出，重新编译烧录：
   ```bash
   idf.py build flash monitor
   ```

## 3. 操作方案 B：硬编码配置 (快速开发)

如果你不想搭建 HTTP 服务器，可以直接修改代码强制使用特定的连接配置。

**修改文件**: `main/ota.cc`

在 `Ota::CheckVersion` 函数中，解析 JSON 之前，手动注入配置。

**示例代码 (注入 WebSocket 配置):**
```cpp
// 在 main/ota.cc 的 CheckVersion 函数中找到 JSON 解析部分
// 或者直接在 Application::InitializeProtocol (main/application.cc) 中强行指定

// 推荐方法：修改 Application::InitializeProtocol
void Application::InitializeProtocol() {
    // 强制使用 Websocket 且硬编码配置
    Settings settings("websocket", true);
    settings.SetString("url", "ws://192.168.1.100:8000/ws");
    settings.SetString("token", "my-token");
    settings.SetInt("version", 1);
    
    // 确保 Ota 认为有 Websocket 配置 (或者直接修改 Application 逻辑忽略 ota check 结果)
    // ...
    
    protocol_ = std::make_unique<WebsocketProtocol>();
    // ...
}
```

**更简单的 Hack 方式**:
修改 `main/ota.cc`，在 `CheckVersion` 成功获取 HTTP 响应后，忽略真实响应，直接模拟解析一个硬编码的 JSON 字符串。

## 4. MCP 服务器端要求

自定义的 MCP 服务器需要处理设备的握手请求。

**设备发送的 Hello 消息示例:**
```json
{
    "type": "hello",
    "version": 1,
    "transport": "websocket", 
    "features": {
        "mcp": true,
        "aec": true
    },
    "audio_params": {
        "format": "opus",
        "sample_rate": 16000,
        "channels": 1,
        "frame_duration": 60
    }
}
```

**服务器应回复的 Hello 消息示例:**
```json
{
    "type": "hello",
    "transport": "websocket",
    "session_id": "sess-123456",
    "audio_params": {
        "sample_rate": 24000,
        "frame_duration": 60
    }
}
```

建立连接后，服务器即可通过 MCP 协议（JSON-RPC 2.0）控制设备。
