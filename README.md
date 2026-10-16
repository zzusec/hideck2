# HiDeck2

> **[yibaiba/hideck](https://github.com/yibaiba/hideck) 的个人工作副本** —— 用于承载本地测试与补丁，修改只留在本仓库，不向上游推送。
>
> 远端约定：`origin` = 本仓库（提交 / 推送目标）；`upstream` = yibaiba/hideck（仅用于 `git fetch upstream` 同步官方更新，push 已禁用）。

## 这是什么项目

HiDeck 是自托管的**高通 4G / LTE / 5G 模组控制台**，在浏览器中管理 USB 蜂窝模组：

- **设备**：USB 自动发现；QMI / MBIM / AT / PC-SC；射频与 SIM 身份实时状态
- **代理**：绑定模组的 SOCKS5 / HTTP 出口（`SO_BINDTODEVICE`）
- **短信**：收件箱、发送、联系人（支持 iOS / Google / 三星 / 小米 / 华为 / OPPO / vivo 的 vcf/csv 导入）
- **电话 / VoWiFi**：WiFi Calling（SWu / IMS）、蜂窝软件 IMS、原生 VoLTE；**国家前置代理池**（按 SIM 归属国路由 VoWiFi 出口，可钉单节点或强制直连）；通话录音
- **eSIM**：档案下载、启用 / 停用 / 重命名 / 删除；激活码或二维码 / PDF 导入
- **自动化与通知**：定时任务；Telegram / 邮件 / Bark / 飞书 / 企业微信 / 微信 / QQ

完整功能、安装方式与协议说明见下方官方 README。

## 相对官方代码的修改

### 1. 卡级前置代理悬空硬失败 —— 补测试（2026-10-16）

- **位置**：`internal/device/pool_vowifi_country_proxy_test.go`（commit `7a4e459`，已合入 main）
- **文件**：`internal/device/pool_vowifi_country_proxy_test.go`（+105 行）
- **背景**：卡级策略 `card_policies.vowifi_upstream_proxy_id` 指向的代理被删除或禁用后，VoWiFi 启动必须**硬失败**——不允许静默回落国家规则或直连，否则被钉住的出口会悄悄漂移。官方既有测试只覆盖了「代理不存在」一种情况。
- **内容**：
  - `TestResolveCardVoWiFiUpstreamProxyBranches`：六个分支——无策略 / 空值回落、`direct` 强制直连、代理不存在硬失败、代理已禁用硬失败、代理可用
  - `TestResolveVoWiFiCountryProxyDoesNotFailOpenDisabledPinnedRoute`：钉了「已禁用」代理时不得回落国家规则
- **运行**：`CGO_ENABLED=0 go test ./internal/device/ -run 'TestResolveCardVoWiFiUpstreamProxyBranches|TestResolveVoWiFiCountryProxy' -v`

### 2. CatBusy 自动恢复补丁 `v2.1.23-catbusy1`（未入库，仅生产二进制部署）

- **状态**：源码未保存在任何仓库；以二进制形式部署在生产机，基于官方 v2.1.23 构建。
- **背景**：eUICC 里挂着未终结的 CAT/REFRESH 会话时，`EnableProfile` 返回 `catBusy`（SGP.22 result=5）。官方实现只重试 3 次（每次间隔 1s）后放弃，`card_reset_settling` 阶段也只 sleep 约 2.7s、**并不真正复位卡**——卡会永久卡死，只能外部断电复位。
- **补丁行为**（`internal/esim` 包）：
  - `enableProfileRecoveringCatBusy` / `disableProfileRecoveringCatBusy`：档案启用 / 停用改走带恢复的流程
  - `runProfileOpRecoveringCatBusy` → `retryProfileOpWhileCatBusy`：命中 catBusy 时进入 `catbusy_recovery` 阶段重试
  - `resetSIMForCatBusyRecovery` → `resetATSIMForCatBusy`：通过 AT `CFUN=0` → `CFUN=1` 对模组 / SIM 真实断电-上电
  - `bringModemOnlineForCatBusyRecovery`：恢复后把模组拉回在线；`recover_attempts` 记录恢复次数
- **部署方式**（生产机）：
  - 运行文件 `/opt/hideck/local/hideck`（`chattr +i` 锁定）
  - 官方升级脚本检测到 `/opt/hideck/KEEP_LOCAL_PATCH` 时直接跳过，不会被覆盖
  - 恢复官方自动升级：删除 `KEEP_LOCAL_PATCH` → 把 `/etc/init.d/hideck` 的 `command` 改回 `/opt/hideck/hideck` → 执行 `chattr -i /opt/hideck/local/hideck`
- **现状**：截至 2026-10-16，上游最新代码（main / v2.1.32）仍无真实复位逻辑，补丁仍然必要；源码待按二进制行为重建后再入库。

---

以下为官方 README 原文。

<div align="center">

# HiDeck

**Self-hosted console for Qualcomm 4G / LTE / 5G modems**

**English** · [简体中文](i18n/README.zh-CN.md) · [日本語](i18n/README.ja.md) · [한국어](i18n/README.ko.md) · [Español](i18n/README.es.md) · [Français](i18n/README.fr.md) · [Deutsch](i18n/README.de.md) · [Português](i18n/README.pt-BR.md) · [Русский](i18n/README.ru.md)

<p>
  <a href="https://github.com/yibaiba/hideck/releases/latest"><img src="https://img.shields.io/github/v/release/yibaiba/hideck" alt="Release"></a>
  <a href="https://hub.docker.com/r/yibaiba/hideck"><img src="https://img.shields.io/badge/docker-yibaiba%2Fhideck-2496ED?logo=docker&logoColor=white" alt="Docker"></a>
  <a href="go.mod"><img src="https://img.shields.io/badge/Go-1.26.4%2B-00ADD8?logo=go" alt="Go"></a>
  <a href="web/package.json"><img src="https://img.shields.io/badge/Vue-3-42b883?logo=vue.js" alt="Vue 3"></a>
  <a href="LICENSE"><img src="https://img.shields.io/static/v1?label=License&message=PolyForm%20NC%201.0.0&color=blue" alt="License: PolyForm Noncommercial 1.0.0"></a>
</p>

</div>

Manage USB modems, cellular proxy, SMS, WiFi calling / IMS voice, eSIM, and scheduled tasks from a browser.

| | |
| --- | --- |
| Web UI | `http://YOUR_IP:7575` |
| Default login | `admin` / `admin` (change on first sign-in) |
| Database | SQLite (`data/hideck.db`) |

## Features

- **Devices** — USB auto-discovery; QMI, MBIM, AT, PC/SC; live radio and SIM identity
- **Proxy** — SOCKS5 / HTTP bound to the modem with `SO_BINDTODEVICE`
- **SMS** — inbox, send, contacts; vcf/csv from iOS, Google, Samsung, Xiaomi, Huawei, OPPO, vivo
- **Phone** — WiFi calling, cellular software IMS, or native VoLTE; hold, call waiting, listen-only answer
- **VoWiFi** — SWu / IMS, recordings; country proxy pool, pin a node, or go direct
- **Native VoLTE** — modem IMS (`phone_mode=volte`), no ePDG; unique China MBN mapping only
- **eSIM** — download, enable, disable, rename, delete; activation code or QR / PDF
- **Automation & notify** — scheduled tasks; Telegram, email, Bark, Feishu, WeCom, WeChat, QQ

Protocol notes: [VoWiFi](docs/vowifi-protocol-alignment.md) · [VoLTE](docs/volte-native.md) · [operators](docs/operator-notes.md) · [hardware](docs/modem-hardware.md) · [module voice troubleshooting](docs/modem-voice-troubleshooting.md)

Outgoing call/SMS quotas are per SIM and survive restarts: [limits and configuration](docs/outbound-limits.md).

Modem carrier profiles: [MBN management, SIM policies and manual trial runs](docs/mbn.md).

WebDAV backup and safe local cleanup: [archive SMS, calls and recordings](docs/webdav-archive.md).

## Quick start

Docker (recommended). Needs Linux, curl, Docker Compose, host networking, and USB access. The image includes ADB, ALSA tools, AMR/MP3 libraries, and the embedded module-side voice runtime. The host still needs a USB audio driver; see [module voice prerequisites](DOCKERHUB.md#module-voice-dependencies).

```bash
curl -fsSL https://raw.githubusercontent.com/yibaiba/hideck/main/deploy.sh | sh
```

```bash
curl -fsSL https://raw.githubusercontent.com/yibaiba/hideck/main/deploy.sh | HIDECK_DIR=/opt/hideck sh
```

Then open `http://YOUR_IP:7575`.

With a public hostname (DNS A/AAAA to the host, open `443/TCP` and `443/UDP`):

```bash
curl -fsSL https://raw.githubusercontent.com/yibaiba/hideck/main/deploy.sh | \
  HIDECK_DOMAIN=hideck.example.com \
  HIDECK_DIR=/opt/hideck sh
```

Image: `yibaiba/hideck:latest`. Compose uses `network_mode: host`, `privileged: true`, `/dev`, and persists `config/`, `data/`, `logs/`. See [DOCKERHUB.md](DOCKERHUB.md) and [HTTPS / WebRTC](docs/https-webrtc.md).

For Feiniu fnOS, the optional [FPK packaging](packaging/fnos/README.md) reuses the Docker image with an application entry and persistent app directories; on-device validation is still required.

```bash
docker compose ps
docker compose logs -f hideck
```

For PC/SC USB smart-card readers, also configure the host service and optional
Compose override described in [Docker PC/SC setup](DOCKERHUB.md#pcsc-smart-card-readers).

## Screenshots

![Login](docs/images/login.jpg)

![Dashboard](docs/images/dashboard.jpg)

| Devices | Phone |
| --- | --- |
| ![Devices](docs/images/devices.jpg) | ![Phone](docs/images/phone.jpg) |

| Commands | Proxy |
| --- | --- |
| ![Commands](docs/images/commands.jpg) | ![Proxy](docs/images/console-proxy.jpg) |

## Binary install

The installer also installs ADB, ALSA tools and recording libraries, then checks ADB capabilities and host USB audio support. Missing dependencies produce a nonzero exit status even if the HiDeck service was installed; they are not reported as a fully successful deployment. These checks do not connect to or reboot a modem.

OpenWrt keeps module voice optional: use the separate `hideck-adb` and `hideck-modem-voice` packages. See [OpenWrt packaging](packaging/openwrt/README.md) for IPK/APK installation and target matching.

```bash
curl -fsSL https://raw.githubusercontent.com/yibaiba/hideck/main/deploy-binary.sh | sh
```

```bash
curl -fsSL https://raw.githubusercontent.com/yibaiba/hideck/main/deploy-binary.sh | \
  HIDECK_DIR=/opt/hideck \
  HIDECK_VERSION=v2.1.23 \
  HIDECK_ARCH=linux_amd64 sh
```

| Asset | Platform |
| --- | --- |
| `hideck_v2.1.23_linux_amd64` | x86_64, glibc + UPX |
| `hideck_v2.1.23_linux_arm64` | ARM64 / Raspberry Pi OS 64-bit, glibc + UPX |
| `hideck_v2.1.23_linux_armv7` | 32-bit ARM, glibc + UPX |
| `hideck_v2.1.23_openwrt_amd64` | OpenWrt x86_64, musl static, no UPX |
| `hideck_v2.1.23_openwrt_arm64` | OpenWrt aarch64 |
| `hideck_v2.1.23_openwrt_armv7` | OpenWrt 32-bit ARM |
| `hideck_v2.1.23_openwrt_dynamic_amd64` | OpenWrt x86_64, dynamic musl, MP3 encoding capable |
| `hideck_v2.1.23_openwrt_dynamic_arm64` | OpenWrt aarch64, dynamic musl, MP3 encoding capable |
| `hideck_v2.1.23_openwrt_dynamic_armv7` | OpenWrt 32-bit ARM, dynamic musl, MP3 encoding capable |

On OpenWrt, use `openwrt_*` only. The package-manager archives and binary installer use the static assets. The dynamic assets require matching runtime libraries and are intended for MP3 call recording; see [OpenWrt packaging](packaging/openwrt/README.md#openwrt-dynamic-musl).

## Configuration

Copy [config/config.example.yaml](config/config.example.yaml) to `config/config.yaml`.

| Key | Default | Notes |
| --- | --- | --- |
| `server.port` | `7575` | HTTP |
| `server.https_enabled` | `false` | Built-in HTTPS; keep off behind a reverse proxy |
| `server.https_port` | `7576` | Built-in HTTPS port |
| `server.webrtc_udp_address` | `:7580` | WebRTC audio UDP |
| `server.webrtc_public_host` | empty | Public ICE host behind NAT |
| `web.username` / `web.password` | `admin` | Change immediately |
| `vowifi.enabled` | `false` | Global VoWiFi |
| `system.openwrt_dynamic_interfaces` | `false` | OpenWrt netifd mapping |

Do not commit secrets. SIM PIN fields store environment variable *names* only (`HIDECK_SIM_PIN_READER1`). `PROXY_WEB_PASSWORD` overrides the file.

## Build

Go `1.26.4+`, Node.js / npm. `make build-*` requires UPX.

```bash
make build-amd64    # also: build-arm64, build-armv7, build-all
```

```bash
npm ci --prefix web && npm run build --prefix web
cp -R web/dist internal/web/dist
GOWORK=off CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
  go build -trimpath -tags "with_utls nomsgpack" -o dist/hideck_linux_amd64 ./cmd/hideck
```

```bash
npm test --prefix web && npm run typecheck --prefix web
go test -timeout=60s ./cmd/... ./internal/... ./pkg/...
```

Dev proxy: `VITE_API_PROXY_TARGET=http://127.0.0.1:7575` in `web/.env.local`. API docs: `/api/docs`.

## License

Personal learning, research, and testing. Not a production telephony stack. Independent of Quectel, Qualcomm, and carriers. Follow local law and your operator’s terms.

[PolyForm Noncommercial 1.0.0](LICENSE). `third_party/vowifi-go` is AGPL-3.0. See [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) before distributing binaries or images.

Thanks: [LINUX DO](https://linux.do/) · [iniwex5/vohive-release](https://github.com/iniwex5/vohive-release) · [boa-z/vowifi-go](https://github.com/boa-z/vowifi-go)
