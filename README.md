# biu-ffmpeg-build

本项目通过 GitHub Actions 自动从 FFmpeg 最新开发分支构建一个极度精简版的 `ffmpeg`，只保留“流复制（copy 编码）+ 常见封装格式”能力，用于支持如下几类命令：

```bash
ffmpeg -i video -i audio -c:v copy -c:a copy -shortest output
ffmpeg -i video -c:v copy -c:a copy output
ffmpeg -i audio -c:a copy output
```

目标是：

- 不做任何音视频重新编码，只做容器层面的复用 / 封装调整；
- 显著减小二进制体积，便于在 CI/CD、服务器或桌面应用中内置和分发；
- 同时提供 Linux、macOS（Arm/Intel）、Windows 三个平台的可执行文件。

---

## 1. 工作流概览

工作流文件：`.github/workflows/build-ffmpeg-minimal.yml`

触发条件：

- 任意 `push`；
- 手动在 GitHub Actions 页面通过 `workflow_dispatch` 触发。

核心 Job：`build-ffmpeg`，使用矩阵并行构建四种平台：

```yaml
matrix:
  include:
    # Linux x86_64，本地命令行使用最广泛
    - os: ubuntu-latest
      target: linux
    # macOS 15 Arm64（Apple Silicon），原生 arm64 二进制
    - os: macos-15
      target: macos-arm64
      mac_arch: arm64
    # macOS 15 Intel，原生 x86_64 二进制，兼容老款 Intel Mac
    - os: macos-15-intel
      target: macos-x86_64
      mac_arch: x86_64
    # Windows Server 2025，对应桌面 Windows x86_64
    - os: windows-latest
      target: windows
```

每个平台使用相同的 FFmpeg 源码和几乎相同的 `./configure` 参数，只是安装前缀略有差异：

- Linux/macOS：`PREFIX="$HOME/ffmpeg-min"`
- Windows（MSYS2）：`PREFIX="/c/ffmpeg-min"`，对应 Windows 路径 `C:\ffmpeg-min`

构建完成后，分别上传四个 Artifact：

- `ffmpeg-minimal-linux`：Linux 可执行文件；
- `ffmpeg-minimal-macos-arm64`：Apple Silicon 原生二进制；
- `ffmpeg-minimal-macos-x86_64`：Intel Mac 原生二进制；
- `ffmpeg-minimal-windows`：Windows `.exe`。

---

## 2. FFmpeg 源码与构建流程

### 2.1 源码获取

Linux/macOS 步骤：

```bash
git clone --depth 1 https://github.com/FFmpeg/FFmpeg.git ffmpeg
```

Windows 在 MSYS2 shell 中执行相同命令。

- 使用官方 GitHub mirror，对应 FFmpeg 主仓库；
- `--depth 1` 只拉取最新提交，减少 CI 时间和网络流量；
- 没有额外打补丁，始终使用“最新开发分支”的状态。

### 2.2 安装前缀

Linux/macOS：

```bash
PREFIX="$HOME/ffmpeg-min"
```

Windows（MSYS2）：

```bash
PREFIX="/c/ffmpeg-min"
```

- 通过 `--prefix="$PREFIX"` 控制安装目录；
- 避免污染系统路径（例如 `/usr/local`），便于在 CI 里直接打包二进制；
- Windows 下 `/c/ffmpeg-min` 对应 `C:\ffmpeg-min`，上传 Artifact 时使用的是 Windows 路径。

---

## 3. `./configure` 参数逐项说明

Linux 和 macOS 使用的配置命令（Windows 版本除 `--prefix` 路径外完全一致）：

```bash
./configure \
  --prefix="$PREFIX" \
  --disable-debug \
  --disable-doc \
  --disable-ffplay \
  --disable-ffprobe \
  --disable-avdevice \
  --disable-postproc \
  --disable-swscale \
  --disable-network \
  --enable-ffmpeg \
  --disable-everything \
  --enable-protocol=file \
  --enable-demuxer=mov,matroska,flac,mp3,aac,mpegts \
  --enable-muxer=mp4,matroska,flac \
  --enable-parser=aac,h264,hevc,mp3,flac,vorbis,opus \
  --enable-bsf=aac_adtstoasc,h264_metadata,hevc_metadata \
  --enable-small
```

