# YZ-B032 AI 边缘计算盒火警复核服务使用手册

本文档面向设备使用方和联调人员，说明如何启动、停止、检查和调用部署在 YZ-B032（BM1684X）上的 Qwen2-VL 火警复核服务。

## 1. 服务概览

| 项目 | 内容 |
| --- | --- |
| 设备 | YZ-B032 / BM1684X |
| 服务目录 | `/data/sophon-demo-new/sample/Qwen2-VL/python` |
| 启动文件 | `fire_api.py` |
| 监听地址 | `0.0.0.0:8080` |
| 应用协议 | HTTP |
| 正式心跳接口 | `GET /api/v1/health` |
| 火警复核接口 | `POST /api/v1/fire/analyze` |
| 模型 | `qwen2-vl-7b_int4_seq1536_1dev.bmodel` |
| 模型加载方式 | 仅加载设备本地模型和配置，不联网下载 |

服务接收一张 JPG、JPEG 或 PNG 图片以及对应事件元数据，在盒子本地完成视觉大模型推理，并返回是否存在起火现象及置信度等级。

## 2. 日常使用：开机后如何启动

### 2.1 登录设备

在能够访问盒子的计算机上登录设备。以下地址为当前直连/局域网示例，实际部署时以现场分配的地址为准。

```text
SSH 地址：192.168.150.1
用户名：linaro
初始密码：linaro
```

```bash
ssh linaro@192.168.150.1
```

出现密码提示后输入 `linaro`。Linux 终端输入密码时不会显示字符或星号，输入完成后按 Enter 即可。

### 2.2 检查是否已有服务实例

```bash
pgrep -af "python3.*fire_api.py"
```

如果已经显示 `python3 fire_api.py`，不要再次启动。重复加载模型可能耗尽 TPU 内存。

### 2.3 前台启动服务

首次验收或排查问题时建议前台运行，以便直接查看日志：

```bash
cd /data/sophon-demo-new/sample/Qwen2-VL/python
export LD_LIBRARY_PATH=/opt/sophon/libsophon-0.5.1/lib:$LD_LIBRARY_PATH
AI_TIMEOUT_SECONDS=15 python3 fire_api.py
```

出现以下信息表示 HTTP 服务已经监听 8080 端口：

```text
Application startup complete.
Uvicorn running on http://0.0.0.0:8080
```

模型加载需要一定时间。服务启动后，还应通过正式心跳接口确认 `status` 为 `READY`，不能只依据端口已监听判断模型可用。

### 2.4 检查服务状态

在盒子的另一个终端执行：

```bash
curl http://127.0.0.1:8080/api/v1/health
```

正常返回示例：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "status": "READY",
    "modelVersion": "qwen2-vl-7b-int4-seq1536-1dev",
    "uptimeSeconds": 126
  }
}
```

状态说明：

- `READY`：模型已加载，可以接收火警复核请求。
- `LOADING`：模型正在加载，应稍后重试。
- `ERROR`：模型加载失败，应查看启动日志。

`GET /health` 为本机调试接口；上游平台应使用正式接口 `GET /api/v1/health`。

## 3. 停止和重启服务

### 3.1 前台运行时停止

在运行服务的终端按 `Ctrl+C`。

### 3.2 查找并停止后台进程

```bash
pgrep -af "python3.*fire_api.py"
kill <PID>
```

将 `<PID>` 替换为上一条命令显示的进程号。停止后再次执行 `pgrep`，确认没有残留实例，再重新启动。

## 4. 火警复核接口

### 4.1 请求定义

```text
POST /api/v1/fire/analyze
Content-Type: multipart/form-data
```

表单字段：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `image` | File | 是 | JPG/JPEG/PNG；文件不超过 5 MB；分辨率不低于 640×480 |
| `metadata` | JSON String | 是 | 事件元数据字符串 |

`metadata` 示例：

```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440000",
  "cameraId": "PTS-0A0C112234B5",
  "eventTime": "2026-06-01 10:20:30"
}
```

字段约束：

- `eventId` 必须是合法 UUID。
- `cameraId` 不能为空。
- `eventTime` 格式必须为 `yyyy-MM-dd HH:mm:ss`。
- 图片扩展名必须与实际文件格式一致。

### 4.2 成功响应

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "isFire": true,
    "confidence": 90
  }
}
```

返回字段：

| 字段 | 说明 |
| --- | --- |
| `isFire` | `true` 表示模型判断存在起火现象，`false` 表示未发现起火现象 |
| `confidence` | 模型对当前“有/无”结论的自评置信等级，取值为 30、40……100 |

`confidence` 是工程化自评等级，不是经过统计校准的真实概率。业务平台应结合现场规则使用，不应将其直接解释为精确概率。

### 4.3 业务错误码

