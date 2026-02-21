# 智能家居摄像头方案

基于 UGREEN NAS 的本地化家用摄像头监控系统，实现三路摄像头统一录像、AI 检测、Home Assistant 联动。

---

## 硬件清单

| 设备 | 型号 | 位置 | 接入方式 |
|------|------|------|---------|
| NAS | UGREEN DX4600（Intel N5105） | - | 运行所有 Docker 服务 |
| 摄像头 1 | 海康威视 DS-IPC-T12HV3-LA | 儿童房 | 原生 RTSP 直连 Frigate |
| 摄像头 2 | 小米智能摄像机 4 4K | 客厅 | miloco → micam → go2rtc → Frigate |
| 摄像头 3 | 小米智能摄像机 4 4K | 儿童房 | miloco → micam → go2rtc → Frigate |

---

## 系统架构

```
局域网
│
├── 海康威视（儿童房）
│   ├── 主流 RTSP :554/Channels/101 ────────────────┐
│   └── 子流 RTSP :554/Channels/102 ────────────────┤
│                                                    ▼
├── 小米摄像头（客厅）                           Frigate NVR
│   └── 小米私有协议                             (Docker, NAS)
│        └── miloco ──→ micam                        │
│                   ──→ go2rtc :8556 ────────────────┤
│                        xiaomi_living_room           │  OpenVINO GPU
│                                                    │  AI 检测
└── 小米摄像头（儿童房）                             │
    └── 小米私有协议                                  │  NAS 录像存储
         └── miloco ──→ micam_kid_room               │
                    ──→ go2rtc :8556 ────────────────┘
                         xiaomi_kid_room
                              │
                              ▼
                         MQTT (Mosquitto)
                              │
                              ▼
                       Home Assistant :8123
                              │
                              ▼
                       Lovelace Dashboard
```

完整说明见 [docs/architecture.md](docs/architecture.md)

---

## 录像规则（三台摄像头统一）

| 类型 | 说明 | 保留时长 |
|------|------|---------|
| 连续录像 | 24 小时不间断 | 30 天 |
| 警报片段 | 事件前 10s + 事件后 30s | 30 天 |
| 检测片段 | 事件前 5s + 事件后 15s | 30 天 |

**AI 检测目标**：人、猫、狗
**检测引擎**：OpenVINO，使用 N5105 iGPU（`device: GPU`）

---

## Docker 服务清单

| 容器 | 用途 | 端口 |
|------|------|------|
| `frigate` | NVR 主服务 | 5000（Web UI）、8554（RTSP） |
| `homeassistant` | 智能家居中枢 | 8123 |
| `mosquitto` | MQTT（需 network_mode: host） | 1883 |
| `miloco` | 小米协议中间件（需 network_mode: host） | 8001 |
| `go2rtc_xiaomi` | 小米 RTSP 服务器 | 8556、1985 |
| `micam` | 客厅小米主流推送 | - |
| `micam_kid_room` | 儿童房小米主流推送 | - |

---

## 部署步骤

### 1. 海康威视摄像头

直接在 `frigate/config.yml` 填写 RTSP 地址，无需额外配置。

### 2. 小米摄像头桥接

```bash
cd /volume2/homeRecording_KID_FIX/micam
sudo docker compose up -d
```

需先通过 miloco 完成小米账号认证（参见 [micam 项目](https://github.com/miiot/micam)）。

### 3. Frigate NVR

```bash
sudo docker restart frigate
# 验证：浏览器访问 http://<NAS_IP>:5000
```

### 4. Home Assistant 集成

- 安装 **Frigate** 集成（HACS）→ 填写 Frigate URL
- 安装 **Xiaomi Home** 集成 → 米家账号登录 → 启用摄像头设备
- 导入 `homeassistant/lovelace-dashboard.yaml` 到 Dashboard

---

## 注意事项

- **OpenVINO**：必须 `device: GPU`，不能用 `device: AUTO`（N5105 缺少 AVX-512）
- **Mosquitto**：必须 `network_mode: host`，否则 Docker 网络隔离导致连接失败
- **HA 启动顺序**：HA 重启后等 MQTT 集成加载完再重启 Frigate，否则 `KeyError: mqtt`
- **小米子流**：miloco 当前只支持 channel 0，单路流同时用于录像和检测
- **凭证安全**：密码不提交仓库，使用本地 `.secrets` 文件管理

---

## 目录结构

```
.
├── README.md
├── docs/
│   └── architecture.md          # 详细架构说明
├── frigate/
│   └── config.yml               # Frigate 配置（脱敏）
├── micam/
│   ├── docker-compose.yml       # 小米摄像头桥接服务
│   └── go2rtc.yaml              # go2rtc 流配置
└── homeassistant/
    └── lovelace-dashboard.yaml  # HA Dashboard 配置
```
