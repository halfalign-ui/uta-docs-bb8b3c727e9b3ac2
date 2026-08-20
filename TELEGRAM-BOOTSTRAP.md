# UTA Telegram Interface & Local Conversation Runtime

This document details the Telegram transport layer and its integration with the Local Conversation Runtime introduced for the Post-F8 development cycle.

## Purpose

The Telegram bot acts as the external, user-facing interface for UTA. It routes messages into the **Conversation Runtime**, which connects to the local UTA model backend to provide a continuous conversational experience.

- It uses the local `llama-server` as the core intelligence (UTA "Soul").
- It translates raw Telegram JSON payloads into a normalized internal UTA event (`MessageEvent`).
- It isolates conversation contexts (memory) based on the session ID (e.g., `telegram:<chat_id>`).
- It does **not** yet invoke the Agent Runtime or external Tools.
- It does **not** route to cloud models.

## Architecture

```text
User
  ↓
Telegram API (getUpdates)
  ↓
TelegramAdapter (gate/interfaces/telegram/bot.py)
  ↓
MessageEvent (Normalized internal representation)
  ↓
ConversationRuntime (gate/interfaces/runtime.py)
  ↓
LocalModelProvider (gate/agent/providers/local.py)
  ↓
llama-server (http://127.0.0.1:8080)
```

## Configuration & Secrets

The configuration is managed via the existing `gate.config` infrastructure.

### Environment Variables
- `UTA_TELEGRAM_TOKEN`: The Telegram Bot token (required). Treat as a sensitive secret. Never hardcode or commit to version control.
- `UTA_LOCAL_MODEL_URL`: The endpoint of the local model (defaults to `http://127.0.0.1:8080`).

### Running the Interface

To run the bot locally:
```bash
export UTA_TELEGRAM_TOKEN="<YOUR_TELEGRAM_BOT_TOKEN>"
python -m gate.interfaces.telegram
```

In a production environment, this token should be securely added to the `/etc/uta/gate.env` file.

## Error Behavior & Token Accounting
- If the local model is unavailable, the runtime responds with a graceful error message without exposing backend stack traces, configurations, or paths to the user.
- The `LocalModelProvider` parses model responses into `ModelResponse` objects, which include `TokenUsage` metadata (prompt, completion, and total tokens). This is logged internally for future token accounting and quota enforcement.

## Next Steps
This milestone establishes the first alive interface to the local model. Future development will expand the runtime to include the `Receptionist` layer, long-term memory, cloud routing, and Agent Tool invocation.
