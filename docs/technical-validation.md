# Technical validation: official MCP Python SDK v2

Date: 2026-10-13. Disposable spike: `/tmp/hn-mcp-v2-spike.AayvfOCe` (removed after validation). Python: 3.12.3. These observations are for the installed versions below, not a promise about future releases. No application code or dependency manifest was added to this repository.

The authoritative Prompt 0.2 text is restored in [phase-0-requirements.md](phase-0-requirements.md). Status labels below describe historical execution where stated, and current official-document checks where stated; this audit did not repeat the removed spike. On 2026-10-14, the official [SDK v2 docs](https://py.sdk.modelcontextprotocol.io/whats-new/) described v2 as stable, and [PyPI](https://pypi.org/project/mcp/) listed 2.3.0 (uploaded 2026-10-02) as the latest release. The original install command used `mcp>=2,<3` and resolved 2.3.0. **VERIFIED (historical install and current source check).**

## Sources read

- [What's new in v2](https://py.sdk.modelcontextprotocol.io/whats-new/)
- [Add to an existing app](https://py.sdk.modelcontextprotocol.io/run/asgi/)
- [Deploy & scale](https://py.sdk.modelcontextprotocol.io/run/deploy/)
- [Testing](https://py.sdk.modelcontextprotocol.io/get-started/testing/)
- [Handling errors](https://py.sdk.modelcontextprotocol.io/servers/handling-errors/)
- [Migration Guide: v1 to v2](https://py.sdk.modelcontextprotocol.io/v2/migration/)

## Exact shell commands and observed output

The spike script existed only at the `/tmp` path below. It built an `MCPServer` with one async `inspect_hn(query: str) -> dict[str, object]` tool, a FastAPI `/health` route, and a `Mount("/", app=mcp.streamable_http_app(...))` after the health route. The host lifespan entered `async with mcp.session_manager.run()`. It started uvicorn on a loopback ephemeral port and used the SDK's `Client` for live and in-memory checks. For HTTP mocks it used `httpx2.MockTransport` with `streamable_http_client(..., http_client=...)`; for a prospective HN HTTP client it used `respx` with plain `httpx.AsyncClient`. The script is summarized here because the disposable source was removed.

```sh
mktemp -d /tmp/hn-mcp-v2-spike.XXXXXXXX
# /tmp/hn-mcp-v2-spike.AayvfOCe
python3 -m venv /tmp/hn-mcp-v2-spike.AayvfOCe/venv
/tmp/hn-mcp-v2-spike.AayvfOCe/venv/bin/python -m pip install --disable-pip-version-check 'mcp>=2,<3' fastapi uvicorn httpx respx pytest-httpx
```

The first pip invocation inside the restricted sandbox could not resolve the package index (`Name or service not known`); it was interrupted and the exact pip command was rerun with network permission. The successful run ended with:

```text
Successfully installed ... fastapi-0.143.0 ... httpx-0.28.1 httpx2-2.13.1 ... mcp-2.3.0 mcp-types-2.3.0 ... respx-0.23.1 ... uvicorn-0.54.0
```

The metadata inspection command executed was:

```sh
/tmp/hn-mcp-v2-spike.AayvfOCe/venv/bin/python -c 'import inspect, importlib.metadata as md, mcp, httpx2; from mcp import Client; from mcp.server import MCPServer; print("mcp", md.version("mcp"), "fastapi", md.version("fastapi"), "uvicorn", md.version("uvicorn")); print("requires-python", md.metadata("mcp").get("Requires-Python")); print("requires-dist", md.requires("mcp")); print("Client", inspect.signature(Client)); print("streamable_http_app", inspect.signature(MCPServer.streamable_http_app)); print("httpx2.MockTransport", hasattr(httpx2, "MockTransport"))'
```

Observed output, trimmed:

```text
mcp 2.3.0 fastapi 0.143.0 uvicorn 0.54.0
requires-python >=3.10
httpx2.MockTransport True
```

The final spike command executed was:

```sh
/tmp/hn-mcp-v2-spike.AayvfOCe/venv/bin/python /tmp/hn-mcp-v2-spike.AayvfOCe/spike.py
```

Observed output, trimmed (session ID and ephemeral port varied between runs):

```text
VERSION mcp 2.3.0
VERSION mcp-types 2.3.0
VERSION fastapi 0.143.0
VERSION uvicorn 0.54.0
VERSION httpx2 2.13.1
VERSION httpx 0.28.1
VERSION respx 0.23.1
VERSION pytest-httpx 0.36.2
REQUIRES_PYTHON >=3.10
HTTPX_COEXIST True
MEMORY_PROTOCOL 2026-07-28
MEMORY_LIST ['inspect_hn']
MEMORY_CALL {'is_error': False, 'structured_content': {'query': 'python', 'count': 1}}
MEMORY_ERROR tool-error {'is_error': True, 'content': ['Error executing tool inspect_hn: recoverable miss'], 'structured_content': None}
MEMORY_ERROR protocol-error MCPError invalid request code= -32602
MEMORY_ERROR crash {'is_error': True, 'content': ['Error executing tool inspect_hn'], 'structured_content': None}
HEALTH 200 {'status': 'ok'}
DEFAULT_HOST 421 Invalid Host header
HTTP_PROTOCOL 2026-07-28
HTTP_LIST ['inspect_hn']
HTTP_CALL {'is_error': False, 'structured_content': {'query': 'news', 'count': 1}}
HTTP_ERROR tool-error {'is_error': True, 'content': ['Error executing tool inspect_hn: recoverable miss'], 'structured_content': None}
HTTP_ERROR protocol-error MCPError invalid request code= -32602
HTTP_ERROR crash {'is_error': True, 'content': ['Error executing tool inspect_hn'], 'structured_content': None}
STATEFUL_LEGACY_SESSION_HEADERS ['91b514a3f080454eb7b5650c5ba995c2', ... same ID across five responses]
STATELESS_LEGACY_PROTOCOL 2025-11-25
STATELESS_LEGACY_LIST ['inspect_hn']
STATELESS_LEGACY_CALL {'is_error': False, 'structured_content': {'query': 'legacy', 'count': 1}}
ALLOWED_HOST_AND_STATELESS_HEADERS [(200, None), (202, None), (200, None), (200, None)]
STATELESS_MODERN_PROTOCOL 2026-07-28
STATELESS_MODERN_CALL {'is_error': False, 'structured_content': {'query': 'modern', 'count': 1}}
SDK_MOCK_CALLS ['/mcp', '/mcp']
SDK_MOCK_LIST ['inspect_hn']
SDK_MOCK_CALL {'is_error': False, 'structured_content': {'query': 'mocked', 'count': 1}}
SDK_MOCK_SUCCESS_CALLS ['/mcp', '/mcp', '/mcp']
HN_RESPX True 200 {'id': 1, 'title': 'HN item'}
```

`MEMORY_CALL`, `HTTP_CALL`, and other successful tool output also contained JSON text in `content`; those text blocks are omitted above. The script exited with status 0. An earlier attempt to run live HTTP inside the sandbox failed at `socket.socket()` with `PermissionError: [Errno 1] Operation not permitted`; the same script succeeded with loopback permission.

Cleanup command executed after the successful run (exit status 0; no output):

```sh
rm -r /tmp/hn-mcp-v2-spike.AayvfOCe
```

## SDK package metadata

**VERIFIED** `pip install 'mcp>=2,<3'` resolved `mcp==2.3.0` in this environment. The distribution's `Requires-Python` is `>=3.10`. The complete `Requires-Dist` metadata observed was:

```text
anyio>=4.10; python_version >= '3.14'
anyio>=4.9; python_version < '3.14'
httpx2>=2.10.0
jsonschema>=4.20.0
mcp-types==2.3.0
opentelemetry-api>=1.28.0
pydantic>=2.12.0
pyjwt[crypto]>=2.10.1
python-multipart>=0.0.9
pywin32>=311; sys_platform == 'win32'
sse-starlette>=3.0.0
starlette>=0.27; python_version < '3.14'
starlette>=0.48.0; python_version >= '3.14'
typing-extensions>=4.13.0
typing-inspection>=0.4.1
uvicorn>=0.31.1; sys_platform != 'emscripten'
python-dotenv>=1.0.0; extra == 'cli'
typer>=0.16.0; extra == 'cli'
rich>=13.9.4; extra == 'rich'
```

**VERIFIED** The SDK requires `httpx2`, not plain `httpx`. Both packages imported and ran in the same venv; `httpx.AsyncClient is not httpx2.AsyncClient`. Plain `httpx` is therefore usable alongside the SDK when installed explicitly for the HN integration, but `httpx` objects should not be passed into SDK APIs expecting `httpx2` objects. The latter type rule follows the [Migration Guide](https://py.sdk.modelcontextprotocol.io/v2/migration/#httpx-and-httpx-sse-replaced-by-httpx2) and was not tested with a deliberately mismatched object, so that specific failure mode is **UNVERIFIED** here.

## Integration and transport behavior

**VERIFIED** `MCPServer` with an async dict-returning tool produced `structured_content` as a dict. Both `Client(mcp)` in memory and `Client("http://127.0.0.1:<port>/mcp")` over live HTTP returned the tool in `list_tools()` and its structured result in `call_tool()`. The modern protocol reported `2026-07-28` in both paths. The FastAPI `/health` route returned HTTP 200 beside `/mcp`. The host lifespan used `mcp.session_manager.run()`.

**VERIFIED** With no `transport_security`, a live `POST /mcp` carrying `Host: mcp.example.test` returned `421 Invalid Host header`. A second app used `TransportSecuritySettings(allowed_hosts=["127.0.0.1:*", "localhost:*", "mcp.example.test", "mcp.example.test:*"], allowed_origins=["https://app.example.test"])`; a valid SDK request with `Host: mcp.example.test` then returned HTTP 200 and the tool call succeeded. Browser CORS and Origin behavior were not exercised (**UNVERIFIED**); the [ASGI guide](https://py.sdk.modelcontextprotocol.io/run/asgi/#cors-for-browser-clients) describes additional browser configuration.

**VERIFIED** A legacy client (`mode="legacy"`, protocol `2025-11-25`) worked against both apps. With default stateful HTTP, five observed responses carried the same `Mcp-Session-Id`. With `stateless_http=True`, four observed legacy responses had no `Mcp-Session-Id`, and `list_tools()` plus `call_tool()` still worked. A modern client also worked on the stateless-configured app. Multiworker routing and back-channel behavior were not exercised (**UNVERIFIED**); see [Deploy & scale](https://py.sdk.modelcontextprotocol.io/run/deploy/).

**VERIFIED** Error behavior was the same in memory and over live HTTP: `ToolError("recoverable miss")` returned `is_error=True`, preserved the message in `content`, and had no `structured_content`; `MCPError(INVALID_PARAMS, "invalid request")` raised `MCPError` at the client with code `-32602`; `RuntimeError("private failure")` returned `is_error=True` with only `Error executing tool inspect_hn`, without the private text. The unexpected exception also produced a server traceback in the first, unsuppressed run.

## HTTP mocking decision

**VERIFIED** For the SDK HTTP transport, `httpx2.MockTransport` with a supplied `httpx2.AsyncClient` intercepted requests to `/mcp`. A mock returning 421 received two SDK requests; a mock forwarding to the FastAPI ASGI app received three requests and let the SDK client list and call the tool successfully. The HTTP client was supplied through `Client(streamable_http_client(url, http_client=mock_client))`.

**VERIFIED** For a future HN integration using plain `httpx.AsyncClient`, `respx 0.23.1` intercepted a mocked `GET https://hacker-news.firebaseio.com/v0/item/1.json` and returned `{'id': 1, 'title': 'HN item'}` without a network request. This establishes a workable test pair: `httpx2.MockTransport` for SDK transport tests and `respx` for HN API tests if the HN client is plain `httpx`. The actual HN integration does not exist yet and was not tested (**UNVERIFIED**). `pytest-httpx` was installed but not exercised (**UNVERIFIED**); no compatibility claim is made for it.

## Phase 0 audit refresh (2026-10-14)

The official [What's new in v2](https://py.sdk.modelcontextprotocol.io/whats-new/) and [ASGI mounting guide](https://py.sdk.modelcontextprotocol.io/run/asgi/) were rechecked on 2026-10-14. They confirm that v2 is the stable SDK line, uses `MCPServer` and the first-class `Client`, moves HTTP client integration to `httpx2`, supports the 2026-07-28 protocol while retaining compatibility with 2025-era clients, and requires the host lifespan to enter `mcp.session_manager.run()` when the MCP app is mounted. The ASGI guide also documents the Host allowlist and browser CORS configuration.

Protocol nuance: modern 2026-07-28 HTTP requests do not use `Mcp-Session-Id`; legacy 2025-11-25 clients still use session semantics unless served with `stateless_http=True`. The historical spike observed both paths, including stateless legacy calls. Its multiworker and browser CORS behavior remains untested.

This audit did not reinstall or rerun the disposable spike. The documented package/test results above are historical observations from the removed venv and script. In the current repository environment, Python is 3.12.3; `mcp`, FastAPI, `httpx`, `httpx2`, and `respx` are not installed. Therefore this refresh verifies current official documentation, not a fresh SDK runtime test.

## Prompt 0.2 claim status

| Claim | Status | Evidence and limit |
| --- | --- | --- |
| Disposable installation of stable SDK v2, FastAPI, and Uvicorn; exact resolved versions | VERIFIED (historical) | Install command, successful resolver output, and package versions are recorded above. PyPI/current SDK docs rechecked 2026-10-14 identify 2.3.0 as current stable. Spike environment was removed as required. |
| Python requirement and dependency metadata, including httpx2 | VERIFIED (historical) | `Requires-Python >=3.10` and complete captured `Requires-Dist` output are recorded above. Current SDK installation docs state Python 3.11+; architecture targets 3.13. The discrepancy between package metadata and current docs is disclosed, not treated as runtime compatibility proof. |
| `MCPServer`, structured output, `/mcp` mount, `/health`, and host lifespan | VERIFIED (historical) | Script description and observed memory/live HTTP output above show the class, structured content, route responses, and lifespan run context. The script itself is no longer available for independent replay. |
| Host allowlist and default 421 behavior | VERIFIED (historical) | Captured default 421 and allowed-host success are recorded under Integration and transport behavior. |
| Stateless HTTP and in-memory / real-HTTP clients | VERIFIED (historical) | Captured modern and legacy client results and session-header observations are recorded above. Multiworker behavior is UNVERIFIED. |
| Error handling | VERIFIED (historical) | ToolError, protocol error, and redacted unexpected exception outcomes were captured for memory and live HTTP. Traceback suppression is not claimed; a traceback was observed server-side. |
| Compatible HTTP mocking tools | VERIFIED (historical) | `httpx2.MockTransport` intercepted SDK transport and `respx` intercepted plain-httpx HN calls; `pytest-httpx` was installed but not exercised (UNVERIFIED). |
| Browser CORS/Origin, multiworker behavior, real HN adapter, and current repeatability | UNVERIFIED | Not exercised in the spike/current audit; the temporary spike source was removed. |
