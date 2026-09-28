# Hermes Manual Agent

A public GitHub Actions workflow that runs **on-demand** (triggered manually from the GitHub mobile app) and delivers results to your **Telegram bot**. It boots a self-contained Hermes Agent + FreeLLMAPI stack on a GitHub Actions runner, waits for your voice note, processes it with all available tools, and sends the output back to your Telegram chat.

## Architecture

```
GitHub Mobile → workflow_dispatch → GitHub Actions runner (ubuntu-latest)
                                    │
                                    ├─ FreeLLMAPI (Node.js, port 3001)
                                    │   └─ loads ALL your free API keys from FREEAPI_CONFIG_JSON
                                    │
                                    ├─ Hermes Agent (Python 3.11)
                                    │   └─ uses FreeLLMAPI as its model provider
                                    │       └─ local STT (faster-whisper) for voice messages
                                    │
                                    └─ Telegram Gateway
                                        └─ receives your voice note → STT → Hermes → result → Telegram
```

## Quick Start

### 1. Trigger the workflow

Open the **Actions** tab in this repository → select **`manual-agent`** → click **Run workflow**.

- `test_mode = false` (default): waits up to 10 minutes for a voice note on Telegram.
- `test_mode = true`: runs 5 self-tests instead of waiting (see below).

### 2. Send a voice note

While the workflow is running, send a **voice message** to your Telegram bot. Hermes transcribes it locally with `faster-whisper`, processes the text with its toolset, and sends the result back.

### 3. Stop a running workflow

In the **Actions** tab → click the running workflow → **Cancel workflow**.

### 4. Test mode

Set `test_mode = true` to verify the full stack without waiting for a voice note:

| Test | What it checks |
|------|----------------|
| FreeLLMAPI connectivity | `/api/ping` responds 200 within 3 minutes |
| Model connectivity | Hermes sends `reply with the single word "ok"` through FreeLLMAPI; asserts response contains "ok" |
| STT connectivity | Generates audio with `espeak`, runs STT, asserts output contains "test" or "transcription" |
| Telegram file send | Creates a dummy PDF, sends it as a `document` to your Telegram chat, asserts exit code 0 |
| Secret leakage scan | `git diff` / `git status` — fails if any secret-pattern file is staged |

## Secrets Required

Go to **Settings → Secrets and variables → Actions** and create these 5 secrets:

| Secret | Description | Example |
|--------|-------------|---------|
| `TELEGRAM_BOT_TOKEN` | From BotFather (`/newbot`) | `1234567890:ABC...` |
| `TELEGRAM_CHAT_ID` | Your numeric chat ID | `123456789` |
| `TELEGRAM_ALLOWED_USERS` | Your Telegram user ID (allowlist) | `123456789` |
| `FREEAPI_CONFIG_JSON` | JSON blob with all provider keys | See below |
| `FREELLMAPI_UNIFIED_KEY` | Unified key (see note below) | Read from first run if needed |

### `FREEAPI_CONFIG_JSON` format

```json
{
  "keys": [
    { "platform": "groq", "key": "gsk_...", "label": "main" },
    { "platform": "google", "key": "AIza...", "label": "main" },
    { "platform": "cerebras", "key": "csk-...", "label": "main" },
    { "platform": "mistral", "key": "...", "label": "main" }
  ]
}
```

List **every** provider key you have. FreeLLMAPI will use them all in the fallback chain. Supported platforms include: `groq`, `google`, `cerebras`, `mistral`, `openai`, `anthropic`, `deepseek`, `cohere`, `together`, `fireworks`, `perplexity`, `xai`, and more.

### Unified Key Note

FreeLLMAPI generates a unified API key on first boot. If it is not deterministic across runs, read it from the first successful run's logs and save it as `FREELLMAPI_UNIFIED_KEY`. The workflow attempts to read it automatically from the FreeLLMAPI database; if that fails, the workflow will error with instructions.

## Security

- **No secrets in code, logs, or commits.** All secrets live exclusively in GitHub Secrets (encrypted at rest).
- **`workflow_dispatch`-only trigger** prevents fork-based secret exfiltration: the workflow never runs on `pull_request` or `push` from forks, so a malicious fork cannot trigger the workflow with access to your secrets.
- **`permissions: contents: read`** minimizes the `GITHUB_TOKEN` scope — the workflow cannot push, delete, or modify anything in the repo.
- **No `echo` on secrets, no `printenv`, no writing secrets to files.**
- **Secret leakage scan** runs at the end of every workflow: `git diff` and `git status` are checked; if any file matching secret patterns is staged, the workflow fails.
- **No artifacts are stored.** All outputs are sent to Telegram only and discarded when the runner is reclaimed.

## `.gitignore`

```
.env
*.json
~/.hermes/
**/provider_keys.json
**/freellmapi_keys*
```