下面按类别解释每个参数存在的理由，以及与“仅支持 copy 的三条命令”的关系。

### 3.1 基础参数

#### `--prefix="$PREFIX"`

- 指定 `make install` 的安装路径；
- 所有编译产物（`ffmpeg` 可执行文件、头文件、库文件等）会被安装到该目录；
- 便于在 CI 中将 `$PREFIX/bin/ffmpeg` 单独打包为 Artifact。

#### `--disable-debug`

- 关闭调试信息生成（如大量的符号表、调试段）；
- 减小二进制体积，提升加载速度；
- 对于作为工具分发的二进制，一般不会直接在生产环境用 gdb 调试，关闭是合理的。

#### `--disable-doc`

- 不生成 man page、HTML 文档等；
- 文档对运行时没有影响，只占用构建时间和存储空间；
- 该项目的目标是生成纯二进制，不需要内置离线文档。

### 3.2 关闭不需要的程序/库

#### `--disable-ffplay`

- 不编译 `ffplay` 播放器；
- ffplay 依赖 SDL 等额外库，体积大且与“只做封装复用”无关；
- 可以显著减少依赖和可执行文件数量。

#### `--disable-ffprobe`

- 不编译 `ffprobe` 探测工具；
- 本项目的使用场景只需要 `ffmpeg` 执行容器级操作，不需要额外的分析工具；
- 如果未来有需求，可以重新启用。

#### `--disable-avdevice`

- 关闭 `libavdevice` 相关功能；
- 这意味着不支持从摄像头、屏幕采集等设备直接采集；
- 对三个典型命令（全部从文件读写）没有影响。

#### `--disable-postproc`

- 禁用后处理库（`libpostproc`）；
- 主要用于视频解码后的画质增强/降噪等；
- 本配置不执行解码，postproc 完全用不到。

#### `--disable-swscale`

- 禁用 `libswscale`（视频缩放/像素格式转换）；
- 只有在存在解码/编码或滤镜时才需要；
- 对纯粹的流复制来说，禁止缩放是安全的，可以进一步减少体积。

#### `--disable-network`

- 禁用所有网络协议（如 `http`、`rtmp`、`rtsp` 等）；
- 三条命令全部是本地文件输入输出，不使用任何网络流；
- 在安全性和体积上都是有利的取舍。

### 3.3 控制可执行程序

#### `--enable-ffmpeg`

- 明确启用 `ffmpeg` 命令行工具；
- 在 `--disable-everything` 的前提下，主动打开所需程序，以免被整体禁止。

### 3.4 从“全禁用”开始按需开启

#### `--disable-everything`

- 在功能层面“一刀切”全部关闭：
  - 所有编解码器（encoder/decoder）；
  - 所有滤镜（filter）；
  - 所有协议、demuxer、muxer、parser、bitstream filter 等；
- 然后逐个用 `--enable-xxx` 重新打开真正需要的部分；
- 这是实现“最小化构建”的关键。

### 3.5 协议（Protocol）

#### `--enable-protocol=file`

- 开启 `file` 协议，用于从本地文件读取和写入；
- 对三条命令都是必需的，因为所有 `-i` 和输出路径都是文件路径；
- 关闭网络后保留 file，即形成“纯离线、本地封装工具”的能力边界。

### 3.6 封装格式：demuxer / muxer

#### `--enable-demuxer=mov,matroska,flac,mp3,aac,mpegts`

- demuxer 负责“从容器中拆流”（读入）；
- 当前启用的格式主要围绕将 **任意常见输入** 重封装成 `mp4` / `m4a` / `flac` / `mkv`：
  - `mov`：包含 `mp4`/`mov`；
  - `matroska`：包含 `mkv`/`webm`；
  - `flac`：FLAC 容器；
  - `mp3`：裸 MP3 文件（方便 `mp3` → `m4a/flac` 场景）；
  - `aac`：裸 AAC（方便 `aac` → `m4a` 场景）；
  - `mpegts`：MPEG-TS 流（常见于直播/录制场景，方便 `ts` → `mp4/mkv`）。

#### `--enable-muxer=mp4,matroska,flac`

