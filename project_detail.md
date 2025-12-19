# xiaozhi-esp32 项目详解

## 1. 项目概览

xiaozhi-esp32 是一个基于 ESP32 系列芯片的开源语音交互机器人项目，通过流式 ASR + 大语言模型（LLM）+ TTS 的架构，实现“会说话、会看、能控制设备”的 AI 聊天机器人。

项目核心特点：

- 支持 Qwen、DeepSeek 等多种大模型，通过官方云端服务或自建服务器接入
- 采用 MCP（Model Context Protocol）作为统一协议，同时支持“设备端 MCP” 与 “云端 MCP”
- 支持 Wi‑Fi 与 ML307 Cat.1 4G 多种联网方式
- 提供离线唤醒词、语音识别、声纹识别、多语言 TTS 等完整语音链路
- 适配 70+ 款开源/商业硬件开发板，覆盖 ESP32‑C3 / ESP32‑S3 / ESP32‑P4 等平台

当前仓库为 v2 版本，分区表与 v1 不兼容。如需使用 v1 固件，可切换到 `v1` 分支（维护至 2026 年 2 月）。

## 2. 已实现功能概览

根据 `README_zh.md` 与相关文档，项目当前已实现的能力包括：

- 网络
  - Wi‑Fi 连接
  - ML307 Cat.1 4G 模块
- 语音交互
  - ESP-SR 离线语音唤醒
  - 流式 ASR（语音转文字）
  - LLM 对话（Qwen / DeepSeek 等）
  - TTS 语音合成
  - 3D Speaker 声纹识别，识别当前说话人
- 显示与交互
  - OLED / LCD 显示屏，支持表情与多语言 UI
  - 电量显示与电源管理
  - 多种按键、触摸键、传感器输入
- 多语言支持
  - 至少支持中文、英文、日文，更多语言通过资源文件扩展
- 硬件平台
  - ESP32‑C3 / ESP32‑S3 / ESP32‑P4 多平台支持
  - 通过板级抽象层（Board）适配众多开发板
- MCP 能力
  - 设备端 MCP：控制音量、灯光、电机、GPIO 等 IoT 外设
  - 云端 MCP：扩展智能家居控制、PC 桌面操作、知识搜索、邮件收发等能力
- 自定义资源
  - 自定义唤醒词、字体、表情、聊天背景
  - 提供在线“自定义 Assets 生成器”工具仓库 `xiaozhi-assets-generator`

## 3. 仓库结构与核心模块

顶层目录结构（部分）：

- `main/`
  - ESP-IDF 应用主体代码
  - `main.cc`：入口函数，负责 NVS 初始化并启动 `Application`
  - `application.cc` / `application.h`：应用主状态机、事件循环和业务逻辑
  - `assets/`：
    - `common/`：通用提示音（如低电量、成功、弹出等）
    - `locales/`：多语言语音资源，每个语言一个目录，包含数字朗读、错误提示、欢迎词等
  - `boards/`：
    - 针对不同硬件开发板的适配目录（如 `labplus-mpython-v3/`、`waveshare-s3-epaper-1.54/` 等）
    - 每个板子通常包含：
      - `README.md` 或 `README_en.md`：编译、烧录和硬件说明
      - 可选的 `config.json`：一键构建脚本使用的配置
      - 板级初始化代码、LCD/音频驱动配置等
  - `Kconfig.projbuild`：
    - 定义项目级配置项，例如：
      - OTA 固件检查地址 `OTA_URL`
      - 是否烧录默认或自定义 Assets
      - 设备默认语言
      - MCP / 网络相关开关等
- `docs/`
  - 项目配套文档：
    - `custom-board.md`：自定义开发板指南，介绍如何编写板级代码并集成到项目
    - `mcp-usage.md`：使用 MCP 协议控制 IoT 设备的方式
    - `mcp-protocol.md`：MCP 协议交互流程与设备端实现
    - `mqtt-udp.md`：MQTT + UDP 混合通信协议说明
    - `websocket.md`：详细的 WebSocket 通信协议文档
  - `v0/`、`v1/`：不同版本硬件接线图和效果图
- 根目录
  - `README.md` / `README_zh.md` / `README_ja.md`：多语言项目总览
  - `CMakeLists.txt`：ESP-IDF 项目入口 CMake 配置
  - `.github/workflows/`：CI 构建配置

## 4. 构建与烧录方式

项目基于 ESP-IDF（推荐 5.4 及以上）进行开发，支持两类常见构建方式：

### 4.1 通用构建流程

以 ESP32‑S3 开发板为例：

1. 设置目标芯片

   ```bash
   idf.py set-target esp32s3
   ```

