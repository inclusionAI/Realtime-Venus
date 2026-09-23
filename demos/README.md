# Realtime-Venus Demo

**English** · [简体中文](README_ZH.md) · [Project overview](../README.md) · [Harness](../harness/README.md)

Browser demo with Realtime-Venus-Omni or Realtime-Venus-Audio and a Codex-backed Harness. Choose Audio for microphone/audio upload, or Omni for camera with microphone/video upload.

## Two loops, one conversation

<p align="center"><img src="../assets/paper-loops.svg" width="100%" alt="Paper Figure 3: the interaction loop and the asynchronous capability loop" /><br /><sub>Figure 3 from the <a href="https://arxiv.org/pdf/2609.13814">paper</a>. The demo selects an Omni or Audio frontend at launch.</sub></p>

The selected frontend handles streaming media and speech in the **interaction loop**. In the **capability loop**, Harness executes `<delegate>...</delegate>` requests asynchronously and returns results through `<backend>...</backend>` for the frontend to speak.

## Service layout

| Component | Responsibility | Implementation |
| --- | --- | --- |
| Browser | Capture devices or upload audio/video, stream input, play speech, show task progress and downloads. | [`static/app.js`](static/app.js) |
| Web service · 8032 | Accept HTTP/WebSocket requests, own the browser session, and assemble model and Harness connections. | [`server/app.py`](server/app.py), [`server/resources.py`](server/resources.py) |
| Session host | Order media input, consume model output, schedule private feedback, and relay playback acknowledgments. | [`server/session.py`](server/session.py), [`VenusOmniServingHost`](../harness/bridge/host.py) |
| Model service · 8031 | Load the checkpoint, retain model state, run streaming inference, and generate native speech. | [`model/server.py`](model/server.py), [`model/adapter.py`](model/adapter.py) |
| Harness + Codex | Embedded task execution, progress, and result delivery. | [`server/resources.py`](server/resources.py), [Harness](../harness/README.md#architecture) |
| Launcher and settings | Installation, process management, and saved configuration. | [`launcher/`](launcher/), [`install.py`](install.py), [`settings.py`](settings.py) |

<p align="center"><img src="../assets/experience-en.png" width="100%" alt="Realtime-Venus browser demo" /><br /><sub>The browser is the entry point for live media, task progress, and result downloads.</sub></p>

## Prepare

Use a Linux server with an NVIDIA CUDA GPU and a working driver. Download the complete checkpoint for your mode following the [model download guide](../README.md#1-installation), retaining its model code, tokenizer, reference audio, and speech assets. The default downloader places it in `Realtime-Venus-Omni/` or `Realtime-Venus-Audio/`. Set `model.path` accordingly.

The Codex backend requires network access and a usable login. The installer reuses an existing Codex installation or installs it; first startup guides login when needed.

<details>
<summary>Expected checkpoint layout</summary>

```text
Realtime-Venus-Omni/  # Or Realtime-Venus-Audio/
├── config.json
├── tokenizer.json / tokenizer_config.json
├── *.py                       # Checkpoint model code
├── model.safetensors          # Or all shards and their index JSON
└── assets/
    ├── HT_ref_audio.wav
    └── token2wav/
        ├── flow.yaml / flow.pt / hift.pt
        ├── campplus.onnx
        └── speech_tokenizer_v2_25hz.onnx
```

A nested `model_weight/model_weight/` layout is also recognized.

</details>

## Start

Edit root [`config.json`](../config.json) to configure the frontend model and point `harness.config` to your configured Harness file. Follow [Two configuration files](#two-configuration-files) below and the [Harness README](../harness/README.md#configuration-file).

Run `bash install.sh` once from the repository root, then launch after configuration:

```bash
bash start.sh --config config.json
```

## Open on your computer

In a terminal **on your local computer**, keep this tunnel open, substituting your SSH login:

```bash
ssh -N -L 8032:127.0.0.1:8032 user@server
```

Open **[http://localhost:8032](http://localhost:8032)** and start a conversation.

| Input mode | What to do |
| --- | --- |
| **Audio · microphone** | Allow microphone access and listen through your local speakers. |
| **Audio · audio upload** | Upload WAV, MP3, M4A, FLAC, OGG or other audio, up to **200 MB**. |
| **Omni · camera** | Share the camera and microphone together; the page shows a live preview. |
| **Omni · video upload** | Choose Video and upload a local file, up to **200 MB**. Its soundtrack and sampled frames are streamed to the model. |

The side panel provides task progress, cancellation, and artifact downloads.

The service supports **one active conversation**. It listens on loopback by default; SSH forwarding provides a browser-trusted `localhost` origin. Direct access through a server IP requires HTTPS for camera/microphone permissions. If the server web port is changed to `9032`, forward with `-L 8032:127.0.0.1:9032` instead.

## Configuration

### Two configuration files

**Demo configuration** (root `config.json`) contains frontend model settings, the web service, and a reference to the Harness file:

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

Configuration steps:

1. Choose `model.type`: `video` (also accepts `omni`) offers camera with microphone and video upload; `audio` offers microphone and audio upload.
2. Set `model.path` to the complete checkpoint directory, or the downloaded Hugging Face root containing both variants. The modes load separate checkpoints.
3. Set `model.reference_audio` to a reference voice file, or leave it empty for the checkpoint’s default voice.
4. Point `harness.config` to your Harness configuration and follow its README to set workspace, component models and timeouts. The default is `harness/config.json`.
5. Adjust the runtime options below as needed, save and start.

| Field | Default | Purpose |
| --- | --- | --- |
| `model.port` | `8031` | Model API port. |
| `model.memory_minutes` | `40` | Omni video memory in minutes; unused by Audio. |
| `model.length_penalty` | `0.8` | Speaking length, range 0.1–5; lower values encourage earlier turn endings. |
| `model.timeout_s` | `180` | Model service request timeout in seconds. |
| `web.host` / `web.port` | `127.0.0.1` / `8032` | Web bind address and port. |
| `startup_timeout` | `600` | Startup wait per service in seconds. |

To switch modes, run `bash start.sh --stop`, edit `model.type`, `model.path` and reference audio in the same file, then run `bash start.sh --config config.json` again.

**Harness configuration** (`harness/config.json`) is edited directly in the standalone package’s [configuration file](../harness/config.json). It controls Codex, routing, Polish (`responses`), multimodal, Gemini, workspace, progress feedback, execution/queue/delivery timeouts and component budgets. See the [Harness guide](../harness/README.md#configuration-file) for fields and standalone loading.

Relative paths inside each JSON resolve from that file’s own directory; CLI/environment paths and `--config` resolve from the caller’s directory. Empty `model.reference_audio` selects the checkpoint’s `assets/HT_ref_audio.wav`. `model.memory_minutes` applies only to Omni video memory; `model.timeout_s` controls model service request timeout; `startup_timeout` is the per-service startup wait. Config files contain no weights.

Precedence: **CLI → environment → config → defaults**. `--harness-config` or `HARNESS_CONFIG` overrides the reference. Supported model environment variables: `VENUS_MODEL_TYPE`, `MODEL_PATH`, `AUDIO_MODEL_PATH` / `OMNI_MODEL_PATH`, `REF_AUDIO`, `VENUS_MODEL_PORT`, `VENUS_MEMORY_MINUTES`. Web variables: `VENUS_WEB_HOST`, `VENUS_WEB_PORT`; startup wait: `VENUS_STARTUP_TIMEOUT`.

Without `--config`, the launcher reads root `config.json` if present. Legacy flat deployment files and `runtime/harness.json` remain supported. The new format requires `harness.config` to reference an existing file. Unknown fields, invalid values and missing assets fail clearly. `--stop` / `--status` do not depend on readable configuration. Restart after changing the model or service ports.

### Browser settings

Task workspace starts empty. In Settings, enter a server directory for task files before starting a conversation; relative paths are resolved from the settings file’s directory. Saved user choices are shown when reopening Settings. Exclude `runtime/` and task directories from source releases because they contain local settings, keys, logs and task files. If configuration is incomplete, voice, camera and video entry points prompt the user to finish Settings, validating the selected providers only.

Before starting a conversation, the browser and WebSocket admission both verify the server’s Codex login with `codex login status`. Settings shows the result and offers a recheck button after server login. Missing credentials, check failures, or a 5-second timeout block new conversations; configuration can still be saved. This checks local CLI login status, not quota or remote service availability.

Automatic spoken task progress is off by default. In Settings → Your conversation, explicitly enable “Speak task progress automatically”, save, and start a new conversation. New progress is spoken in one short sentence. Turning it off does not disable execution, final results, or progress you ask for. The boolean `feedback.proactive_progress` defaults to `false` in the file referenced by `harness.config`; this setting is shared by users of the service.

End the current conversation before saving settings. Task options are saved to the referenced Harness file; speaking length is saved to `model.length_penalty` in the Demo file. They apply to the next session. Harness fields not exposed in the UI are preserved.

| Setting | Default | Effect |
| --- | --- | --- |
| **Task mode** | General | Sends each delegate straight to the task agent, without a routing model call. Select Auto for automatic capability selection and task continuation. |
| **Task execution** | Server model, `low` effort | Controls the background worker. |
| **Auto routing** | Task model, `low`, 30 seconds | Used only in Auto mode. A timeout ends the routing request without launching its worker. |
| **Polish** | Codex, task model, `low` | Independently select official Gemini for progress and result preparation; the three-sentence cap remains. |
| **Multimodal** | Codex, task model, `low` | Independently select official Gemini for direct answers/media understanding when Auto routes to this capability. |
| **Speaking length** | `length_penalty = 0.8` | Values below 1 encourage earlier turn endings; 1 preserves the original behavior; values above 1 encourage longer turns. Range: 0.1–5. |

Blank Codex model names inherit the task model, then the server default. Blank Gemini model names use the default Gemini model below (`gemini-flash-latest` initially; a fixed version can be entered). Reasoning-effort settings apply only to Codex; Gemini uses its model defaults. Legacy shared Direct settings migrate into the independent Multimodal section.

#### Official Gemini

1. Enter a Google AI Studio API key in Settings → Task connection → Official Gemini connection, or set `GEMINI_API_KEY` / `GOOGLE_API_KEY` before starting the service.
2. Independently select Codex or Gemini for Polish, Auto routing, and Multimodal, with optional role-specific model IDs. Selecting Gemini sends that role's text/media to Google's API.
3. Save and start a new conversation. Defaults remain General and Codex for every role. Routing and Multimodal execute only through Auto; the General tool worker remains Codex and the streaming Omni model is unchanged.

A blank key field preserves the stored key. The configuration API returns only key availability. Keys entered through the browser are stored server-side in the file referenced by `harness.config` with mode `0600`; never commit this file. To remove a stored key, empty `gemini.api_key` in that file and check the service environment variables. Settings are shared across users of this service.

The implementation calls Google's fixed [GenerateContent endpoint](https://ai.google.dev/api/generate-content), with no configurable company relay and no silent provider fallback. Multimodal requests use the frozen delegate audio, video segment or image evidence. Inline requests above 20 MB fail explicitly; shorten the media window. Model access, quota and region support depend on your Google project.

## Service management

```bash
bash start.sh --config config.json --check --no-login  # Preflight while this checkout's services are stopped
bash start.sh --config config.json --detach           # Start in the background; return when ready
bash start.sh --status
bash start.sh --stop
```

Use the same `--config` file for preflight and startup. Each checkout manages one stack; stop it before switching configurations. `--no-login` reports missing authentication without starting login. For a foreground launch, Ctrl+C stops both services.

| Runtime location | Contents |
| --- | --- |
| `runtime/logs/model.log` | Model loading and inference. |
| `runtime/logs/web.log` | Browser connections, uploads, and session errors. |
| `runtime/logs/stack.log` | Detached startup and supervision. |
| the file referenced by `harness.config` | Saved model/task preferences and workspace selection. |
| `runtime/workspace/` | Example task workspace (if configured). Generated files remain after a session; browser download links are session-scoped. |
