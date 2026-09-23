# Realtime-Venus Demo

[English](README.md) · **简体中文** · [项目总览](../README_ZH.md) · [Harness](../harness/README_ZH.md)

基于 Realtime-Venus-Omni 或 Realtime-Venus-Audio，以及 Codex 任务后端的网页 Demo。Audio 模式提供麦克风与音频上传；Omni 模式提供摄像头加麦克风与视频上传。

## 两个 loop，同一段对话

<p align="center"><img src="../assets/paper-loops.svg" width="100%" alt="论文图 3：实时交互 loop 与异步能力执行 loop" /><br /><sub>论文<a href="https://arxiv.org/pdf/2609.13814">图 3</a>。Demo 启动时选择 Omni 或 Audio 作为交互前端。</sub></p>

**Interaction loop** 由所选前端模型处理流式媒体与语音；**Capability loop** 由 Harness 异步执行 `<delegate>...</delegate>` 请求，通过 `<backend>...</backend>` 回传结果，再由前端模型播报。

## 服务组成

| 组件 | 职责 | 实现入口 |
| --- | --- | --- |
| 浏览器 | 设备采集或视频上传、流式输入、语音播放、任务进度和产物下载。 | [`static/app.js`](static/app.js) |
| 网页服务 · 8032 | 接收 HTTP/WebSocket 请求，管理浏览器会话，组装模型与 Harness 连接。 | [`server/app.py`](server/app.py)、[`server/resources.py`](server/resources.py) |
| 会话 host | 排序媒体输入、消费模型输出、安排私有反馈、转交播放回执。 | [`server/session.py`](server/session.py)、[`VenusOmniServingHost`](../harness/bridge/host.py) |
| 模型服务 · 8031 | 加载权重、维护模型状态、执行流式推理并生成原生语音。 | [`model/server.py`](model/server.py)、[`model/adapter.py`](model/adapter.py) |
| Harness + Codex | 嵌入网页服务，执行任务、跟踪进度并回传结果。 | [`server/resources.py`](server/resources.py)、[Harness](../harness/README_ZH.md#架构) |
| 启动与配置 | 环境安装、进程管理、配置持久化。 | [`launcher/`](launcher/)、[`install.py`](install.py)、[`settings.py`](settings.py) |

<p align="center"><img src="../assets/experience-zh.png" width="100%" alt="Realtime-Venus 网页体验界面" /><br /><sub>浏览器统一承载实时媒体、任务进度和结果下载。</sub></p>

## 准备

使用配备 NVIDIA CUDA GPU 和可用驱动的 Linux 服务器。按[主 README](../README_ZH.md)下载所选模式的完整权重，保留模型代码、tokenizer、参考音频及语音资产。默认下载目录为 `Realtime-Venus-Omni/` 或 `Realtime-Venus-Audio/`，将 `model.path` 指向对应目录。

Codex 后端需要网络和可用登录。安装器复用已有 Codex 或补齐安装，首次启动时按提示登录。

<details>
<summary>权重目录示例</summary>

```text
Realtime-Venus-Omni/  # Or Realtime-Venus-Audio/
├── config.json
├── tokenizer.json / tokenizer_config.json
├── *.py                       # 随权重发布的模型代码
├── model.safetensors          # 或全部分片及其索引 JSON
└── assets/
    ├── HT_ref_audio.wav
    └── token2wav/
        ├── flow.yaml / flow.pt / hift.pt
        ├── campplus.onnx
        └── speech_tokenizer_v2_25hz.onnx
```

已有的 `model_weight/model_weight/` 嵌套布局也可识别。

</details>

## 启动

直接编辑仓库根目录的 [`config.json`](../config.json)，配置前端模型，并通过 `harness.config` 指向已配置好的 Harness 文件。具体步骤见下方[两份配置文件](#两份配置文件)，Harness 参数详见 [Harness README](../harness/README_ZH.md#配置文件)。

首次使用先在仓库根目录运行 `bash install.sh`，完成配置后启动：

```bash
bash start.sh --config config.json
```

## 在本地电脑打开

在**本地电脑的终端**中保持以下隧道运行，将 `user@server` 换成你的 SSH 登录地址：

```bash
ssh -N -L 8032:127.0.0.1:8032 user@server
```

打开 **[http://localhost:8032](http://localhost:8032)**，开始会话。

| 输入模式 | 如何体验 |
| --- | --- |
| **语音** | 允许麦克风访问，通过本地扬声器收听。 |
| **摄像头** | 同时共享摄像头和麦克风，网页显示实时预览。 |
| **视频** | 选择视频模式并上传本地文件，上限 **200 MB**。视频音轨与采样画面会流式送入模型。 |

侧边任务面板支持查看进度、取消任务和下载产物。

服务同一时间支持**一个活跃会话**，默认只监听回环地址。SSH 转发提供浏览器信任的 `localhost` 来源；直接通过服务器 IP 使用摄像头／麦克风需要 HTTPS。若服务器网页端口改为 `9032`，转发参数相应改为 `-L 8032:127.0.0.1:9032`。

## 配置

### 两份配置文件

**Demo 配置**（根目录 `config.json`）只配置前端模型、网页服务和 Harness 配置路径：

```json
{
  "model": {
    "type": "video",
    "path": "Realtime-Venus-Omni",
    "reference_audio": "",
    "port": 8031,
    "memory_minutes": 40,
    "length_penalty": 0.8,
    "timeout_s": 180
  },
  "harness": {
    "config": "harness/config.json"
  },
  "web": {
    "host": "127.0.0.1",
    "port": 8032
  },
  "startup_timeout": 600
}
```

配置步骤：

1. 选择 `model.type`：`video`（也接受 `omni`）提供摄像头与麦克风、上传视频；`audio` 提供麦克风、上传音频。
2. 将 `model.path` 填为对应的完整权重目录，或包含两种权重子目录的 Hugging Face 下载目录。两种模式使用独立权重。
3. `model.reference_audio` 填参考音色文件路径；留空使用权重自带的默认音色。
4. 将 `harness.config` 指向自己的 Harness 配置，并按 Harness README 填写任务目录、组件模型和等待时长。默认指向 `harness/config.json`。
5. 按需调整下列运行参数，保存后启动。

| 字段 | 默认值 | 作用 |
| --- | --- | --- |
| `model.port` | `8031` | 模型 API 端口。 |
| `model.memory_minutes` | `40` | Omni 视频记忆时长（分钟）；Audio 不使用。 |
| `model.length_penalty` | `0.8` | 发言长度参数，范围 0.1–5；越小越倾向于提前结束发言。 |
| `model.timeout_s` | `180` | 模型服务请求超时（秒）。 |
| `web.host` / `web.port` | `127.0.0.1` / `8032` | 网页监听地址与端口。 |
| `startup_timeout` | `600` | 每个服务的启动等待时间（秒）。 |

切换模式时，先运行 `bash start.sh --stop`，修改同一份配置中的 `model.type`、`model.path` 和参考音频，然后重新运行 `bash start.sh --config config.json`。

**Harness 配置**（`harness/config.json`）直接编辑该独立包内的 [配置文件](../harness/config.json)；它包含 Codex、路由、Polish（`responses`）、multimodal、Gemini、任务目录、反馈间隔、执行／排队／结果等待时长及各组件预算。可脱离 Demo 加载；字段说明和独立使用方法见 [Harness 文档](../harness/README_ZH.md#配置文件)。

所有配置文件中的相对路径均以各自文件所在目录为准；`--config` 及命令行／环境变量路径以执行命令时的目录为准。`model.reference_audio` 留空使用所选权重的 `assets/HT_ref_audio.wav`；`model.memory_minutes` 仅影响 Omni 视频记忆，`model.timeout_s` 控制模型服务请求超时，`startup_timeout` 是每个服务的启动等待秒数。配置文件不含权重。

优先级为 **命令行 → 环境变量 → 配置文件 → 默认值**。`--harness-config` 或 `HARNESS_CONFIG` 可以覆盖引用；模型相关变量仍支持 `VENUS_MODEL_TYPE`、`MODEL_PATH`、`AUDIO_MODEL_PATH` / `OMNI_MODEL_PATH`、`REF_AUDIO`、`VENUS_MODEL_PORT`、`VENUS_MEMORY_MINUTES`，网页支持 `VENUS_WEB_HOST`、`VENUS_WEB_PORT`，启动等待支持 `VENUS_STARTUP_TIMEOUT`。

不传 `--config` 时继续读取仓库根目录 `config.json`；旧的扁平格式及 `runtime/harness.json` 仍可使用。新格式中必须填写 `harness.config`，引用的文件必须存在。错别字、非法数值或缺失资产会明确报错；`--stop` / `--status` 不依赖配置文件可读。切换模型或服务端口后需要重启。

### 网页设置

任务工作目录默认留空，用户需在 Settings 中填写服务器上的任务目录后才能开始体验；相对路径以配置文件所在目录为基准。保存后再次打开 Settings 会显示用户自己配置的路径。开源发布时不要打包 `runtime/` 或任务目录，它们包含本地配置、密钥、日志及任务文件。设置未完成时，选择语音、摄像头或视频会提示先完成 Settings；只检查当前选择的提供方所需配置。

开始对话前，网页和服务端 WebSocket 入口都会通过 `codex login status` 检查服务器上的 Codex 登录状态。Settings 会显示结果，并提供登录后的重新检查按钮。未登录、检查失败或超过 5 秒超时都会阻止新会话，仍可保存配置。该检查确认 CLI 本地登录状态，不代表额度或远端服务可用性检查。

主动播报任务进度默认关闭。在网页设置的“你的对话”中选择“开启：我同意接收语音进度反馈”，保存并开启新对话后生效；有新进展时才用一句话播报。关闭不影响任务执行、完成结果或主动询问进度。对应配置为 `feedback.proactive_progress`（布尔值，默认 `false`），保存在 `harness.config` 指向的文件；当前设置由此服务的用户共享。

请先结束当前对话再保存。任务参数写回 `harness.config` 指向的 Harness 文件，发言长度写回 Demo 配置的 `model.length_penalty`，下一次会话生效。未在网页暴露的 Harness 字段会保留。

| 设置项 | 默认值 | 作用 |
| --- | --- | --- |
| **任务模式** | General | 每个委托直接交给执行器，不调用路由模型。选择 Auto 后启用自动能力选择及任务续接。 |
| **任务执行** | 服务器模型、`low` 强度 | 控制后台任务执行器。 |
| **Auto 路由** | 任务模型、`low`、30 秒 | 仅在 Auto 模式下使用。路由超时会终止该请求，不启动执行器。 |
| **Polish 进度／结果整理** | Codex、任务模型、`low` | 可独立选择官方 Gemini；仍最多三句话。 |
| **Multimodal** | Codex、任务模型、`low` | Auto 路由选择该能力时负责直接问答和多模态理解，可独立选择官方 Gemini。 |
| **发言长度** | `length_penalty = 0.8` | 小于 1 更倾向提前结束发言；1 保持原行为；大于 1 更倾向长发言。范围为 0.1–5。 |

Codex 的模型名留空时，依次使用任务模型、服务器默认模型；Gemini 留空时使用下方的默认 Gemini 模型（初始为 `gemini-flash-latest`，可填写固定版本）。推理强度仅适用于 Codex，Gemini 使用其模型默认思考配置。旧配置中共用的直接问答参数会迁移到独立的 Multimodal 设置。

#### 使用官方 Gemini

1. 在网页“设置 → 任务连接 → 官方 Gemini 连接”填写 Google AI Studio API Key，或在启动服务前设置 `GEMINI_API_KEY` / `GOOGLE_API_KEY` 环境变量。
2. 分别为 Polish、Auto 路由、Multimodal 选择 Codex 或 Gemini，按需设置各自模型；选择 Gemini 后，该环节的文本或媒体将发送到 Google 官方 API。
3. 保存并开始新对话。默认任务模式仍为 General、所有提供方仍为 Codex；路由和 Multimodal 只有在 Auto 路径中才会实际使用。General 工具执行器继续使用 Codex，实时 Omni 模型不变。

密钥输入框留空表示保留已有密钥。网页配置接口只返回“已配置”状态；通过网页保存的密钥存于服务器权限为 `0600` 的 `harness.config` 指向的文件，不要提交到代码库。清除已存密钥时将文件中的 `gemini.api_key` 置空，同时检查服务器环境变量。设置由该服务的用户共享。

实现固定调用 Google 的 [GenerateContent 官方接口](https://ai.google.dev/api/generate-content)，不接受自定义中转地址，不自动切换到其他提供方。Gemini 多模态输入使用委托时冻结的音频、视频片段或图像；内联请求超过 20 MB 会明确报错，需要缩短媒体窗口。实际可用模型、配额和地区由 API Key 所属项目决定。

## 服务管理

```bash
bash start.sh --config config.json --check --no-login  # 当前仓库服务停止时，检查启动条件
bash start.sh --config config.json --detach           # 后台启动，待服务就绪后返回
bash start.sh --status
bash start.sh --stop
```

预检查和启动使用同一个 `--config` 文件。每个仓库只管理一套运行中的服务，切换配置前先停止旧服务。`--no-login` 会直接报告未登录状态，不发起登录流程。前台运行时，Ctrl+C 会停止两个服务。

| 运行路径 | 内容 |
| --- | --- |
| `runtime/logs/model.log` | 模型加载与推理。 |
| `runtime/logs/web.log` | 网页连接、上传与会话错误。 |
| `runtime/logs/stack.log` | 后台启动与进程管理。 |
| `harness.config` 指向的文件 | 模型／任务偏好及工作目录配置。 |
| `runtime/workspace/` | 任务工作区示例（需自行配置）。生成文件在会话结束后仍保留；网页下载链接只在对应会话内有效。 |