| HTTP 状态 | 业务码 | 含义 | 常见处理方式 |
| --- | ---: | --- | --- |
| 200 | 0 | 成功 | 读取 `data` |
| 504 | 1001 | AI 分析超时 | 稍后重试，并检查设备负载和超时配置 |
| 400 | 1002 | 参数错误 | 检查 metadata、图片大小和最低分辨率 |
| 400 | 1003 | 图片为空 | 重新上传有效图片 |
| 400 | 1004 | 图片格式错误或图片损坏 | 使用内容与扩展名一致的 JPG/JPEG/PNG |
| 503 | 2001 | AI 服务异常或等待队列已满 | 检查心跳和日志，稍后重试 |
| 500 | 5000 | 系统异常 | 保存 eventId 和发生时间，检查服务日志 |

调用方必须同时检查 HTTP 状态码和响应体中的业务 `code`。

## 5. 调用示例

### 5.1 Linux curl

```bash
curl -X POST "http://192.168.150.1:8080/api/v1/fire/analyze" \
  -F "image=@/path/to/image1.jpg;type=image/jpeg" \
  -F 'metadata={"eventId":"550e8400-e29b-41d4-a716-446655440000","cameraId":"PTS-0A0C112234B5","eventTime":"2026-06-01 10:20:30"}'
```

### 5.2 Windows PowerShell curl.exe

PowerShell 中建议先将 metadata 写入临时文件，避免 JSON 引号被命令行错误解析：

```powershell
$img = "C:\path\to\image1.jpg"
$metaFile = Join-Path $env:TEMP "fire_metadata.json"

'{"eventId":"550e8400-e29b-41d4-a716-446655440000","cameraId":"PTS-0A0C112234B5","eventTime":"2026-06-01 10:20:30"}' |
  Set-Content -Path $metaFile -Encoding ASCII

curl.exe -X POST "http://192.168.150.1:8080/api/v1/fire/analyze" `
  -F "image=@$img;type=image/jpeg" `
  -F "metadata=<$metaFile"
```

PowerShell 不能直接输入 `POST http://...`，应使用 `curl.exe -X POST`、Postman 或上游业务程序发送请求。

### 5.3 Postman

1. 新建请求并选择 `POST`。
2. 地址填写 `http://<盒子地址>:8080/api/v1/fire/analyze`。
3. 打开 **Body → form-data**。
4. 新增 `image`，类型选择 **File**，选择测试图片。
5. 新增 `metadata`，类型选择 **Text**，填写完整 JSON 字符串。
6. 不要手动填写 `Content-Type`；Postman 会自动生成带 boundary 的 `multipart/form-data` 请求头。
7. 单击 **Send**，检查 HTTP 状态码和响应体中的 `code`。

## 6. 内网与外网联调

局域网联调时直接使用盒子的局域网 IP 和 8080 端口。使用内网穿透时，调用方应使用穿透平台当前显示的完整外网地址，例如：

```text
http://<穿透域名>:<外网端口>/api/v1/health
http://<穿透域名>:<外网端口>/api/v1/fire/analyze
```

外网端口可能由平台动态分配，不能仅复制域名并遗漏端口。`ping` 不通不代表 HTTP 映射不可用，应直接访问正式心跳接口验证。

当前服务使用明文 HTTP，且应用本身没有 API Key 鉴权。若需要跨公网长期运行，应由网络或平台侧增加访问控制、来源限制以及 HTTPS 反向代理，不建议将 8080 端口无保护地直接暴露到公网。

## 7. 队列、并发和超时

为适配单颗 BM1684X 的模型内存占用，当前服务配置为：

- 同一时刻只执行 1 个 TPU 推理任务。
- 最多缓存 4 个等待任务。
- 即最多容纳“1 个正在推理 + 4 个等待”的请求。
- 容量已满后，新请求立即返回 HTTP 503、业务码 `2001`。
- 排队等待时间不计入 `AI_TIMEOUT_SECONDS`，该参数只限制任务真正开始后的推理时间。

不要通过增加 Uvicorn worker、启动多个 `fire_api.py` 进程或并行加载多个模型来提高吞吐量。Qwen2-VL 模型已占用大部分设备内存，多实例可能出现 `bm_alloc_gmem failed`、推理失败或连接被重置。

当前单张图片实测通常需要约 5～7 秒。建议：

- 盒子端使用 `AI_TIMEOUT_SECONDS=15`，为偶发抖动留出余量。
- 上游 HTTP 请求超时应覆盖排队时间，建议不少于 40 秒，并结合现场请求峰值调整。
- 上游仅在出现疑似火警事件时发送关键帧，不要将连续视频逐帧提交给该接口。
- 队列满或临时错误时采用有限次数、带退避的重试，避免立即高频重发。

## 8. 日志和运行状态

前台启动时，日志直接显示在当前终端。需要保留文件日志时可执行：

```bash
cd /data/sophon-demo-new/sample/Qwen2-VL/python
mkdir -p logs
export LD_LIBRARY_PATH=/opt/sophon/libsophon-0.5.1/lib:$LD_LIBRARY_PATH
AI_TIMEOUT_SECONDS=15 python3 fire_api.py 2>&1 | tee -a logs/fire_api.log
```

