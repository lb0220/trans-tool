# 星闪传（TransTool）—— 手机与电脑快速传文件

微信小程序客户端 + **Go 编写的多平台服务端**，通过局域网在手机和电脑之间批量传输文件。
支持 **SECRET_KEY 鉴权、实时探活、批量选图上传、分块传输、断点续传、失败重试、传输历史（可一键清空）**。

## 手机服务端模式（手机互传）

小程序不只做客户端——在连接页点击「切换为服务端模式」，本机即可变成接收端，让**另一台手机**扫码向它传文件：

- 小程序没有 TCP 服务端 API，手机服务端基于 **UDP 数据报协议**（`wx.createUDPSocket`，需基础库 2.18+）：
  `hello` 握手鉴权 → `meta` 建立会话 → `chunk` 分块传输（ACK 确认 + 超时重传）→ `done` 落盘
- 启动后自动绑定 UDP 端口 8765，生成密钥并**在本机渲染连接二维码**（`utils/qrcode.js` 零依赖编码器），
  内容格式为 `IP:端口:密钥:UDP`（电脑端为 `IP:端口:密钥`，客户端扫码自动识别类型）
- 接收的文件保存到小程序用户目录 `received/`，接收记录在服务端模式页可查看/清空
- 客户端扫码连接手机服务端后，传输页自动切换到 UDP 上传器（`utils/udp-uploader.js`），
  进度/暂停/续传/火箭特效等体验与电脑端完全一致

## 目录结构

```
TransTool/
├── dist/                       # 免安装可执行文件（双击/一键运行）
│   ├── transtool-server.exe               # Windows amd64
│   ├── transtool-server-macos-amd64       # macOS Intel
│   ├── transtool-server-macos-arm64       # macOS Apple Silicon
│   ├── transtool-server-linux-amd64       # Linux x64
│   └── transtool-server-linux-arm64       # Linux ARM（树莓派等）
├── server-go/                  # 服务端源码（Go，零第三方依赖）
├── miniprogram/                # 微信小程序客户端（用微信开发者工具打开此目录）
│   ├── pages/connect/          # 连接配置页：IP / 端口 / SECRET_KEY，自动探活
│   ├── pages/transfer/         # 传输页：相册批量选图 + 自定义文件 + 进度/续传/重试
│   ├── pages/history/          # 历史记录页（支持一键清空）
│   ├── pages/server/           # 手机服务端模式：UDP 接收服务 + 连接二维码
│   └── utils/                  # api.js（网络层） uploader.js（HTTP 分块上传器）
│                               # udp-uploader.js（UDP 上传器） udp-server.js（UDP 接收服务）
│                               # qrcode.js（二维码编码器，零依赖）
└── README.md
```

## 快速开始

> 服务端二进制托管在 Gitee 仓库：https://gitee.com/lb0220/trans-tool
> 小程序首页（连接页）底部内置了「下载并运行服务端」引导卡片，用户可一键复制该链接到电脑浏览器下载；
> 链接地址在 `miniprogram/pages/connect/connect.js` 顶部的 `DOWNLOAD_URL` 常量中维护。

### 1. 启动电脑端服务（免安装，一键运行）

服务端启动后会在终端打印**连接二维码**（内含 IP / 端口 / 密钥），
打开小程序连接页点击「扫码自动连接」扫一下即可完成配置并建立连接；
也可以按传统方式手动输入。二维码使用纯 ASCII 字符渲染，Windows / macOS / Linux 终端均不会乱码。

从 `dist/` 选择对应平台的文件直接运行：

| 平台 | 文件 | 运行方式 |
|---|---|---|
| Windows | `transtool-server.exe` | 双击运行，或 `.\transtool-server.exe` |
| macOS | `transtool-server-macos-*` | `chmod +x transtool-server-macos-arm64 && ./transtool-server-macos-arm64` |
| Linux | `transtool-server-linux-*` | `chmod +x transtool-server-linux-amd64 && ./transtool-server-linux-amd64` |

- **默认端口 8765**，如需更改：`./transtool-server 9000`（第一个参数即端口）。
- 启动后打印 **本机 IP、端口和 SECRET_KEY**：
  - SECRET_KEY 不落本地文件，首次启动自动生成并写入**系统环境变量 `TRANS_SECRET_KEY`**
    （Windows 写入用户级环境变量 setx；macOS/Linux 追加到 `~/.zshrc` 或 `~/.bashrc` 的 export 行），
    后续启动从该变量读取，密钥保持稳定。也可以在启动前自行预设该变量。
- 接收的文件存放在可执行文件同目录的 `received/` 下。
- macOS 首次运行如被 Gatekeeper 拦截：系统设置 → 隐私与安全性 → 仍要打开。
- Linux/macOS 记得防火墙放行端口；Windows 双击运行时会弹防火墙授权，点击允许。

