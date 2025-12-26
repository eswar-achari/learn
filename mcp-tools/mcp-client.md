Below is a **Python example** showing how to connect to a **Streamable HTTP MCP server** (Model Context Protocol) using the Python SDK and standard HTTP requests.

The **Streamable HTTP transport** in MCP works by sending JSON-RPC 2.0 messages to the remote MCP endpoint using **HTTP POST** and optionally listening for server responses via the same stream. The client should set an `Accept` header that supports both `application/json` and `text/event-stream` in case the server uses HTTP streaming (SSE) as part of the transport. ([modelcontextprotocol.io][1])

---

## 📌 Basic Python MCP Client (Streamable HTTP)

This example shows a minimal Python client that:

1. Sends an `InitializeRequest` to start a session
2. Sends a JSON-RPC request over HTTP POST
3. Reads responses (synchronously)

You will need:

* Python 3.7+
* `httpx` (for HTTP requests)
* Optionally `sseclient-py` if you expect real SSE streams

Install dependencies:

```bash
pip install httpx sseclient-py
```

---

### 💻 Code Example

```python
import httpx
import json
from sseclient import SSEClient

# URL of your streamable MCP server
MCP_URL = "https://example.com/mcp"  # change to your MCP endpoint

# JSON-RPC method names
INITIALIZE_METHOD = "initialize"
PING_METHOD = "ping"

# Helper to build a JSON-RPC message
def jsonrpc_request(method, params=None, req_id=1):
    return {
        "jsonrpc": "2.0",
        "method": method,
        "params": params or {},
        "id": req_id,
    }

def connect_and_send():
    with httpx.Client(timeout=None) as client:
        # ————————————————————————————————
        # INITIALIZE SESSION
        # ————————————————————————————————
        init_req = jsonrpc_request(INITIALIZE_METHOD)

        headers = {
            "Accept": "application/json, text/event-stream",
            "Content-Type": "application/json",
        }

        response = client.post(
            MCP_URL,
            headers=headers,
            data=json.dumps(init_req),
        )

        # If server streams or returns JSON:
        content_type = response.headers.get("Content-Type", "")
        print("Init status:", response.status_code, "Content-Type:", content_type)

        if content_type.startswith("application/json"):
            init_resp = response.json()
            print("InitializeResponse:", init_resp)
        else:
            # If the server used SSE for responses
            print("Listening via SSE…")
            for event in SSEClient(response.text):
                print("SSE event:", event.data)

        # ————————————————————————————————
        # SEND A NORMAL JSON-RPC REQUEST
        # ————————————————————————————————
        ping_req = jsonrpc_request(PING_METHOD, params={"message": "hello!"}, req_id=2)

        post_resp = client.post(
            MCP_URL,
            headers=headers,
            data=json.dumps(ping_req),
        )

        # Handle JSON or SSE reply
        ct = post_resp.headers.get("Content-Type", "")
        if ct.startswith("application/json"):
            print("JSON Response:", post_resp.json())
        else:
            print("SSE Stream:")
            for evt in SSEClient(post_resp.text):
                print(evt.data)

if __name__ == "__main__":
    connect_and_send()
```

---

## 📝 Notes

### 📤 POST Requests

* Each JSON-RPC message (initialize/query/notification) must be sent via **HTTP POST** to the same MCP endpoint. ([modelcontextprotocol.io][1])

### 📡 HTTP Headers

* Include an `Accept: application/json, text/event-stream` header so that the server can use a streamable HTTP response if it supports streaming. ([modelcontextprotocol.io][2])
* You may also need to pass authentication headers (Bearer tokens, API keys, etc.) depending on your server.

### 📨 Receiving Server Events

* If the server returns **SSE** (`Content-Type: text/event-stream`), use an SSE client or parse the raw stream. (The `sseclient-py` package can help.) ([modelcontextprotocol.io][2])

### 🪪 Sessions

* Some servers return a session ID (`Mcp-Session-Id`) in the initial response. If so, you must include that header in subsequent requests. ([modelcontextprotocol.io][3])

### 🛠 Python SDK Alternative

If you’d prefer to use the **official Python MCP SDK**, it may have built-in helpers for HTTP transport and session handling. Installation and usage vary depending on the version — check the SDK README on GitHub. ([GitHub][4])

---

If you want, I can also provide an **asynchronous (async/await)** version using `httpx.AsyncClient` and streaming with `async for` loops—just let me know!

[1]: https://modelcontextprotocol.io/specification/2025-06-18/basic/transports?utm_source=chatgpt.com "Transports"
[2]: https://modelcontextprotocol.io/docs/concepts/transports?utm_source=chatgpt.com "Transports"
[3]: https://modelcontextprotocol.io/specification/2025-03-26/basic/transports?utm_source=chatgpt.com "Transports"
[4]: https://github.com/modelcontextprotocol/python-sdk?utm_source=chatgpt.com "The official Python SDK for Model Context Protocol servers ..."