访问日志会记录来源 IP、来源临时端口、请求方法、路径和 HTTP 状态码。模型结果日志会记录 `isFire`、置信度以及截断后的原始模型输出。联调排障时应同时记录 eventId、请求时间、HTTP 状态和业务码。

检查模型进程和设备内存：

```bash
pgrep -af "python3.*fire_api.py"
bm-smi
```

## 9. 首次部署或环境恢复

已交付且能够正常启动的盒子无需重复执行本节。只有重装系统、更换设备或 Python 环境损坏时才需要恢复环境。

### 9.1 已验证的软件环境

```text
Python             3.8.2
libsophon          0.5.1 LTS SP4
sophon-ffmpeg      0.12.0
sophon-opencv      0.12.0
sophon-arm         3.10.7
numpy              1.24.4
```

API 依赖文件 `python/requirements_fire_api.txt` 只包含：

```text
fastapi==0.103.2
uvicorn==0.23.2
python-multipart==0.0.6
```

该文件不会安装 `hf-xet`，服务也不会联网下载 Hugging Face 模型。Qwen2-VL 基础运行环境还需要设备适配版 `sophon.sail`、PyTorch、TorchVision、Pillow、OpenCV、Transformers 和 SILK2 等依赖；恢复时应优先使用随交付包提供、与 Python 3.8/aarch64/SP4 匹配的离线 wheel。

验证主要模块：

```bash
python3 -c "from sophon import sail; import torch, torchvision, cv2, PIL, numpy; import SILK2.Tools.logger; print('runtime imports OK')"
```

### 9.2 必需的本地文件

以下文件或目录必须存在，且不应在设备端修改模型内容：

```text
/data/sophon-demo-new/sample/Qwen2-VL/
├── models/BM1684X/qwen2-vl-7b_int4_seq1536_1dev.bmodel
└── python/
    ├── fire_api.py
    ├── qwen2_vl.py
    ├── vision_process.py
    ├── qwen2vl_config/
    └── requirements_fire_api.txt
```

检查文件：

```bash
cd /data/sophon-demo-new/sample/Qwen2-VL
test -f models/BM1684X/qwen2-vl-7b_int4_seq1536_1dev.bmodel && echo "bmodel OK"
test -f python/fire_api.py && echo "fire_api.py OK"
test -f python/qwen2vl_config/config.json && echo "config OK"
```

### 9.3 动态库检查

```bash
ls -l /opt/sophon/libsophon-0.5.1/lib/libbmrt.so*
ls -l /opt/sophon/libsophon-0.5.1/lib/libyuv.so*
ldconfig -p | grep -E "libbmrt|libyuv"
```

当前设备环境曾使用以下兼容软链接：

```text
libbmrt.so.1.0 -> libbmrt.so.9.9
libyuv.so.0    -> libyuv.so
```

如果再次出现 `libbmrt.so.1.0` 或 `libyuv.so.0` 找不到，应先核对 SP4 安装路径和实际库版本，再由设备管理员恢复动态库配置；不要盲目链接到不存在或版本不匹配的文件。

## 10. 常见问题排查

| 现象 | 原因与处理 |
| --- | --- |
| `ModuleNotFoundError: sophon` | 未安装与 SP4、Python 3.8 和 aarch64 匹配的 `sophon-arm` wheel |
| `libbmrt.so.1.0: cannot open shared object file` | 动态库搜索路径或软链接未生效；检查 `LD_LIBRARY_PATH`、`ldconfig` 和实际文件 |
| `ModuleNotFoundError: torch/cv2/SILK2` | Qwen2-VL 运行依赖不完整；使用交付的兼容 wheel 恢复，避免在线安装到不匹配版本 |
| 心跳返回 `ERROR` | 查看启动日志中的 `modelError` 或异常堆栈，检查模型及配置文件 |
| HTTP 404 `Not Found` | 访问了根路径 `/`；应使用 `/api/v1/health` 或 `/api/v1/fire/analyze` |
| HTTP 405 `Method Not Allowed` | 对分析接口使用了 GET；该接口必须使用 POST multipart/form-data |
| HTTP 503、`AI service busy` | 当前已有 1 个任务推理且 4 个任务等待；调用方应退避后重试 |
| HTTP 504、业务码 1001 | 单次推理超过盒子端限制；检查负载，必要时调整 `AI_TIMEOUT_SECONDS` |
| `bm_alloc_gmem failed` | TPU/VPU/VPP 内存紧张；确认没有重复模型进程，停止多余实例并检查 `bm-smi` |
| `Recv failure: Connection was reset` | 服务进程异常退出或底层设备内存失败；查看盒子日志和进程状态，必要时重启服务 |
| 内网穿透域名可访问但 POST 失败 | 检查是否遗漏动态外网端口、映射是否在线、上传大小限制和平台超时设置 |

提交前请再次执行 `git status`，不要使用 `git add .` 将模型、数据集、缓存或无关修改一并加入提交。