2. 进入配置界面并选择开发板：

   ```bash
   idf.py menuconfig
   ```

   在菜单中：

   ```
   Xiaozhi Assistant -> Board Type -> 选择具体板型
   ```

3. 按需调整 Flash、分区表、PSRAM 等配置（部分板卡在 README 中给出推荐值）。

4. 编译工程：

   ```bash
   idf.py build
   ```

5. 烧录并查看日志：

   ```bash
   idf.py build flash monitor
   ```

6. 某些板卡需要通过 `idf.py merge-bin` 或 `esptool.py` 生成/烧录特定地址的固件，详见对应 `main/boards/<board>/README*.md`。

### 4.2 一键构建脚本

部分板卡提供脚本化构建，例如：

- `main/boards/aipi-lite/README_en.md`

  ```bash
  python scripts/release.py aipi-lite -c config_en.json
  ```

- `main/boards/sensecap-watcher/README_en.md`

  ```bash
  python scripts/release.py sensecap-watcher -c config_en.json
  ```

脚本会自动设置目标、应用特定的 `sdkconfig_append`，并完成编译与打包。

## 5. MCP 协议与通讯架构

xiaozhi-esp32 通过 MCP 协议将本地设备与云端大模型连接在一起：

- 设备端 MCP
  - 运行在 ESP32 设备上，将按钮、传感器、显示、音响等硬件抽象为 MCP 工具
  - 大模型通过调用 MCP 工具控制设备，例如：
    - 播放提示音
    - 控制 RGB 灯颜色
    - 驱动电机或舵机
    - 读写 GPIO 状态

- 云端 MCP
  - 部署在服务器端，扩展大模型能力，如：
    - 智能家居控制
    - 桌面操作
    - 数据搜索与知识库问答
    - 邮件收发等

设备与云端之间可通过 WebSocket 或 MQTT+UDP 建立流式连接，实现音频和指令的实时收发，具体协议细节可参阅：

- `docs/websocket.md`
- `docs/mqtt-udp.md`
- `docs/mcp-usage.md`
- `docs/mcp-protocol.md`

## 6. 多硬件开发板支持

`main/boards/` 目录下为各类开发板的适配实现，每个子目录通常包含：

- 产品说明链接
- 编译和烧录步骤（`idf.py` 命令、分区表配置、Flash 大小等）
- 特殊外设（摄像头、麦克风、EPaper、AMOLED 等）的初始化代码
- 板载按键、触摸、传感器、RGB 灯等的管脚定义

典型示例：

- `main/boards/labplus-mpython-v3/`：掌控板 V3，包含串口、LCD、ES8388 音频、RGB 等资源说明
- `main/boards/waveshare-s3-epaper-1.54/`：微雪电子电子纸开发板，提供完整的构建和烧录命令
- `main/boards/movecall-cuican-esp32s3/`：璀璨·AI 吊坠，针对特定 Flash 分区与优化选项进行了调整

如需添加自定义板卡，可参考 `docs/custom-board.md` 中的流程：

1. 在 `main/boards/` 下创建新目录
2. 实现板级类，继承 `WifiBoard` 或 `Ml307Board`
3. 配置 `Kconfig.projbuild` 以加入新板卡选项
4. 使用 `DECLARE_BOARD` 宏注册板卡

## 7. 开发环境与代码风格

官方建议的开发环境：

- 编辑器：Cursor 或 VSCode
- SDK：ESP-IDF 5.4 及以上版本
- 操作系统：推荐 Linux（编译更快、驱动问题更少）

代码风格：

- C++ 代码遵循 Google C++ Style
- 建议在提交代码前使用 clang-format 等工具进行格式化

## 8. 与大模型及服务器配合

xiaozhi-esp32 默认连接官方 `xiaozhi.me` 服务器，个人用户可以在控制台配置所使用的大模型（如 Qwen 实时模型）。

如果希望在本地或自建云端部署服务器，可以参考以下相关项目：

- `xinnan-tech/xiaozhi-esp32-server`：Python 服务器实现
- `joey-zhou/xiaozhi-esp32-server-java`：Java 服务器实现
- `AnimeAIChat/xiaozhi-server-go`：Golang 服务器实现

## 9. 许可证与社区

- 本项目以 MIT 协议开源，可自由用于学习、修改及商业用途
- 欢迎通过 GitHub Issues 反馈问题或提建议
- 官方 QQ 交流群：1011329060

通过阅读本项目和配套文档，你可以：

- 快速上手一个完整的语音交互硬件项目
- 学习如何将 LLM 能力通过 MCP 协议扩展到 IoT 设备
- 为自己或客户定制一款“会说话的 AI 设备”（例如 AI 女友、语音助手、机器人等）

