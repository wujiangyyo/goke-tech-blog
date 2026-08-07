---
title: "小米老款摄像头解锁本地 RTSP：micam + miloco + go2rtc 实战全记录"
description: "2019 年小米云台摄像头（chuangmi.camera.ipc019）纯云零端口、无法本地取流？通过 miloco + micam + go2rtc 三件套破解 denylist，拿到 1080P HEVC 本地 RTSP 流。含 WSS 拉流、桥接网络、HEVC 黑帧等全部踩坑记录。"
tags: [xiaomi, camera, rtsp, miloco, micam, go2rtc, homelab, iot]
categories: ["技术"]
author: "果壳科技 塔塔"
date: 2026-08-08T02:50:00+08:00
draft: false
weight: 4
image: /goke-tech-blog/img/cover-miloco-camera.jpg
ShowToc: true
TocOpen: true
---

## 背景

家里的米家摄像头「大眼睛」是 2019 年的老款云台版（chuangmi.camera.ipc019，MJSXJ02CM/MJSXJ05CM）。这类老设备有个共性：**纯云、零开放端口**——80/443/554/8554 全关，只能走米家 App 看，第三方软件（Jellyfin、Frigate、VLC）全都接不进去。

想把它接入本地媒体栈，必须找一条"中间人"链路：用小米账号登录云端取流，再转成标准 RTSP 推给局域网。

## 方案选型

调研后锁定两个项目：

| 项目 | 定位 | 说明 |
|------|------|------|
| XiaoMi/xiaomi-miloco | 小米官方开源全屋智能 AI | 完整版 2.0 是 OpenClaw/Hermes Agent 插件形态，带 MiMo 大模型感知（需 API key 按量付费） |
| miiot/micam | 社区拉流桥接项目 | Docker 三件套（miloco 基础版 + go2rtc + micam），纯拉流、无 GPU 要求，**大部分机器都能跑** |

目标是"先把画面弄进来"，选 **micam 三件套**：轻量、免费、无 AI 依赖。完整版 AI 感知留作后续升级路径。

## 架构

```
米家摄像头（纯云，局域网零端口）
    ↓ 米家云端
miloco 容器（host 网络，REST + WSS 取流，denylist 已破解）
    ↓ WebSocket 视频流（wss://127.0.0.1:8000）
micam 容器（WS 客户端 + ffmpeg 转推）
    ↓ RTSP 推流
go2rtc 容器（:8554，预配空流 dayanjing）
    ↓
rtsp://<C机IP>:8554/dayanjing  （HEVC 1920×1080，局域网任意设备可拉）
```

## 部署步骤

### 1. Docker Compose 三件套

```yaml
services:
  miloco:
    container_name: miloco
    image: ghcr.nju.edu.cn/miiot/miloco:main
    network_mode: host
    environment:
      BACKEND_PORT: ${MILOCO_PORT:-8000}
      TZ: ${TZ:-Asia/Shanghai}
    volumes:
      - ./miloco:/app/miloco_server/.temp
      - ./miloco/configs/camera_extra_info.yaml:/app/miot_kit/miot/configs/camera_extra_info.yaml:ro
    restart: unless-stopped

  go2rtc:
    container_name: go2rtc
    image: ghcr.nju.edu.cn/alexxit/go2rtc
    network_mode: host
    privileged: true
    volumes:
      - ./go2rtc:/config
    restart: unless-stopped

  micam1:
    image: ghcr.nju.edu.cn/miiot/micam:main
    depends_on: [miloco, go2rtc]
    environment:
      MILOCO_BASE_URL: ${MILOCO_BASE_URL:-https://miloco:8000}
      MILOCO_PASSWORD: ${MILOCO_PASSWORD}
      CAMERA_ID: ${CAMERA_ID}
      RTSP_URL: ${RTSP_URL}
      VIDEO_CODEC: ${VIDEO_CODEC:-hevc}
      STREAM_CHANNEL: ${STREAM_CHANNEL:-0}
    restart: always
    extra_hosts:
      - "miloco:${MILOCO_HOST_IP:-host-gateway}"
```

`.env` 文件：

```bash
MILOCO_PASSWORD=<miloco登录密码的MD5>
CAMERA_ID=<摄像头DID>
RTSP_URL=rtsp://<C机IP>:8554/dayanjing
```

### 2. 破解 denylist（老款摄像头关键步骤）

`chuangmi.camera.ipc019` 不在 miloco 官方支持列表，默认被 `camera_extra_info.yaml` 的 denylist 拒之门外。修改容器内文件，把该型号移出黑名单：