### 2. 配置并连接小程序

1. 用微信开发者工具打开 `miniprogram/` 目录（appid 可用测试号）。
2. 在「详情 → 本地设置」勾选 **不校验合法域名**（开发阶段直连 http 局域网地址）。
3. 小程序首页输入服务端 IP、端口、SECRET_KEY，点击「建立连接」。
   连接成功后自动进入传输页，并每 5 秒实时探活（页面顶部绿点表示在线）。
   下次打开若配置有效会自动连接并跳转。

### 3. 传输文件

- **上传目录**：页面顶部可选择目标目录，或「+ 新建目录」（支持子路径如 `travel/2026`），
  上传时服务端会**同步创建该目录**，文件落入 `received/<目录>/`。
  目录芯片右上角的 **×** 可删除目录（仅删除客户端本地记录，服务端目录与已传文件不受影响）。
- **修改连接**：连接成功进入传输页后，可随时点击顶部「修改连接」重新配置 IP/端口/SECRET_KEY 并重连。
- **统一添加文件**：点击「+ 添加文件」，可从相册（`mediaType: mix`，图片视频**同次混选**）、
  聊天记录（任意格式文件）或拍摄中批量选择最多 20 个文件，**不区分图片/视频/其它格式，统一走同一上传流程**；
  也可多次「+ 添加文件」从不同来源追加，一起上传。
- **并发上传**：默认 20 并发，点击并发卡下拉可切换 30/40/50（选择本地记忆，无需解锁）。
- **文件数上限**：默认单次可选 100 个文件，点击文件数卡下拉可切换 200/300/400/500；
  相册选择受微信平台单次 20 个硬限制，超出后会**自动分批连选**（每轮 20 个、自动累计到目标数量），
  聊天记录一次最多 100 个（平台限制）；一次选择超过 200 个时会提示可能占用较多内存，建议分批上传。
  （注：微信小程序沙盒机制无法按路径浏览手机整个文件系统，故通过系统选择器近似实现）
- 列表内单个文件可单独「上传」，也可「全部上传」。
- 传输中显示实时进度（按分块粒度更新）；支持 **暂停/继续**；
  失败自动重试 3 次，仍失败可手动「重试」；中断后重新上传同一文件会 **自动断点续传**。

### 4. 查看历史记录

底部 Tab「历史」或传输页右上角入口，可查看所有已传文件记录（文件名、大小、时间、在电脑上的保存路径），支持下拉刷新。

## 接口协议（服务端 API）

所有请求需带请求头 `X-Secret-Key: <SECRET_KEY>`，否则返回 401。

| 方法 | 路径 | 说明 |
|---|---|---|
| GET  | `/api/ping` | 探活，返回 `{ok, ts}` |
| POST | `/api/upload/init` | body `{name, size, lastModified, chunkSize}` → `{uploadId, received:[已收分块号], completed}` |
| GET  | `/api/upload/exists?name=&dir=` | 检查目标目录下是否已有同名文件（上传前弹窗询问覆盖/跳过） |
| POST | `/api/upload/chunk?uploadId=&index=` | body 为该分块原始字节（octet-stream） |
| POST | `/api/upload/complete` | body `{uploadId, name, size, overwrite}`，合并分块并校验大小；同名且未选覆盖时返回 `skip:true` |
| GET  | `/api/upload/status?uploadId=` | 查询分块接收情况 |
| GET  | `/api/history` | 传输历史（最多 500 条） |
| POST | `/api/history/clear` | 一键清空历史（不删除已保存的文件） |

- 分块默认 512KB；`uploadId` 由 `name|size|lastModified` 决定，服务端按分块号去重，天然支持断点续传。
- 合并时校验总大小，写完后在 `received/` 目录生成文件。
- **文件名保持原样**：同名文件上传前小程序会弹窗询问「覆盖 / 跳过」，由用户选择。

## 从源码构建服务端

需要 Go 1.21+，在 `server-go/` 目录：

```bash
# Windows
GOOS=windows GOARCH=amd64 go build -o ../dist/transtool-server.exe
# macOS (Apple Silicon 用 arm64)
GOOS=darwin GOARCH=amd64 go build -o ../dist/transtool-server-macos-amd64
# Linux
GOOS=linux GOARCH=amd64 go build -o ../dist/transtool-server-linux-amd64
```

## 安全提示

SECRET_KEY 为唯一鉴权手段，且通信为明文 HTTP，请仅在**可信局域网**内使用；
不要将端口暴露到公网。如需公网使用，建议前面加反向代理并配置 HTTPS。
