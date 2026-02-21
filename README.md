# Smart Home Camera Solution

本项目记录了一套完整的家用摄像头方案，从摄像头接入、本地录像存储，到 Home Assistant 仪表盘展示的全流程搭建过程。适合从零开始搭建。

---

## 硬件清单

| 设备 | 型号 | 用途 |
|------|------|------|
| IP 摄像头 | 海康威视 DS-IPC-T12HV3-LA | 儿童房监控，RTSP 推流 |
| 智能摄像头 | 小米智能摄像机 4 4K 2 | 小米生态，通过 xiaomi_home 接入 HA |
| NAS | UGREEN NAS | 运行 Docker（HA + Frigate），存储录像 |

---

## 软件架构

```
海康威视摄像头 (RTSP)
       │
       ▼
  Frigate NVR  ──────────► NAS 录像存储 (/recordings)
  (Docker on NAS)
       │
       │ snapshot API
       ▼
  Home Assistant
  (Docker on NAS)
       │
       ▼
  Lovelace 仪表盘（单页房间布局）
```

完整架构图见 [docs/architecture.md](docs/architecture.md)

---

## 搭建步骤

### 1. 准备 NAS 目录

在 NAS 上创建以下目录结构（路径可自定义，需与 docker-compose.yml 一致）：

```
/volume1/homeRecording_KID_FIX/
├── frigate-config/     ← 存放 config.yml
└── recordings/         ← Frigate 写入录像（自动创建）
```

### 2. 部署 Frigate NVR

**2.1 填写配置文件**

复制 `frigate/config.yml` 到 NAS 的 `frigate-config/` 目录，替换以下占位符：

```yaml
# 替换为实际摄像头信息
- path: rtsp://<USERNAME>:<PASSWORD>@<CAMERA_IP>:554/Streaming/Channels/101
```

**2.2 启动容器**

将 `frigate/docker-compose.yml` 上传到 NAS，通过 Docker Manager 或 SSH 执行：

```bash
docker compose up -d
```

> **UGREEN NAS 提示**：可在 NAS 的 Docker Manager 图形界面中直接导入 docker-compose.yml。

**2.3 验证 Frigate 运行**

浏览器访问 `http://<NAS_IP>:5000`，应看到 Frigate Web UI 和摄像头画面。

**Frigate 关键 API**：

| API | 说明 |
|-----|------|
| `GET /api/<camera>/latest.jpg` | 最新快照 |
| `GET /api/events` | 检测事件列表 |
| `GET /api/stats` | 运行状态 |

---

### 3. 部署 Home Assistant

> 如果 HA 已在运行，跳过此步。

推荐使用 Docker 部署（与 Frigate 同一台 NAS）：

```bash
docker run -d \
  --name homeassistant \
  --restart unless-stopped \
  -v /volume1/ha-config:/config \
  -v /etc/localtime:/etc/localtime:ro \
  -p 8123:8123 \
  ghcr.io/home-assistant/home-assistant:stable
```

---

### 4. 在 HA 中添加 Frigate 摄像头实体

HA 通过 Frigate 的 snapshot API 显示摄像头画面，无需安装额外插件。

**4.1 在 `configuration.yaml` 中添加：**

```yaml
camera:
  - platform: generic
    name: "儿童房 Frigate"
    still_image_url: "http://<NAS_IP>:5000/api/kid_room/latest.jpg"
    # stream_source 可选，需 NAS 开放 8554 端口
    # stream_source: "rtsp://<NAS_IP>:8554/kid_room"
```

**4.2 重启 HA**，新实体 `camera.ertong_fang_frigate`（或自动生成的 ID）将出现在设备列表中。

---

### 5. 接入小米摄像头（可选）

1. HA 中安装 **xiaomi_home** 集成（Settings → Integrations → 搜索 Xiaomi Home）
2. 使用米家账号登录授权
3. 小米摄像机的实体会自动出现，包括：
   - `switch.xxx_motion_detection_xxx`：移动侦测开关（即"看家助手"）
   - `camera.xxx`：视频流

---

### 6. 配置 Lovelace 仪表盘

**6.1 创建新仪表盘**

HA → Settings → Dashboards → Add Dashboard

- Title: `智能家居`
- URL Path: `home`
- Icon: `mdi:home`

**6.2 应用配置**

进入仪表盘 → 右上角菜单 → Edit → 三点菜单 → Raw Configuration Editor

将 `homeassistant/lovelace-dashboard.yaml` 的内容粘贴进去，根据实际实体 ID 修改后保存。

---

## 功能对照表

| 功能 | 提供方 | 说明 |
|------|--------|------|
| 本地录像（存 NAS） | **Frigate** | 60天循环覆盖 |
| AI 目标检测（人/猫/狗） | **Frigate** | 本地离线运行，无需云端 |
| 快照 API | **Frigate** | `GET /api/<cam>/latest.jpg` |
| RTSP 双码流 | **海康威视自带** | 主流 101（录像）/ 子流 102（检测） |
| H.265 编码 | **海康威视自带** | 节省存储空间 |
| 摄像头本地 SD 卡录像 | **海康威视自带** | 独立于 Frigate，摄像头内置 |
| 移动侦测（看家助手） | **小米摄像头 + xiaomi_home** | HA 中通过 switch 实体控制 |
| Lovelace 仪表盘 | **Home Assistant** | 单页按房间分组 |

---

## 注意事项

- **凭证安全**：`config.yml` 中的摄像头用户名密码请勿提交到公开仓库。可使用环境变量或 HA Secrets 管理。
- **CPU 检测性能**：Frigate 默认使用 CPU 检测（`type: cpu`），延迟较高。有条件可接入 Google Coral USB Accelerator 或使用 OpenVINO。
- **存储空间**：60 天 `mode: all` 连续录像会占用大量空间，请根据实际磁盘容量调整 `days` 和 `mode`。
- **防火墙**：确保 NAS 的 5000、8554、8555 端口在局域网内可访问。

---

## 目录结构

```
.
├── README.md
├── docs/
│   └── architecture.md          # 完整系统架构图
├── frigate/
│   ├── docker-compose.yml       # Frigate 容器部署配置
│   └── config.yml               # Frigate 摄像头与录像配置
└── homeassistant/
    └── lovelace-dashboard.yaml  # HA 仪表盘配置模板
```
