# AG-UI Workbench

A workbench for the [AG-UI](https://github.com/ag-ui-protocol/ag-ui) agent event protocol. Connect to a live agent endpoint and watch its typed event stream render into an app in real time, or explore the protocol offline with built-in demo agents, stream replay, and per-event validation against the official 1.0 JSON Schema.

**Live demo: https://meetaxel.github.io/ag-ui-workbench/**

Everything is a single static `index.html`: no build step, no backend of its own, no account, no API key.

## Three ways to use it

**1. Live agent.** Point it at any running AG-UI HTTP agent. It POSTs a `RunAgentInput` (thread id, run id, your message history, plus optional state, tools and context) with `Accept: text/event-stream` and renders the SSE stream as it arrives: streaming text, tool calls with arguments and results, steps, reasoning, activities, subagents and shared state. Multi-turn works; "New thread" starts a fresh conversation.

**2. Built-in demos.** Four recorded scenarios (streaming chat, tool calling, shared state with JSON Patch, and a full agent run) that need nothing at all. Good for learning the protocol or demoing without a server.

**3. Replay file.** Load a `.jsonl` event stream (one AG-UI event per line), for example one saved with the Download button, and replay it through the renderer.

Every event in every mode can be validated against the official AG-UI 1.0 JSON Schema (bundled). Invalid events get a red badge with the exact schema errors. The Download button exports the current stream as `.jsonl`.

## Run it

Use the live demo above, or open `index.html` in a modern browser, or serve the folder with any static server. The whole app is the one file; the protocol schema and the ajv validator are bundled inside it.

## Try live mode free: a minimal local agent

Adapted from the upstream repo's README (MIT). Requires Python 3.9+:

```bash
pip install ag-ui-protocol fastapi uvicorn
```

```python
# server.py
import uuid
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from fastapi.middleware.cors import CORSMiddleware
from ag_ui.core import (
    PROTOCOL_VERSION, RunAgentInput, RunStartedEvent, RunFinishedEvent,
    TextMessageStartEvent, TextMessageContentEvent, TextMessageEndEvent,
)
from ag_ui.encoder import EventEncoder

app = FastAPI()

# browsers enforce CORS; allow the workbench page to call this server
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])

@app.post("/")
async def run_agent(input_data: RunAgentInput, request: Request):
    encoder = EventEncoder(accept=request.headers.get("accept"))
    async def events():
        yield encoder.encode(RunStartedEvent(
            thread_id=input_data.thread_id, run_id=input_data.run_id,
            protocol_version=PROTOCOL_VERSION))
        message_id = str(uuid.uuid4())
        yield encoder.encode(TextMessageStartEvent(message_id=message_id, role="assistant"))
        for chunk in ["Hello ", "from ", "AG-UI!"]:
            yield encoder.encode(TextMessageContentEvent(message_id=message_id, delta=chunk))
        yield encoder.encode(TextMessageEndEvent(message_id=message_id))
        yield encoder.encode(RunFinishedEvent(thread_id=input_data.thread_id, run_id=input_data.run_id))
    return StreamingResponse(events(), media_type=encoder.get_content_type())
```

```bash
uvicorn server:app --port 8000
```

Enter `http://localhost:8000/` as the endpoint in the workbench and run it. Real LLM-backed agents work the same way; those need whatever keys their framework needs, which is your choice and outside this tool.

## Honest scope

- Live mode talks only to the endpoint URL you enter. Like any browser client, it needs that server to allow this page's origin (CORS).
- Built-in demos and replay are local simulations: no language model, no agent framework, no network.
- Schema validation checks each event against the official AG-UI 1.0 JSON Schema. It does not check event ordering or runtime behaviour, so it is not a full conformance suite.
- AG-UI's binary protobuf framing is detected and reported, but not decoded in this build; ask the endpoint for `text/event-stream`.

## Bundled components and credit

- The official AG-UI 1.0 JSON Schema, bundled inside `index.html`, from [ag-ui-protocol/ag-ui](https://github.com/ag-ui-protocol/ag-ui), MIT License, copyright 2025 the AG-UI authors (created by CopilotKit).
- [ajv](https://ajv.js.org) 8, bundled inside `index.html`, MIT License, copyright Evgeny Poberezkin and contributors.
- Headings use [Cal Sans](https://www.fontshare.com/fonts/cal-sans) from Fontshare, with a system fallback when offline.
- Event names, field shapes and reference descriptions follow the upstream specification.

This is an independent community project, not affiliated with or endorsed by the AG-UI project.

## License

MIT. See [LICENSE](LICENSE). Bundled third-party components remain under their own MIT licenses as noted above.
