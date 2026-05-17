# codex-switch

桌面 GUI，给 [codex CLI](https://github.com/openai/codex) 用户切换 OpenAI / DeepSeek / GLM 等 profile，自动处理 moonbridge 代理 + API key + model catalog。

[English](./README.md#english) | [中文](#中文)

---

## 中文

### 截图

![main window](docs/screenshot-main.png)

### 解决什么问题

codex CLI 0.130 砍掉了 `wire_api = "chat"`，只支持 OpenAI 的 `/v1/responses` 协议。要让 codex 连 DeepSeek / 智谱 GLM / 任何不暴露 Responses API 的国内云，必须走协议翻译代理（[moonbridge](https://github.com/ZhiYi-R/moon-bridge)）。手工编 TOML / YAML 麻烦，key 又得管，这个 app 一站式干掉。

### 为什么不用 cc-switch？

[cc-switch](https://github.com/farion1231/cc-switch)（71K stars）是个多 CLI 切换 profile 的通用工具，支持 Claude / Codex / Gemini 等。**如果你用 codex CLI + OpenAI 官方 / OpenRouter / 其他暴露 Responses API 的 provider，cc-switch 完全够用。**

但 codex CLI 0.130 + 国内云（DeepSeek / 智谱 GLM / 等）这个组合 cc-switch **不 work**，原因：

1. **codex 0.130 砍 `wire_api = "chat"`** → 只能走 `/v1/responses`。DeepSeek / GLM 没这个 endpoint，必须协议翻译
2. **cc-switch 内置 proxy 覆盖不全** → 实测缺 reasoning effort 透传 / SSE 兼容 / DeepSeek `extensions.deepseek_v4` 这些细节，跑不通 codex + DeepSeek 完整 session
3. **必须 moonbridge** —— [moonbridge](https://github.com/ZhiYi-R/moon-bridge) 是 GPL-3.0 Go binary，专门做 Responses ↔ Anthropic / OpenAI Chat 多协议翻译。cc-switch 不集成它
4. **cc-switch 是单 active provider 设计** → 一次只一个 provider 进 codex config。codex-switch 走 multi-profile 共存（codex 0.130 原生 `--profile` + session 内 `/model` 切，零摩擦切 7 个 GLM model 或两套 key）

codex-switch 是 **codex CLI 0.130 + 国内云**组合的 specialist。把 cc-switch 的 UI 思路 vendor 过来（preset 卡片 / form / JSON 编辑），但 Rust 业务逻辑 / moonbridge 集成 / multi-provider catalog 都是新写。

### 功能 (v0.2.0-alpha.1)

- **通用 Add Provider** — 选模板（DeepSeek / 智谱 GLM）→ 输 API key → 自动 probe 真实 model 列表 → 多选启用哪些 → 一键写盘
- **Multi-provider 共存** — DeepSeek + GLM 同时装，codex 通过 `--profile <name>` 切，tray 菜单和主窗口都能切
- **Re-probe** — provider 升级新 model 时，Edit 里点重新探测，自动 merge 进现有 enabled 状态
- **Moonbridge daemon 管理** — 顶栏徽章看 listener 状态，start/stop 不离 app
- **明文 toggle** — API key 输入框眼睛图标，临时显示明文核对
- **中英双语 UI** — 跟系统语言或手动切
- **未签名 dmg** — 首次开需手动绕过 Gatekeeper（见下面安装说明）

### 平台支持

- ✅ **macOS 11+ (Apple Silicon arm64 only)** — 已发布
- ⏳ macOS Intel (x86_64) — coming later
- ⏳ Windows / Linux — coming later (Windows 需补 stop/lsof 替代，Linux 缺 CI)

### 安装

1. 在 [Releases](https://github.com/qinqiangdavid/codex-switch/releases) 下载最新 `.dmg`
2. 双击挂载，把 `codex-switch.app` 拖到 Applications
3. **绕过 Gatekeeper**（未签名）：
   - 右键 `.app` → 打开 → "仍要打开"
   - 或终端跑 `xattr -cr /Applications/codex-switch.app`
4. 首次开会看到 "Setup incomplete" 提示，照 Settings tab 装：
   - [codex CLI](https://github.com/openai/codex) (`npm install -g @openai/codex` 或 brew)
   - [moonbridge](https://github.com/ZhiYi-R/moon-bridge) (`go install github.com/ZhiYi-R/moon-bridge/...@latest`)

### Quick Start

1. 打开 codex-switch，主窗口 Profiles tab 点 `[+ Add Provider]`
2. 选 **DeepSeek**（或 GLM），输入 API key
3. 点 **[Probe & Add]**，看到上游真实 model 列表 → 勾选要启用的 → 选 default model → Finish
4. 顶栏看 Moonbridge 徽章绿色 = listener 已起
5. 终端跑 `codex --profile deepseek` 或 `codex --profile glm` 开始用

### 已知限制 (alpha)

- arm64 only，Intel Mac 装不了
- 未签名，每次升级新 dmg 需重新允许 Gatekeeper
- moonbridge daemon 需用户单独装（自动下载 binary 还没做）
- API key 现在以明文存 `~/.codex-switch/providers.json` (权限 0600)。OS Keychain 在 alpha 阶段因为未签名 app 拒访已绕过

### ⚠️ LLM 自报 model identity 不准（重要）

在 codex session 里问"你是哪个模型"，得到的答案**经常错**（比如切到 GLM-4.7 但 LLM 说自己是 GLM-5.1，或者切到 GLM 但说自己是 DeepSeek）。这**不是 codex-switch 或路由 bug**：

1. **实际请求路由是对的** —— moonbridge 收到的 model 字段、转发的上游 endpoint、收到的 response 都准。看 codex `/status` 才是 source of truth
2. **codex 0.130 `/model` session 内切换只改 outgoing 请求**，不重发 system context，所以 LLM 看到的 system prompt 还是启动时那个
3. **LLM 本身不知道自己叫什么** —— 没有 model identity API。中国系 LLM（GLM / DeepSeek / Qwen 等）训练数据互相 contaminate，被问"你是谁"时 hallucinate

**判断 model 实际跑啥**：用 `codex /status` 看 Model 字段；或者退出 + `codex --profile X --model Y` 重启 session（启动时指定 model 才会进 system context）。不要按 LLM 自报判断。

### License

MIT。详见 [LICENSE](./LICENSE)。

---

## English

Desktop GUI for [codex CLI](https://github.com/openai/codex) users to switch profiles between OpenAI / DeepSeek / GLM and other providers, with automatic moonbridge proxy + API key + model catalog management.

### Why

codex CLI 0.130 removed `wire_api = "chat"` support, so connecting codex to DeepSeek / GLM / any provider that doesn't expose OpenAI's `/v1/responses` requires a protocol bridge ([moonbridge](https://github.com/ZhiYi-R/moon-bridge)). Manually editing TOML / YAML and managing API keys is painful — this app takes care of it.

### Why not cc-switch?

[cc-switch](https://github.com/farion1231/cc-switch) (71K stars) is a generic profile switcher for multiple AI CLIs (Claude / Codex / Gemini). **If you use codex CLI with OpenAI / OpenRouter / any provider exposing `/v1/responses`, cc-switch works perfectly.**

But **codex 0.130 + Chinese cloud providers (DeepSeek / GLM / ...) doesn't work with cc-switch**, because:

1. **codex 0.130 dropped `wire_api = "chat"`** → must use `/v1/responses`; DeepSeek / GLM don't expose that, need protocol translation
2. **cc-switch's built-in proxy is incomplete** for reasoning effort pass-through, SSE compat, and DeepSeek `extensions.deepseek_v4` quirks
3. **moonbridge is required** — a GPL-3.0 Go binary that does Responses ↔ Anthropic / OpenAI Chat translation. cc-switch doesn't integrate it
4. **cc-switch is single-active-provider** — one provider in codex config at a time. codex-switch goes multi-profile coexistence (codex 0.130 native `--profile` + in-session `/model` switching)

codex-switch is a **codex CLI 0.130 + Chinese cloud** specialist. UI patterns vendored from cc-switch (preset cards / form / JSON editor), business logic / moonbridge integration / multi-provider catalog all rewritten in Rust.

### Features (v0.2.0-alpha.1)

- **Generic Add Provider** — pick template (DeepSeek / GLM) → enter API key → auto-probe real model list → multi-select which to enable → one-click write
- **Multi-provider coexistence** — DeepSeek + GLM at the same time, switch via `codex --profile <name>` or tray menu
- **Re-probe** — when provider releases new models, click Re-probe in Edit dialog to merge into existing enabled state
- **Moonbridge daemon control** — listener up/down badge in header, start/stop without leaving the app
- **Show / hide API key toggle** — eye icon in inputs to verify pasted keys
- **Bilingual UI** — English / Simplified Chinese, follows OS locale

### Platform Support

- ✅ **macOS 11+ (Apple Silicon arm64 only)** — released
- ⏳ macOS Intel (x86_64) — coming later
- ⏳ Windows / Linux — coming later

### Installation

1. Download latest `.dmg` from [Releases](https://github.com/qinqiangdavid/codex-switch/releases)
2. Mount and drag `codex-switch.app` into Applications
3. **Bypass Gatekeeper** (unsigned):
   - Right-click `.app` → Open → "Open anyway"
   - Or `xattr -cr /Applications/codex-switch.app` in terminal
4. On first launch, install dependencies as instructed in the Settings tab:
   - [codex CLI](https://github.com/openai/codex)
   - [moonbridge](https://github.com/ZhiYi-R/moon-bridge)

### Quick Start

1. Open codex-switch, click `[+ Add Provider]` in Profiles tab
2. Pick **DeepSeek** or **GLM**, paste API key
3. Click **[Probe & Add]**, select which models to enable → pick default → Finish
4. Verify Moonbridge badge in header is green
5. Run `codex --profile deepseek` or `codex --profile glm`

### Known limitations (alpha)

- arm64 only, Intel Mac not yet supported
- Unsigned, every dmg upgrade requires re-allowing Gatekeeper
- moonbridge daemon installation is manual (auto-download not yet implemented)
- API keys stored as plaintext in `~/.codex-switch/providers.json` (mode 0600). OS Keychain bypassed in alpha due to unsigned-app access issues.

### ⚠️ LLM self-reports of model identity are unreliable

When you ask the LLM "what model are you?" in a codex session, the answer is **often wrong** (e.g. you switched to GLM-4.7 but the LLM says it's GLM-5.1, or you're on GLM but it claims to be DeepSeek). This is **not a codex-switch or routing bug**:

1. **Actual request routing is correct** — moonbridge receives the right `model` field, forwards to the right upstream endpoint. Trust `codex /status` as source of truth
2. **codex 0.130 `/model` switching only changes outgoing requests**, doesn't replay system context. LLM still sees the system prompt from session start
3. **LLMs don't actually know their own identity** — there's no model identity API. Chinese LLMs (GLM / DeepSeek / Qwen) have training-data overlap and hallucinate when asked

**To know what model is really running**: use `codex /status` to read the Model field. Or restart with `codex --profile X --model Y` to put the desired model into the system context from the start. Don't trust LLM self-reports.

### License

MIT, see [LICENSE](./LICENSE).
