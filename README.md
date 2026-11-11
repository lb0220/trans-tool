# TransTool —— 手机与电脑快速传文件

微信小程序客户端 + **Go 编写的多平台服务端**，通过局域网在手机和电脑之间批量传输文件。
支持 **SECRET_KEY 鉴权、实时探活、批量选图上传、分块传输、断点续传、失败重试、传输历史（可一键清空）**。

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
│   └── utils/                  # api.js（网络层） uploader.js（分块上传器）
└── README.md
```

## 快速开始

> 小程序首页（连接页）底部内置了「下载并运行服务端」引导卡片，
> 用户可一键复制下载链接到电脑浏览器下载。发布服务端二进制后，
> 请把 `miniprogram/pages/connect/connect.js` 顶部的 `DOWNLOAD_URL`
> 替换为实际的下载地址。

### 1. 启动电脑端服务（免安装，一键运行）

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
- **统一添加文件**：点击「+ 添加文件」，可从相册（照片/视频）、聊天记录（任意格式文件）或拍摄中批量选择
  最多 20 个文件，**不区分图片/视频/其它格式，统一走同一上传流程**。
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