```bash
# 容器内路径
docker exec -it miloco sh
vi /app/miot_kit/miot/configs/camera_extra_info.yaml
# 删除/注释 denylist 中 chuangmi.camera.ipc019 条目
# 保留 ipc019b / ipc019e（其他型号）
```

改完重启 miloco，`camera_list` 出现摄像头即破解成功。

**持久化**：容器重建会丢改动，必须把修改后的文件挂载回容器（见上方 compose 的 `:ro` 挂载）。

### 3. miloco 登录 + 确认摄像头在线

```bash
# 登录（admin + 密码MD5），cookie 存文件
curl -sk -X POST https://127.0.0.1:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"<MD5>"}' \
  -c /tmp/miloco_cookies.txt

# 摄像头列表
curl -sk https://127.0.0.1:8000/api/miot/camera_list -b /tmp/miloco_cookies.txt

# 离线时刷新唤醒
curl -sk "https://127.0.0.1:8000/api/miot/refresh_miot_cameras" -b /tmp/miloco_cookies.txt
```

`status: 4` = 正常在线。容器重建后摄像头会短暂离线，刷新一次即恢复。

### 4. WSS 直连验证（绕过 micam，验证链路）

miloco 强制 HTTPS（自签证书），WS 客户端正确姿势：

```python
import websocket

ws = websocket.create_connection(
    "wss://127.0.0.1:8000/api/miot/ws/video_stream?camera_id=<DID>&channel=0",
    header=["Cookie: access_token=<token>"],
    origin="https://127.0.0.1:8000",   # ⚠️ 必须用参数，放 header 会 400
    timeout=10,
    sslopt={"cert_reqs": 0, "check_hostname": False},
)
```

10 秒能收到 101 个 binary 帧（约 90KB）即成功，其中关键帧（IDR）约 23KB。

### 5. go2rtc 预配空流

go2rtc 不预先声明流名就拒绝接收推流。`go2rtc.yaml`：

```yaml
streams:
  dayanjing:
```

重启 go2rtc 后，`GET :1984/api/streams` 能看到流被激活（producer 来自 micam 容器）。

### 6. 验证全链路

```bash
ffprobe -v error -rtsp_transport tcp -i "rtsp://127.0.0.1:8554/dayanjing" \
  -show_entries stream=codec_name,width,height -of csv=p=0
# → hevc,1920,1080
```

## 踩坑记录

### 🔥 坑 1：WSS Origin 400

websocket-client 把 `origin` 放进 header 会报 `invalid Origin header: multiple values`。必须用 `origin=` 参数传。

### 🔥 坑 2：micam 的 127.0.0.1 指向容器自身

micam1 是 bridge 网络，`RTSP_URL` 写 `rtsp://127.0.0.1:8554/...` 会 Connection refused。**必须写宿主 IP**（go2rtc 是 host 网络）。

### 🔥 坑 3：HEVC 黑帧 ≠ 故障

流启动初期 HEVC 参考帧未到齐，立即抓帧是黑的（亮度实测 avg 130、0% 暗像素，其实画面正常）。**等 30 帧后再抓**：

```bash
ffmpeg -rtsp_transport tcp -i "rtsp://.../dayanjing" \
  -t 6 -vf "select=gte(n\,30)" -frames:v 1 out.jpg
```

播放器也一样，连上后等 2-3 秒画面出现，正常现象。

### 🔥 坑 4：目录权限

go2rtc 配置目录容器初始化后是 root 所有，直接编辑会 Permission denied。`sudo chown -R xiaoyu:xiaoyu` 解决。

### 🔥 坑 5：mp4 中断 → moov atom not found

抓帧/转码中途中断会写不出 moov atom。改 TS 容器可救：`.ts` 输出再转 `.mp4`。

## 能力边界（重要）

- ✅ 实时画面 1080P HEVC 本地 RTSP
- ✅ 局域网内任意设备（Jellyfin/Frigate/VLC/ffplay）直接拉流
- ❌ **移动侦测/看家助手/人脸识别**：这些是米家云端 AI 能力，基础版 miloco 没有。需要完整版 Miloco 2.0 + MiMo 大模型 API（按量付费）
- ❌ **扬声器播放语音**：miloco 基础版没有音频上行接口（只有音频接收），RTSP 转推也是纯视频
- ⚠️ 老款摄像头走云端中转（LAN 发现未生效），存在延迟，但画面实测正常

## 后续方向

完整版 Miloco 2.0 可以装成 Hermes Agent 插件，实现"感知看家 + 身份识别 + 主动提醒 + 设备控制"，代价是 MiMo API 按量计费。想玩的话先申请 MiMo Token Plan 试用，跑通再决定长期方案。