- muxer 负责“把流写入容器”（写出），这里只保留三种容器：
  - `mp4`：对应 `.mp4` 及 `.m4a`（纯音频的 mp4 容器）；
  - `matroska`：对应 `.mkv`；
  - `flac`：对应 `.flac`。
- 也就是说，**当前构建只支持生成这四种扩展名的文件**：
  - `.mp4`
  - `.m4a`（本质是 mp4 容器）
  - `.mkv`
  - `.flac`

### 3.7 parser：解析编码数据头信息

#### `--enable-parser=aac,h264,hevc,mp3,flac,vorbis,opus`

- parser 用于对编码数据流（如 H.264/AAC 比特流）做轻量解析，识别帧边界、头信息等；
- 在做流复制、重新封装时，parser 负责帮助 demuxer/muxer 正确处理时间戳和分包；
- 该配置选中了与 `mp4/m4a/flac/mkv` 相关的常见编码：
  - `aac`：高级音频编码；
  - `h264`：H.264/AVC；
  - `hevc`：H.265/HEVC；
  - `mp3`：MP3 音频；
  - `flac`：FLAC 音频；
  - `vorbis`：常见于 Matroska/WebM 的 Vorbis 音频；
  - `opus`：常见于 Matroska/WebM 的 Opus 音频。

### 3.8 Bitstream Filter（bsf）

#### `--enable-bsf=aac_adtstoasc,h264_metadata,hevc_metadata`

- bitstream filter 直接在编码比特流层面做变换，而不解码；
- 当前启用的三个过滤器用处分别是：

1. `aac_adtstoasc`
   - 常用于从 MPEG-TS 或 FLV 封装的 AAC 流转封到 MP4；
   - 将 ADTS 头（ADTS 帧）转换为适合 MP4 的 AudioSpecificConfig/ASC 表示；
   - 典型命令场景：
     - `ffmpeg -i input.ts -c:v copy -c:a copy output.mp4`

2. `h264_metadata`
   - 修改 H.264 比特流中的 SPS/PPS 等元数据；
   - 在某些封装转换或兼容性场景中用于调整标志位；
   - 即使当前命令不显式使用，也属于常见“copy + 重封装” 配置的安全选项。

3. `hevc_metadata`
   - 与 `h264_metadata` 类似，但作用于 HEVC/H.265；
   - 用于在不重新编码的前提下调整 HEVC 比特流元数据。

### 3.9 体积优化

#### `--enable-small`

- 启用内部的一系列尺寸优化：
  - 尽可能使用更紧凑的数据结构；
  - 丢弃一些非关键的调试/诊断字符串；
- 会一定程度影响调试体验，但不影响功能正确性；
- 对于分发给最终用户的工具来说是有利的。

---

## 4. 各平台差异与构建环境

### 4.1 Linux（ubuntu-latest）

- 基于官方 `ubuntu-latest` Runner（当前为 Ubuntu 24.04）；
- 安装依赖：

  ```bash
  sudo apt-get update
  sudo apt-get install -y \
    git \
    build-essential \
    pkg-config \
    yasm \
    nasm \
    autoconf \
    automake \
    libtool
  ```

- 使用系统默认的 `gcc` 工具链；
- 安装路径：`$HOME/ffmpeg-min`；
- 上传 Artifact：`ffmpeg-minimal-linux`，路径为 `$HOME/ffmpeg-min/bin/ffmpeg`。

### 4.2 macOS（Apple Silicon / Intel）

#### macOS 15 Arm64（Apple Silicon）

- 使用 `macos-15` Runner，对应 macOS 15 Arm64；
- 安装依赖：

  ```bash
  brew update
  brew install \
    autoconf \
    automake \
    libtool \
    pkg-config \
    yasm \
    nasm
  ```

- 原生 arm64 构建，安装到 `$HOME/ffmpeg-min`；
- 上传 Artifact：`ffmpeg-minimal-macos-arm64`。

#### macOS 15 Intel

- 使用 `macos-15-intel` Runner，对应 macOS 15 Intel；
- 使用同样的 Homebrew 依赖安装脚本；
- 原生 x86_64 构建，安装到 `$HOME/ffmpeg-min`；
- 上传 Artifact：`ffmpeg-minimal-macos-x86_64`。

这样可以分别为 Apple Silicon 与 Intel Mac 提供最匹配的二进制，避免完全依赖 Rosetta。

