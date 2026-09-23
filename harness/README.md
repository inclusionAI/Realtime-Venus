# Realtime-Venus-Harness

**English** · [简体中文](README_ZH.md) · [Project](../README.md) · [Run the Demo](../demos/README.md) · [Paper](https://arxiv.org/pdf/2609.13814)

Harness captures `<delegate>...</delegate>` requests, runs background tasks, and returns results to the same conversation. It tracks task context, execution, progress, cancellation, continuation, and delivery.

## Architecture

<p align="center"><img src="../assets/paper-harness.svg" width="100%" alt="Paper Figure 7: Capture, Dispatch, and Return stages of Realtime-Venus-Harness with work-state tracking and playback acknowledgment" /><br /><sub>Figure 7 from the <a href="https://arxiv.org/pdf/2609.13814">paper</a>. Work tracking spans Capture, Dispatch, and Return.</sub></p>

### 1. Capture: a request with a stable context

The protocol gate assembles delegate spans across chunks, fixes the evidence cutoff at the opening tag, and registers the completed request with its originating session. Structured tasks can use `DelegateHarness.submit_delegate` or `submit_request` directly.

### 2. Dispatch: select and execute a capability

| Path | Purpose | Implementation |
| --- | --- | --- |
| **Multimodal / direct answer** | Answer using the request's available audio or visual evidence. | [`llm/delegate.py`](llm/delegate.py) |
| **General** | Run a task that requires tools or multiple execution steps; retain progress and generated artifacts. | [`agents/`](agents/), [`jobs/`](jobs/) |
| **Skill** | Execute a registered domain capability with a declared argument contract. | [`skills.py`](skills.py) |

**General mode** sends requests directly to the General executor. **Auto mode** asks the planner to choose a capability or handle progress queries and task continuation. The standalone `PlannerClient` defaults to Auto; the browser Demo selects General by default. Available tools and skills depend on the configured executor and registry.

### 3. Return: prepare a reply for this conversation

Harness prepares a spoken reply, checks delivery eligibility, and removes reserved protocol text. The serving host inserts `<backend>...</backend>` into the originating session at an eligible input boundary; Omni then generates speech.

## Frontend and backend integration

| Boundary | Included integration | How to extend it |
| --- | --- | --- |
| Conversational model | Omni via the Demo's `RemoteOmniServingPort` and HTTP model service. | Implement `VenusOmniServingPort`, or submit structured tasks through `DelegateHarness`. |
| Task execution | Codex through `CodexAgentProvider` and its app-server connection. | Implement `GeneralAgentPort` for another executor. |
| Routing, Polish and multimodal understanding | Codex by default; independently select official `GeminiBackend` / `GeminiDirect` in [Demo settings](../demos/README.md#official-gemini). | Implement `PlannerBackend` or the `DelegateBackend` contract. |
| Client playback | Browser playback in the Demo. | Implement `PlaybackPort` and acknowledge only completed playback. |
| Domain capabilities | `AgentSkill` and `SkillRegistry`. | Register a capability description, validated arguments, and executor. |

## Getting started

### Use Harness in your application

Install the package with Python **3.11 or 3.12**, from the repository root:

```bash
python -m pip install -e ./harness
```

Import the installed package as `harness`. To run the full system, follow the [Demo guide](../demos/README.md).

Supply a [`DelegateBackend`](core/backend.py) implementing `plan`, `execute`, `oralize`, and `aclose`:

```python
from harness.core.backend import DelegateBackend
from harness.core.harness import DelegateHarness

async def request_background_work(backend: DelegateBackend):
    harness = DelegateHarness(backend=backend)
    try:
        await harness.open_session("demo-session")
        work_id = await harness.submit_delegate(
            "demo-session", "Create a CSV of the numbers 1–100 and their squares."
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

For backend assembly, see [`factory.py`](factory.py) and [configuration](#configuration-file).

### Connect a streaming model

`VenusOmniAgentHarness` assembles the runtime; `VenusOmniServingHost` connects it to the tokenizer, model serving port, and playback client:

1. Open the host session and validate the model's protocol token IDs.
2. Submit timestamped media through `append_audio` and `append_video_frame`.
3. Use one `next_output` loop to consume raw model output in order and pass permitted speech to playback.
4. Call `begin_backend_turn` at eligible boundaries to admit queued private feedback.
5. Call `acknowledge_playback` only for chunks that have actually finished playing.
6. Close the host session and call `agent.aclose()` when the runtime owner exits.

The working implementation is in [`demos/server/session.py`](../demos/server/session.py). `DelegateParser` provides text parsing.

## Work lifecycle

```text
QUEUED → RUNNING → COMPLETED → DELIVERING → DELIVERED
             └→ FAILED
Cancellation: CANCELLING → CANCELLED
```

| State | Meaning |
| --- | --- |
| `QUEUED` | An accepted task is waiting for execution. |
| `RUNNING` | Routing, execution, or reply preparation is underway. |
| `COMPLETED` | The terminal result and prepared reply are available. |
| `DELIVERING` | Feedback is reserved for admission to the originating conversation. |
| `DELIVERED` | Delivery has completed; for speech-bearing replies, this includes generation completion and acknowledgment of the associated playback. |

Before feedback is returned, Harness checks its freshness and originating session. State definitions and delivery logic are in [`jobs/models.py`](jobs/models.py) and [`bridge/runtime.py`](bridge/runtime.py).

## Configuration file

Configuration and loading live in this package and do not depend on Demo, FastAPI or frontend model weights. The entire `harness/` directory can be published independently.

```bash
# From the Harness package root
python -m pip install -e .
```

Edit [`config.json`](config.json) directly and set your own `workspace`. Codex and role models retain the current defaults. Paths resolve relative to this Harness JSON. No personal workspace or API key is prefilled.

Configuration steps:

1. Set `workspace` to the task files directory. `codex.command` defaults to `["codex", "app-server"]`; replace its first item with the executable path if Codex is not on PATH, and complete Codex login on the server.
2. `routing.mode` defaults to `general`, sending delegates directly to Codex. Choose `auto` to enable routing and multimodal capability selection.
3. In `routing`, `responses` (Polish) and `multimodal`, independently set `provider` (`codex` or `gemini`), `model` and `timeout_s`. Codex `effort` defaults to `low`; Gemini does not use it. The General task executor is still configured under `codex`.
4. For Gemini, set `gemini.api_key` or the `GEMINI_API_KEY` / `GOOGLE_API_KEY` environment variable. `gemini.model` is the provider’s default model.
5. Adjust `codex.execution_timeout_s` for task execution and `feedback.queue_timeout_s` for queue admission. Routing and Polish call budgets use `routing.timeout_s` and `responses.timeout_s`; delivery/playback waits use `harness.result_management`.
6. Progress announcements are off by default. Enable `feedback.proactive_progress` and adjust `feedback.progress_interval_s` if desired.

After saving, reload the configuration in a standalone application; the web Demo reads these settings for its next session.

```python
from harness.settings import load_setup
from harness.factory import build_harness

setup = load_setup("config.json")
agent = build_harness(setup)
# Connect agent to your sessions; await agent.aclose() on shutdown
```

Configuration groups:

| Group | Controls |
| --- | --- |
| `codex` | Executor, model, effort, control/execution/interrupt timeouts. |
| `routing` | General / Auto, Codex / Gemini, routing model and timeout. |
| `responses` / `multimodal` | Independent Polish and multimodal providers, models and timeouts. |
| `gemini` | Gemini model and API key; environment keys are supported. |
| `feedback` | Progress opt-in (off by default), intervals, queue, skill and stage budgets. |
| `harness.delegate` | Request/oralization timeouts, result TTL and delegate limits. |
| `harness.result_management` | Silence windows, queue, frontend reaction and playback wait budgets. |
| `harness.buffer` / `runtime` / `workspace` | Context retention, concurrency and artifact storage. |

Fields ending in `_s` use seconds; `_ms` uses milliseconds. Values are passed to components without Demo overriding wait budgets. The configuration file lists supported fields; unknown fields fail validation.

The Demo references this file with `"harness": {"config": "harness/config.json"}` and its Settings UI writes back to it. Frontend voice, checkpoint and speaking length belong to the Demo file. Keep runtime settings, keys and task files out of source control; `save_setup` uses atomic writes and `0600` permissions. Legacy web files may contain `duplex`; move that Demo-only value to `model.length_penalty` before using a standalone Harness configuration.
