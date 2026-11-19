# FrameCraft

精确截下你暂停的那一帧：保持片源原始分辨率和位深（包括 HDR），所有字幕轨都保留为可切换的图层。

FrameCraft 先向播放器查询暂停位置，只读取片源在这个位置附近的几 MB 数据，解码出屏幕上的那一帧，然后导出三个文件：

- `*_clean.png`：无字幕原图，16 位，原始分辨率，保留 HDR PQ/HLG
- `*_subbed.png`：同一帧，烧入你选择的字幕
- `*_layered.svg`：原图加上每条字幕轨，每轨一个图层，方便在设计软件里继续调整

[English](README.md)

## 功能

- **逐帧准确**：选出显示时段包含暂停位置的那一帧；编码前后都会核对这一帧的时间戳。
- **超大远程文件也能截**：MKV/WebM 按 Cues 索引定位，MPEG-TS/M2TS 按包内时间戳定位。只用多路并行 HTTP Range 请求下载目标附近一段。57 GB 的 M2TS 约 1 分钟出图，不再超时。
- **蓝光 ISO / BDMV（实验性）**：通过 libbluray 读取光盘的播放列表和片段索引，同样可以远程按 Range 读取。
- **字幕图层**：支持 PGS、SRT、ASS/SSA、WebVTT、mov_text 等格式。在字幕工作台里可以开关字幕轨、叠加双语、微调上下位置，也能把图片字幕识别成可编辑文字。离开工作台时自动保存。
- **AB 切片**：先标记起点，再导出起点到当前位置的片段。直接复制视频流，不重新编码。
- **后台队列**：每个截图任务在独立进程里运行，Raycast 不会卡住，也可以同时处理多个截图。

## 环境要求

- macOS
- [FFmpeg](https://ffmpeg.org)：`brew install ffmpeg`
- 仅在截取蓝光 ISO / BDMV 时需要：
  - Xcode 命令行工具（`xcode-select --install`）
  - `brew install libbluray libudfread curl pkg-config`

## 支持的播放器

| 来源 | 获取暂停位置的方式 | 说明 |
|---|---|---|
| 任意 Jellyfin 客户端 | Jellyfin `/Sessions` 接口 | 准确度取决于客户端，见下文 |
| 任意 Emby 客户端 | Emby `/Sessions` 接口 | **实验性**：接口相同，但还没有在真实 Emby 服务器上测试过 |
| IINA | mpv 通信接口（IPC socket） | 本地文件和直链；需要做一次设置，见下文 |
| QuickTime Player | AppleScript | 本地文件 |

### 播放器上报的位置准不准

FrameCraft 截取的就是播放器上报位置上的那一帧。至于这一帧是不是你在屏幕上看到的那一帧，要看播放器。以下是和屏幕截图逐帧比对的结果：

| 播放器 | 结果 |
|---|---|
| **VidHub**（mpv 内核） | MKV、TS、蓝光 ISO 都和屏幕是同一帧。推荐使用。 |
| **IINA**（mpv 内核） | 直接读取 mpv 自己的播放位置。 |
| **Infuse** | 偏差约 −24 到 +11 帧（约 ±1 秒），没有固定规律。 |
| **Swiftly**（VLC 内核） | 同样有明显偏差。 |

原因：
- mpv 内核的播放器暂停时，上报的是屏幕上那一帧的时间。
- Infuse 和 VLC 内核的客户端上报的是播放时钟。暂停时时钟可能停在两帧之间的任意位置，还会叠加音频和画面显示的延迟。这些偏差在播放器外部无法修正。

## 设置

### Jellyfin / Emby

在 Raycast 里打开 FrameCraft 的设置（在任意 FrameCraft 命令上按 `⌘ ,`），填写：

- **Server URL（服务器地址）**：平时在浏览器里打开的地址。如果有子路径（比如 `/jellyfin`），要一并填上。
- **API Key（API 密钥）**：在服务器管理后台的 API 密钥页面创建（Jellyfin：控制台 → API 密钥）。不是登录密码。
- **User Name Filter（只检测这个用户，可选）**：多人同时播放时填写。

先播放视频并暂停，再运行 **Capture Frame**。播放器必须向这台服务器上报播放进度。直接通过 SMB/WebDAV 播放文件不会产生播放会话。

### IINA

1. 打开 IINA → 设置 → 高级，启用高级设置。
2. 在「额外 mpv 选项」里添加一行：名称 `input-ipc-server`，值 `/tmp/framecraft-iina.sock`（名称前不加 `--`）。
3. 完全退出 IINA，再重新打开视频。

这个接口可以控制 IINA，只有本机能访问。删掉这一项就会关闭它。

## 命令

| 命令 | 作用 |
|---|---|
| **Capture Frame** | 截取当前暂停的画面，任务进入后台队列。可选参数：`clean`（不带字幕）、`dual`（中英双语），光盘可以用 `playlist=编号` 指定播放列表。 |
| **Capture Gallery** | 查看任务进度、成片和失败任务。选中一张截图按 ↵，进入字幕工作台。 |
| **Mark Clip Start** | 把当前位置记为切片起点。 |
| **Export Clip** | 导出起点到当前位置的片段。没有标记起点时，导出当前位置之前的 15 秒。 |
| **Copy Stream URL** | 复制正在读取的视频路径或地址。地址里可能带有 API 密钥，不要公开分享。 |

## 设置项

| 设置 | 默认值 | 说明 |
|---|---|---|
| Save Location（保存位置） | `~/Pictures/FrameCraft` | |
| Default Subtitles（默认字幕） | 主字幕轨 | 决定 `_subbed.png` 里显示哪几条字幕。SVG 始终保留所有字幕轨。 |
| FFmpeg Path（FFmpeg 路径） | 自动 | 自动在 Homebrew 的安装位置查找。 |
| Network（网络） | 直连 | 读取视频时不使用系统代理设置。 |
| Clipboard（剪贴板） | 开启 | 截图完成后自动复制图片。 |

## 限制

- 服务器必须支持 HTTP Range 请求（返回 206）。每次远程截图最多读取 512 MiB、发出 512 次请求，超过时直接报错，不会改为下载整个文件。
- 蓝光目前只支持截图，而且是实验性功能。不支持加密光盘和 DVD ISO。光盘的候选播放列表超过 3 个时，需要用 `playlist=编号` 指定。
- HDR（PQ）字幕在线性光下合成，参考白为 203 nits。HLG 片源的字幕颜色转换尚未实现。
- 长时间持续显示的字幕和复杂的 ASS 样式还没有充分验证。

## 隐私

- FrameCraft 只连接你在设置里填写的媒体服务器。
- API 密钥由 Raycast 保存，错误信息里会自动隐去。
- 截图、历史记录和缓存都只保存在你的 Mac 上。缓存 10 分钟后过期。

## 开发

```bash
npm install
npm run dev      # 构建后台进程并启动 Raycast 开发模式
npm run build    # 生产构建
npm run lint
```

- `assets/worker.cjs` 是后台进程的打包文件，由 `npm run build-worker` 从 `src/worker/` 生成。修改了它引用到的代码后，要重新构建。
- `assets/disc-reader.c` 在第一次截取光盘时自动编译。

## 许可证

[MIT](LICENSE)