### 4.3 Windows（windows-latest + MSYS2）

- Runner：`windows-latest`（当前为 Windows Server 2025）；
- 使用 `msys2/setup-msys2@v2` 提供类 Unix 构建环境；
- 安装的 MSYS2 包：

  ```text
  git
  make
  pkg-config
  mingw-w64-x86_64-gcc
  yasm
  nasm
  ```

- 在 MSYS2 shell 中执行 `./configure` 和 `make`：
  - `PREFIX="/c/ffmpeg-min"`；
  - 安装后可执行文件位于 `/c/ffmpeg-min/bin/ffmpeg.exe`（Windows 下为 `C:\ffmpeg-min\bin\ffmpeg.exe`）。
- 上传 Artifact：`ffmpeg-minimal-windows`。

---

## 5. 与三条典型命令的对应关系

假设构建得到的可执行文件命名为 `ffmpeg`（Linux/macOS）或 `ffmpeg.exe`（Windows），下列命令均可使用。

### 5.1 合并视频与音频，仅做流复制

```bash
ffmpeg -i video -i audio -c:v copy -c:a copy -shortest output
```

依赖能力：

372→- `file` 协议：读取 `video`、`audio` 文件并写出 `output`；
373→- 对应容器的 demuxer/muxer（例如 `mov`/`mp4`、`matroska`、`flac` 等）；
- 对应编码的 parser（比如 `h264`、`aac`）；
- 必要时的 bitstream filter（如从 TS → MP4 的 AAC）。

不需要能力：

- 编码器（不重新编码）；
- 解码器；
- 滤镜、缩放、重采样等。

### 5.2 只复制视频流

```bash
ffmpeg -i video -c:v copy -c:a copy output
```

（实际使用中建议显式写上 `-c:a copy`，否则音频可能会被重新编码）

依赖能力与 5.1 的视频部分相同：仅涉及容器级操作和 bitstream 操作。

### 5.3 只复制音频流

```bash
ffmpeg -i audio -c:a copy output
```

依赖能力：

401→- 音频容器 demuxer/muxer（`mp4/m4a`, `flac`, 以及作为输入的 `mp3`, `aac` 等）；
402→- 音频编码 parser（`aac`, `mp3`, `flac`, `vorbis`, `opus`）。

---

## 8. 当前精简版本支持的输出格式总结

当前配置的 ffmpeg **只支持生成以下四种文件格式**：

- `.mp4`
- `.m4a`（实质为 mp4 容器的纯音频）
- `.mkv`
- `.flac`

如果命令中指定其他容器（例如 `-f mp3`，或输出文件名以 `.mp3` 结尾），ffmpeg 会报错提示缺少对应的 muxer。这是为了让二进制体积尽可能小，专注于少数几种封装格式的流复制和重封装场景。 

---

## 6. 如何下载和使用构建结果

1. 将本仓库推送到 GitHub；
2. 在仓库页面打开 **Actions**；
3. 选择 `build-minimal-ffmpeg` 工作流，点击运行记录；
4. 在页面底部 **Artifacts** 中下载对应平台的压缩包：
   - `ffmpeg-minimal-linux`
   - `ffmpeg-minimal-macos-arm64`
   - `ffmpeg-minimal-macos-x86_64`
   - `ffmpeg-minimal-windows`
5. 解压后，将可执行文件加入 `PATH` 或直接在本地项目中引用。

---

## 7. 如何按需调整

如果后续需要支持更多格式或进一步缩减体积，可以直接修改 `.github/workflows/build-ffmpeg-minimal.yml` 中的 `./configure` 行：

- 增加某种封装格式：
  - 比如需要 `webm`：
    - 在 `--enable-demuxer` 和 `--enable-muxer` 增加 `webm`；
- 增加某种编码的解析：
  - 比如需要 `flac` 音频：
    - 在 `--enable-parser` 增加 `flac`；
- 如果未来需要做转码（非 copy）：
  - 需要显式启用对应的 `encoder` / `decoder` / `filter`，例如 `--enable-encoder=aac`、`--enable-filter=aresample` 等。

当前配置严格围绕“copy 封装 + 常见容器/编码的复用”设计，适合作为集成于应用或服务中的轻量级 FFmpeg 工具。 
