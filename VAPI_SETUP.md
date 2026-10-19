# Vapi voice — dashboard setup

Scheduling over voice uses the **same tool handlers** as web chat (`runTool` in `src/lib/chat-tools.ts`). Vapi must call your **Server URL** so tool results reach this app.

## 1. Server URL (production)

In the Vapi dashboard, set the assistant’s **Server URL** to:

```text
https://YOUR_DOMAIN/api/vapi/webhook
```

Use HTTPS (required in production). The route must be publicly reachable from Vapi’s servers.

## 2. Tool / function definitions

Your saved assistant’s model must expose **functions** whose **`name` and JSON schemas** match the web app:

| Tool name | Purpose |
|-----------|---------|
| `match_doctor` | Body-part / concern → specialist |
| `get_available_slots` | List slots for a doctor |
| `book_appointment` | Confirm booking |
| `send_sms_reminder` | SMS after opt-in |
| `get_office_info` | Hours / address |
| `check_prescription_status` | Mock refill status |

**Single source of truth in code:** `getClaudeTools()` in `src/lib/claude.ts`.

**OpenAI-style export for the Vapi UI / API:** `getVapiModelFunctions()` in `src/lib/vapi-tools.ts` — paste or sync that structure into your assistant model (Vapi accepts OpenAI function calling shape).

If names or required fields differ between the dashboard and `runTool`, the webhook will run the wrong handler or the model will send invalid parameters.

## 3. Async / max tokens

Per [Vapi custom tools troubleshooting](https://docs.vapi.ai/tools/custom-tools-troubleshooting), set tools to **sync** (`async: false`, default) and increase **`maxTokens`** on tools that return large slot lists (e.g. 500) if you see truncation warnings in call logs.

## 4. Webhook response rules

`POST /api/vapi/webhook` returns:

- **`tool-calls`:** `{ "results": [ { "name", "toolCallId", "result": "<single-line string>" } ] }` or `error` instead of `result`. Always **HTTP 200**.
- **`assistant-request`:** `{ "assistantOverrides": { "firstMessage": "..." } }` only (no `model.messages` override), so tools from the saved assistant stay attached.

## 5. Outbound “call me” from the web app

`POST /api/vapi/call` starts an outbound call with:

- `ensureSession` + `indexSessionPhone` so **tool-calls** on that call resolve the **same** `sessionId` as the browser.

## 6. Smoke-test checklist

1. Web chat: book once; confirm email (and SMS if you use `send_sms_reminder`).
2. From the same browser session, tap **Call**; complete a booking on the phone.
3. In Vapi call logs, confirm **tool-calls** hit your Server URL and return **results** (not “no result returned”).
4. Optional: dial your Kyron number **inbound** after a web chat; assistant should use `assistant-request` `firstMessage` only; tools should still work if configured on the assistant.


## 7. Webhook Security

The Vapi webhook can execute backend actions such as appointment booking and sending notifications.

For secure deployments:

- Use HTTPS for all webhook communication.
- Authenticate incoming webhook requests using Vapi Custom Credentials.
- Validate tool names, required parameters, and permitted actions before execution.
- Restrict tool execution to supported scheduling and information workflows.
- Avoid logging sensitive patient information, API keys, and authentication tokens.
- Prevent duplicate booking and notification actions during retries.

**Note:** The application currently uses simulated healthcare data and in-memory storage. It is not intended for processing real protected health information without additional security and compliance controls.


## References

- [Server events (tool-calls)](https://docs.vapi.ai/server-url/events)
- [Custom tools troubleshooting](https://docs.vapi.ai/tools/custom-tools-troubleshooting)
