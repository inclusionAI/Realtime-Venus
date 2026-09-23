# Realtime-Venus-Harness

[English](README.md) · **简体中文** · [项目总览](../README_ZH.md) · [启动 Demo](../demos/README_ZH.md) · [论文](https://arxiv.org/pdf/2609.13814)

Harness 接收 `<delegate>...</delegate>` 请求，异步执行任务并将结果送回原会话，管理任务上下文、执行、进度、取消、续接与交付。

## 架构

<p align="center"><img src="../assets/paper-harness.svg" width="100%" alt="论文图 7：Harness 的 Capture、Dispatch、Return 三阶段，贯穿其中的任务状态与播放确认" /><br /><sub>论文<a href="https://arxiv.org/pdf/2609.13814">图 7</a>：任务跟踪贯穿 Capture、Dispatch 和 Return。</sub></p>

### 1. Capture：捕获请求与稳定的上下文

协议入口跨 chunk 组装 delegate 请求，在开始标记处固定上下文边界，请求完整后连同来源会话一起注册。结构化任务可直接调用 `DelegateHarness.submit_delegate` 或 `submit_request`。

### 2. Dispatch：选择能力并执行

| 路径 | 作用 | 实现入口 |
| --- | --- | --- |
| **Multimodal / 直接问答** | 使用请求对应的音频或视觉证据完成理解与问答。 | [`llm/delegate.py`](llm/delegate.py) |
| **General** | 执行需要工具或多个步骤的任务，保留进度与生成产物。 | [`agents/`](agents/)、[`jobs/`](jobs/) |
| **Skill** | 调用已注册的领域能力，按声明的参数契约执行。 | [`skills.py`](skills.py) |

**General 模式**直接将任务交给 General 执行器；**Auto 模式**由 planner 选择能力，或处理进度查询与任务续接。独立使用 `PlannerClient` 时默认为 Auto，网页 Demo 默认为 General。可用工具与技能取决于执行器配置和能力注册。

### 3. Return：为当前对话准备回复

Harness 整理口头回复，检查交付条件并移除保留协议文本。Serving host 在允许的输入边界将 `<backend>...</backend>` 插入原会话，由 Omni 生成语音。

## 模型与后端接入

| 边界 | 当前提供的集成 | 扩展方式 |
| --- | --- | --- |
| 对话模型 | Demo 通过 `RemoteOmniServingPort` 和 HTTP 模型服务连接 Omni。 | 实现 `VenusOmniServingPort`，或通过 `DelegateHarness` 提交结构化任务。 |
| 任务执行 | `CodexAgentProvider` 及其 app-server 连接。 | 为其他执行器实现 `GeneralAgentPort`。 |
| 路由、回复整理与多模态理解 | 默认 Codex；网页可分别选择官方 `GeminiBackend` / `GeminiDirect`，详见 [Demo 配置](../demos/README_ZH.md#使用官方-gemini)。 | 实现 `PlannerBackend` 或 `DelegateBackend` 契约。 |
| 客户端播放 | Demo 中的浏览器播放器。 | 实现 `PlaybackPort`，仅确认已完成的播放。 |
| 领域能力 | `AgentSkill` 和 `SkillRegistry`。 | 注册能力描述、参数校验与执行器。 |

## 开始使用

### 在自己的应用中使用

使用 Python **3.11 或 3.12**，在仓库根目录安装：

```bash
python -m pip install -e ./harness
```

Python 导入名为 `harness`。完整系统的启动方法见 [Demo 指南](../demos/README_ZH.md)。

传入实现了 [`DelegateBackend`](core/backend.py) 的后端，提供 `plan`、`execute`、`oralize` 和 `aclose`：

```python
from harness.core.backend import DelegateBackend
from harness.core.harness import DelegateHarness

async def request_background_work(backend: DelegateBackend):
    harness = DelegateHarness(backend=backend)
    try:
        await harness.open_session("demo-session")
        work_id = await harness.submit_delegate(
            "demo-session", "生成一个包含 1–100 及其平方的 CSV 文件。"
        )
        while True:
            result = await harness.next_delegate_result(
                "demo-session", timeout_s=180
            )
            if result.work_id == work_id and result.status != "pending":
                return work_id, result
    finally:
        await harness.aclose()
```

Codex 后端的组装见 [`demos/server/resources.py`](../demos/server/resources.py)。

### 接入流式模型

`VenusOmniAgentHarness` 组装任务运行时，`VenusOmniServingHost` 连接 tokenizer、模型 serving port 与播放器：

1. 打开 host 会话并校验模型协议 token ID。
2. 通过 `append_audio` 和 `append_video_frame` 输入带时间戳的媒体。
3. 用单一 `next_output` 循环按顺序消费模型原始输出，将允许播放的语音交给播放器。
4. 在符合条件的边界调用 `begin_backend_turn`，注入排队中的私有反馈。
5. 仅对实际播放完成的 chunk 调用 `acknowledge_playback`。
6. 关闭 host 会话；运行时拥有者退出时调用 `agent.aclose()`。

可运行的完整链路见 [`demos/server/session.py`](../demos/server/session.py)。`DelegateParser` 提供纯文本解析。

## 任务生命周期

```text
QUEUED → RUNNING → COMPLETED → DELIVERING → DELIVERED
             └→ FAILED
取消路径：CANCELLING → CANCELLED
```

| 状态 | 含义 |
| --- | --- |
| `QUEUED` | 请求已接受，等待执行。 |
| `RUNNING` | 正在路由、执行任务或准备回复。 |
| `COMPLETED` | 终态结果和准备好的回复已可用。 |
| `DELIVERING` | 反馈已为送回原会话而保留。 |
| `DELIVERED` | 交付完成；带语音的回复需要生成结束，以及关联语音的播放确认。 |

反馈回传前检查时效与会话归属。状态定义与交付逻辑见 [`jobs/models.py`](jobs/models.py) 和 [`bridge/runtime.py`](bridge/runtime.py)。

## 配置文件

Harness 的配置文件和加载代码位于本包内，不依赖 Demo、FastAPI 或前端模型权重。将整个 `harness/` 目录作为独立项目发布即可。

```bash
# 在 Harness 包根目录
python -m pip install -e .
```

直接编辑 [`config.json`](config.json)，填写自己的 `workspace`；Codex 与各角色模型配置采用现有默认值。路径相对于该 Harness JSON 所在目录。默认配置不预填个人工作区或 API Key。

配置方法：

1. `workspace` 填任务文件的工作目录。`codex.command` 默认为 `["codex", "app-server"]`；若 Codex 不在 PATH 中，将第一项改为可执行文件的路径，并在服务器完成 Codex 登录。
2. `routing.mode` 默认为 `general`，委托直接交给 Codex；设置为 `auto` 才启用路由和 multimodal 能力选择。
3. 在 `routing`、`responses`（Polish）、`multimodal` 中分别设置 `provider`（`codex` 或 `gemini`）、`model` 和 `timeout_s`。Codex 的 `effort` 默认 `low`；Gemini 不使用该参数。General 任务执行器仍由 `codex` 配置。
4. 使用 Gemini 时填写 `gemini.api_key`，或设置 `GEMINI_API_KEY` / `GOOGLE_API_KEY` 环境变量；`gemini.model` 为该 provider 的默认模型。
5. 按任务时长调整 `codex.execution_timeout_s`；排队等待调整 `feedback.queue_timeout_s`；路由与 Polish 调用超时分别调整 `routing.timeout_s` 和 `responses.timeout_s`。结果等待与播放反馈时长见 `harness.result_management`。
6. 进度播报默认关闭；需要时将 `feedback.proactive_progress` 设为 `true`，并用 `feedback.progress_interval_s` 调整检查间隔。

保存后，独立应用重新加载配置；网页 Demo 在下一次会话读取这些参数。

```python
from harness.settings import load_setup
from harness.factory import build_harness

setup = load_setup("config.json")
agent = build_harness(setup)
# 将 agent 接入你的会话；生命周期结束时 await agent.aclose()
```

配置分组：

| 分组 | 作用 |
| --- | --- |
| `codex` | 执行器、模型、推理强度、控制／执行／中断超时。 |
| `routing` | General / Auto、Codex / Gemini、路由模型与超时。 |
| `responses` / `multimodal` | Polish 与多模态各自的 provider、模型和调用超时。 |
| `gemini` | Gemini 模型、API Key；也支持环境变量。 |
| `feedback` | 进度播报开关（默认关闭）、间隔、排队、技能与阶段超时。 |
| `harness.delegate` | 请求、结果有效期、口语化超时和 delegate 边界。 |
| `harness.result_management` | 静音窗口、排队等待、前端反应和播放等待时长。 |
| `harness.buffer` / `runtime` / `workspace` | 上下文缓存、并发限制、产物目录。 |

带 `_s` 的字段单位为秒，`_ms` 为毫秒。参数从文件加载后直接传入组件，Demo 不再用固定值覆盖等待时间。配置文件列出全部受支持字段，未知字段会报错。

网页 Demo 在自己的配置中通过 `"harness": {"config": "harness/config.json"}` 引用本文件；网页 Settings 写回同一个文件。前端音色、模型权重和发言长度属于 Demo 配置。运行时设置、密钥与任务文件不要提交；`save_setup` 使用原子写入和 `0600` 权限。旧网页配置中的 `duplex` 是 Demo 专属字段，迁移到独立 Harness 配置前应移到 Demo 的 `model.length_penalty`。
