# Snapcast 多房间同步（NAS 推流 + Windows 采集）

把 43 电脑的声音采集出来，推到绿联 NAS 上的 snapserver，再由 NAS 分发给多台手机
（Snapdroid）和电脑（snapclient），所有客户端**按同一时间轴播放**。

```
43（Windows）
  └─ SnapcastCapture 服务（WASAPI loopback → PCM/TCP）
            │ 4953
            ▼
绿联 DXP4800（Docker：snapserver，host 网络）
            │ 1704 / 1705 / 1780
            ├─ 手机 1..N：Snapdroid → AUX → 音箱
            └─ 电脑：snapclient_win64
```

网络统一走 **192.168.1.x**，NAS 用第二个网口的 **192.168.1.11**。

## 目录

```
snapcast/
├─ docker/
│  ├─ docker-compose.yml   # NAS 上的 snapserver
│  └─ snapserver.conf      # TCP 源、buffer、codec
├─ windows-capture/        # 43 上的采集桥（C#，无第三方运行时）
│  ├─ src/
│  ├─ lib/NAudio.dll
│  ├─ build.ps1
│  ├─ install.ps1 / uninstall.ps1
│  └─ capture.ini
└─ dist/                   # 打好的采集桥压缩包（生成）
```

## 1. NAS：启动 snapserver（图形界面，不用 SSH）

### 1.1 建目录

在 UGOS「文件管理」里选一个存储空间，新建共享文件夹 `snapcast`，里面再建两个子目录：

```
snapcast/
├─ config/
└─ data/
```

记下它在 NAS 上的实际路径（通常是 `/volume1/snapcast`，以文件管理里显示的为准）。
Windows 上也可以映射网络驱动器 `\\192.168.1.11\snapcast` 直接拷文件。

### 1.2 放配置

把 `docker/snapserver.conf` 上传到 `config/`，文件名必须是 `snapserver.conf`。
没有现成文件的话，在 Windows 记事本里新建、粘贴内容，另存为 UTF-8，再传上去。

### 1.3 拉镜像

UGOS Docker → 镜像 → 拉取，填：

```
sweisgerber/snapcast:v0.35.0
```

### 1.4 创建容器

Docker → 容器 → 创建容器：

| 项目 | 填法 |
| --- | --- |
| 镜像 | `sweisgerber/snapcast:v0.35.0` |
| 容器名称 | `snapserver` |
| 网络 | **host**（主机网络） |
| 存储空间 | `/volume1/snapcast/config` → `/config` |
| 存储空间 | `/volume1/snapcast/data` → `/data` |
| 环境变量 | `PUID=1000`、`PGID=1000`、`TZ=Asia/Shanghai` |
| 重启策略 | 除非手动停止（unless-stopped） |
| 端口 | host 模式不需要映射 |

创建后启动。

### 1.5 验证

- 容器日志里应看到 1704 / 1705 / 1780 在监听
- 浏览器打开 `http://192.168.1.11:1780`，出现 snapweb 就正常
- 如果日志报配置错误：先确认文件名是 `config/snapserver.conf`；还不行就用镜像默认配置启动，再把 `source = tcp://192.168.1.11:4953?name=pc&mode=server` 加进去

> 如果 UGOS 没有 host 选项，用桥接网络并映射 `1704/1705/1780/4953` 四个 TCP 端口，
> 同时把 `snapserver.conf` 的 source 改成 `tcp://0.0.0.0:4953?name=pc&mode=server`。

> 会用 SSH 的话也可以直接 `docker compose up -d`，文件在 `docker/docker-compose.yml`。

## 2. 43：安装采集桥

在 43 上解压 `dist/snapcast-capture-1.1.0.zip`，右键 `install.ps1` → 以管理员身份运行。
脚本会安装服务 `SnapcastCapture`（开机自启），采集默认播放设备并推到 NAS。

安装后确认：

```powershell
Get-Service SnapcastCapture
Get-Content "C:\Services\snapcast-capture\logs\capture-$(Get-Date -f yyyyMMdd).log" -Tail 10
```

应看到 `采集设备: 48000 Hz ...` 和 `已连接 192.168.1.11:4953`。

NAS 地址/端口在 `C:\Services\snapcast-capture\capture.ini`，改完 `Restart-Service SnapcastCapture`。

采集桥默认**跟随 43 的 Windows 主音量**（`capture.ini` 里 `followvolume=1`）：
拖动 43 的任务栏音量，所有手机一起变。改成 `followvolume=0` 后忽略系统音量，
输出固定满幅，音量只由 Snapdroid / snapweb 控制。

## 3. 手机：Snapdroid

- 安装 [Snapdroid](https://github.com/badaix/snapdroid/releases/latest)（F-Droid / Google Play 也有）
- 添加服务器 `192.168.1.11`，选择设备并播放
- 关闭电池优化、允许后台；长期使用建议插电并保持常亮
- AUX 接有源音箱

电脑端如果要一起听，用官方 `snapclient_win64.zip`，命令：

```powershell
snapclient.exe --host 192.168.1.11
```

## 4. 验证同步

- 两台手机放在一起播放，应听不到回声/前后差
- 打开 `http://192.168.1.11:1780`（snapweb）能看到所有客户端
- 调整音量、分组也在 snapweb 里做

## 调优

| 问题 | 调整 |
| --- | --- |
| 偶尔卡顿/断音 | `snapserver.conf` 的 `buffer` 调到 400–500，重启容器 |
| 想省带宽 | 保持 `codec = opus`（10 台约 1.3 Mbps） |
| 局域网带宽充足、想更稳 | `codec = flac` |
| 手机声音有延迟差 | 关闭手机上的音效（杜比/均衡器/空间音频） |
| 手机被省电杀掉 | 电池优化设为“不限制”，允许后台 |

## 排错

- **采集桥日志“发送失败”**：NAS 的 4953 没监听，或地址不是 192.168.1.11
- **NAS 日志没有 source**：检查 `snapserver.conf` 的 `source` 行和 host 网络
- **手机连不上**：确认手机能打开 `http://192.168.1.11:1780`；双网卡时手动填 1 网段 IP
- **有声音但不同步**：确认手机装的是 Snapdroid（不是 as-cmd App）；关闭音效；buffer 不要低于 150
- **NAS 重启后电脑没恢复**：采集桥会自动重连（每 2 秒），日志里会看到重连记录

## 说明

- 这套和原来的 as-cmd 服务互不影响，as-cmd 继续用 65530。
- 采集桥只负责“把系统声音变成 PCM”，不做业务逻辑；源码在 `windows-capture/src`。
- Snapcast 服务端在容器里，升级只需改 `docker-compose.yml` 的镜像 tag。
