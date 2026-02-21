# 系统架构详解

## 摄像头接入对比

| | 海康威视（儿童房） | 小米 4K（客厅） | 小米 4K（儿童房） |
|---|---|---|---|
| 协议 | 标准 RTSP | 小米私有协议 | 小米私有协议 |
| 流数量 | 主流 + 子流（双码流） | 单路流（channel 0） | 单路流（channel 0） |
| Frigate 摄像头名 | `kid_room` | `living_room` | `kid_room_xiaomi` |
| 录像流 | Channels/101（高清） | xiaomi_living_room | xiaomi_kid_room |
| 检测流 | Channels/102（低清，独立） | xiaomi_living_room（共用） | xiaomi_kid_room（共用） |

> **为什么小米只有单路流？**
> miloco 当前仅支持 STREAM_CHANNEL: 0（主流），STREAM_CHANNEL: 1（子流）无法推流。
> 因此小米摄像头录像和检测共用同一路 RTSP 流，Frigate 内部进行分辨率缩放。

---

## 数据流详解

### 海康威视

```
海康威视 (192.168.2.28)
  ├── 主流 RTSP :554/Channels/101  →  Frigate kid_room [record]
  └── 子流 RTSP :554/Channels/102  →  Frigate kid_room [detect]
```

### 小米摄像头（客厅）

```
小米摄像头（小米云注册，局域网通信）
  └── 小米私有加密协议
       └── miloco（:8001，network_mode: host）
            官方协议解密中间件
            └── micam（容器）
                 从 miloco 拉取视频，推送标准 RTSP
                 RTSP_URL: rtsp://192.168.2.61:8556/xiaomi_living_room
                 └── go2rtc（:8556）
                      接收推流，对外提供 RTSP
                      └── Frigate living_room [record + detect]
```

### 小米摄像头（儿童房）

```
小米摄像头（DID: 1175497596）
  └── miloco（:8001）
       └── micam_kid_room（容器）
            RTSP_URL: rtsp://192.168.2.61:8556/xiaomi_kid_room
            └── go2rtc（:8556）
                 └── Frigate kid_room_xiaomi [record + detect]
```

---

## Frigate 配置要点

```yaml
detectors:
  ov:
    type: openvino
    device: GPU     # 必须是 GPU，不能用 AUTO
                    # N5105 不支持 AVX-512，AUTO 会触发崩溃

cameras:
  kid_room:                          # 海康威视：双码流分离
    ffmpeg:
      inputs:
        - path: rtsp://...Channels/101
          roles: [record]            # 主流高清录像
        - path: rtsp://...Channels/102
          roles: [detect]            # 子流低清检测（节省 CPU）

  living_room:                       # 小米客厅：单流共用
    ffmpeg:
      inputs:
        - path: rtsp://192.168.2.61:8556/xiaomi_living_room
          roles: [record, detect]

  kid_room_xiaomi:                   # 小米儿童房：单流共用
    ffmpeg:
      inputs:
        - path: rtsp://192.168.2.61:8556/xiaomi_kid_room
          roles: [record, detect]

record:
  enabled: true
  retain:
    days: 30
    mode: all       # 全时段连续录像
  alerts:
    pre_capture: 10
    post_capture: 30
    retain:
      days: 30
  detections:
    pre_capture: 5
    post_capture: 15
    retain:
      days: 30
```

---

## MQTT 消息流

```
Frigate 检测到目标（人/猫/狗）
  └── 发布 MQTT → Mosquitto（:1883，network_mode: host）
       └── Home Assistant 订阅
            └── 触发自动化
                 例：kid_room_person_occupancy → 开灯
```

**Mosquitto 必须使用 `network_mode: host`**，否则 Docker bridge 网络隔离导致 Frigate 无法连接。

---

## Home Assistant 集成来源

```
HA 数据来源
├── Frigate 集成（HTTP API :5000 + MQTT）
│   ├── camera.kid_room             海康威视（经 Frigate）
│   ├── camera.living_room          小米客厅（经 Frigate）
│   ├── camera.kid_room_xiaomi      小米儿童房（经 Frigate）
│   └── binary_sensor.*_person_occupancy  人员检测
│
└── Xiaomi Home 集成（小米云 API）
    ├── sensor.*_people_number       实时人数
    ├── event.*_someone_appeared     有人出现事件
    ├── event.*_no_human_appear      无人事件
    └── switch.*_human_detection     AI 检测开关
```

---

## 存储路径

```
UGREEN DX4600 NAS
├── /volume1/docker/homeassistant/config/   ← HA 配置
└── /volume2/homeRecording_KID_FIX/
    ├── frigate-config/
    │   └── config.yml                       ← Frigate 摄像头配置
    ├── recordings/                          ← 录像（Frigate 管理，30天）
    ├── micam/
    │   ├── docker-compose.yml               ← 小米桥接服务
    │   ├── go2rtc/go2rtc.yaml               ← go2rtc 流定义
    │   └── miloco/                          ← miloco 认证数据
    └── mosquitto-config/                    ← Mosquitto 配置
```

---

## 关键端口

| 端口 | 服务 | 说明 |
|------|------|------|
| 8123 | Home Assistant | Web UI |
| 5000 | Frigate | Web UI + REST API |
| 8554 | Frigate | RTSP 输出 |
| 8556 | go2rtc_xiaomi | 小米摄像头 RTSP 服务 |
| 1985 | go2rtc_xiaomi | go2rtc Web API（流状态查询） |
| 1883 | Mosquitto | MQTT |
| 8001 | miloco | 小米协议解密中间件 |
