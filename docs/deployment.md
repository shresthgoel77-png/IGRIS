# Deployment feasibility and upstream request budget

Checked 2026-10-13 (Asia/Kolkata). This is a pre-deployment comparison for a strictly $0 Hacker News research website and remote MCP server. No account was created and nothing was deployed. **VERIFIED** means an official source was read or a stated command was run; **UNVERIFIED** means project-specific behavior still needs a platform test. Estimates below are models from a small sample, not latency guarantees.

## Official hosting constraints

All source rows were checked on **2026-10-13**. Each cited page supports the numbers in its row.

| Platform | Verified facts | Project implication |
| --- | --- | --- |
| Cloudflare Workers Free | **VERIFIED** [Limits](https://developers.cloudflare.com/workers/platform/limits/): 100,000 requests/day; 10 ms CPU/request; 128 MB per isolate; 50 external subrequests/invocation; up to 6 outgoing connections simultaneously waiting for response headers; 1 s startup; 64 MiB uncompressed Worker size. | A 16-request Algolia path can fit the subrequest count; CPU, memory, startup, and package fit remain **UNVERIFIED** for this app. The 6-connection rule concerns connections waiting for headers, not all open responses. |
| Cloudflare Workers Free | **VERIFIED** [CPU and duration](https://developers.cloudflare.com/workers/platform/limits/#cpu-time): waiting on network I/O does not count as CPU time. [HTTP duration](https://developers.cloudflare.com/workers/platform/limits/#duration) has no hard limit while the client remains connected; [subrequests](https://developers.cloudflare.com/workers/platform/limits/#subrequests) count redirects too. | Multi-second upstream waits do not alone fail the 10 ms CPU gate. Parsing and pruning large JSON trees may; Worker CPU was not measured (**UNVERIFIED**). |
| Python Workers | **VERIFIED** [Packages](https://developers.cloudflare.com/workers/languages/python/packages/): examples require Python `>=3.13`; PyPI packages must be pure Python or have PyEmscripten wheels. **VERIFIED** [FastAPI guide](https://developers.cloudflare.com/workers/languages/python/packages/fastapi/): FastAPI is supported through `workers.asgi.entrypoint(app)`. | FastAPI support does not prove that MCP SDK v2 and all dependencies run under Pyodide. Wheel resolution, lifespan, `httpx2`, and MCP streaming on Workers remain **UNVERIFIED**. |
| Render Free web service | **VERIFIED** [Free services](https://render.com/docs/free): 15 min without inbound traffic triggers spin-down; the next request takes about 1 min to spin up; 750 Free instance-hours per workspace per calendar month, with suspension after exhaustion. **VERIFIED** [Pricing](https://render.com/pricing): Free web-service compute has 512 MB RAM and costs $0/month. | Native Python is a simpler candidate for the locally tested FastAPI/MCP stack, but a roughly minute-long cold start may exceed an MCP client's patience (**UNVERIFIED** without a deployed test). |
| Render Free web service | **VERIFIED** [Free services](https://render.com/docs/free): local files are ephemeral; Free has one instance, and Render may suspend a Free service for unusually high service-initiated public-internet traffic, explicitly including external API calls. No numeric threshold is published there. | Use no local persistence. The outbound traffic threshold is a material **UNVERIFIED** operational risk for repeated research calls. |
| Render billing and signup | **VERIFIED** Render's [free-tier article](https://render.com/articles/platforms-with-a-real-free-tier-for-developers-in-2026) explicitly says no credit card is required; the [first-deploy guide](https://render.com/docs/your-first-deploy) says no payment is required. The [FAQ](https://render.com/docs/faq#all-of-my-services-run-on-free-instances-can-i-still-be-billed) says a workspace **with** a payment method can be billed for excess outbound bandwidth or build minutes; without one, services/builds are suspended or disabled instead. | A card is not required today according to published Render material, but signup-flow behavior was not tested (**UNVERIFIED**). For strict $0, use Free and no payment method; accept quota suspension. |

### Phase 0 source recheck (2026-10-14)

The official [Cloudflare limits](https://developers.cloudflare.com/workers/platform/limits/), [Python package support](https://developers.cloudflare.com/workers/languages/python/packages/), [Render Free](https://render.com/docs/free), and [Render compute plans](https://render.com/docs/compute-plans) pages were rechecked. The limits above remain supported by those sources. Cloudflare's Python package guide uses `requires-python >=3.13` and limits package support to pure-Python, PyEmscripten, or Pyodide-available packages. This does not establish that this project's MCP dependency set runs there.

The first additional Firebase sample attempt failed before connecting with `curl: (6) Could not resolve host: hacker-news.firebaseio.com` (status 000). The same read-only GETs were then executed with network access and succeeded. These are fresh observations, not latency guarantees:

| Request | Decoded JSON | `time_total` | `time_starttransfer` | Observed shape |
| --- | ---: | ---: | ---: | --- |
| Firebase `/v0/item/25254772.json` (small story) | 292 B | 0.951 s | 0.950 s | Story object, 1 direct `kids` ID. |
| Firebase `/v0/item/25254773.json` (comment) | 1,114 B | 0.713 s | 0.713 s | Comment object, no `kids`. |

Exact commands and observed outputs:

```sh
curl --compressed --max-time 20 --retry 0 -sS 'https://hacker-news.firebaseio.com/v0/item/25254772.json' -o /tmp/hn-deploy-firebase-small.json -w 'status=%{http_code} bytes=%{size_download} total=%{time_total} connect=%{time_connect} ttfb=%{time_starttransfer}\n'
curl --compressed --max-time 20 --retry 0 -sS 'https://hacker-news.firebaseio.com/v0/item/25254773.json' -o /tmp/hn-deploy-firebase-comment.json -w 'status=%{http_code} bytes=%{size_download} total=%{time_total} connect=%{time_connect} ttfb=%{time_starttransfer}\n'
```

```text
status=200 bytes=292 total=0.950678 connect=0.174290 ttfb=0.950399
status=200 bytes=1114 total=0.713027 connect=0.064011 ttfb=0.712690
```

The original DNS failure was environmental; it did not change repository files or produce an inferred measurement.

Render's Free page explicitly cautions against using free instances for production applications. Its current compute-plan table lists the Free web service at 0.1 CPU and 512 MB RAM; Free has one instance, ephemeral files, spin-down, and the documented service-initiated traffic suspension risk. The provisional Render recommendation must therefore be treated as a hobby/prototype hosting candidate, not evidence of production suitability. Signup flow, this application's memory/CPU use, MCP client cold-start behavior, and the external-API traffic suspension threshold remain untested.

Render's approximately one-minute spin-up is a documented expectation, not a measured cold start for this project. Cloudflare's 1 s number is a Worker startup **limit**, not measured response time. No service-level performance guarantee is inferred from either.

## Live upstream measurements

**VERIFIED** Six HTTP GETs were made with `curl` on 2026-10-13. `--compressed` was enabled; decoded JSON file sizes matched curl's `size_download`. Timings are curl `time_total` from this environment, including connection setup. All returned HTTP 200. These single observations can vary by network, cache, item size, and API load.

| Request | Decoded JSON | `time_total` | `time_starttransfer` | Observed shape |
| --- | ---: | ---: | ---: | --- |
| [Algolia search](https://hn.algolia.com/api/v1/search), `tags=story`, `num_comments>500`, 1 hit | 3,023 B | 0.801 s | 0.800 s | Story `11116274`, search count 967 comments. |
| [Algolia search](https://hn.algolia.com/api/v1/search), `tags=story`, `num_comments<5`, 1 hit | 1,224 B | 0.573 s | 0.572 s | Story `25254772`, search count 1 comment. |
| [Algolia search](https://hn.algolia.com/api/v1/search), `tags=story`, `num_comments>50`, 15 hits | 43,229 B | 0.694 s | 0.621 s | 15 results, search counts 363–3,859 comments. |
| [Algolia `/items/25254772`](https://hn.algolia.com/api/v1/items/25254772) | 832 B | 0.778 s | 0.778 s | Nested `children`: 1 descendant. |
| [Algolia `/items/11116274`](https://hn.algolia.com/api/v1/items/11116274) | 575,777 B | 2.428 s | 1.398 s | Nested `children`: 949 descendants, 202 direct children. |
| [Firebase `/v0/item/11116274.json`](https://hacker-news.firebaseio.com/v0/item/11116274.json) | 2,209 B | 0.953 s | 0.953 s | Story object with 225 direct `kids` IDs; no comment bodies. |

Exact curl commands executed (temporary JSON files were outside the repository):

```sh
curl --compressed --max-time 20 --retry 0 -sS -G 'https://hn.algolia.com/api/v1/search' --data-urlencode 'tags=story' --data-urlencode 'numericFilters=num_comments>500' --data-urlencode 'hitsPerPage=1' -o /tmp/hn-deploy-search-large.json -w 'status=%{http_code} bytes=%{size_download} total=%{time_total} connect=%{time_connect} ttfb=%{time_starttransfer}\n'
curl --compressed --max-time 20 --retry 0 -sS -G 'https://hn.algolia.com/api/v1/search' --data-urlencode 'tags=story' --data-urlencode 'numericFilters=num_comments<5' --data-urlencode 'hitsPerPage=1' -o /tmp/hn-deploy-search-small.json -w 'status=%{http_code} bytes=%{size_download} total=%{time_total} connect=%{time_connect} ttfb=%{time_starttransfer}\n'
curl --compressed --max-time 30 --retry 0 -sS 'https://hn.algolia.com/api/v1/items/25254772' -o /tmp/hn-deploy-item-small.json -w 'status=%{http_code} bytes=%{size_download} total=%{time_total} connect=%{time_connect} ttfb=%{time_starttransfer}\n'
curl --compressed --max-time 40 --retry 0 -sS 'https://hn.algolia.com/api/v1/items/11116274' -o /tmp/hn-deploy-item-large.json -w 'status=%{http_code} bytes=%{size_download} total=%{time_total} connect=%{time_connect} ttfb=%{time_starttransfer}\n'
curl --compressed --max-time 20 --retry 0 -sS 'https://hacker-news.firebaseio.com/v0/item/11116274.json' -o /tmp/hn-deploy-firebase-item.json -w 'status=%{http_code} bytes=%{size_download} total=%{time_total} connect=%{time_connect} ttfb=%{time_starttransfer}\n'
curl --compressed --max-time 20 --retry 0 -sS -G 'https://hn.algolia.com/api/v1/search' --data-urlencode 'tags=story' --data-urlencode 'numericFilters=num_comments>50' --data-urlencode 'hitsPerPage=15' -o /tmp/hn-deploy-search-15.json -w 'status=%{http_code} bytes=%{size_download} total=%{time_total} connect=%{time_connect} ttfb=%{time_starttransfer}\n'
```

Observed curl output in the same order (exact seconds; rounded table values above):

```text
status=200 bytes=3023 total=0.800635 connect=0.132547 ttfb=0.800286
status=200 bytes=1224 total=0.572921 connect=0.052234 ttfb=0.572431
status=200 bytes=832 total=0.777769 connect=0.042834 ttfb=0.777587
status=200 bytes=575777 total=2.427612 connect=0.089838 ttfb=1.397643
status=200 bytes=2209 total=0.953259 connect=0.185717 ttfb=0.952912
status=200 bytes=43229 total=0.693511 connect=0.044726 ttfb=0.621203
```

The high-comment Algolia search reported 967 comments, while `/items` JSON had 949 nested descendants; Firebase listed 225 direct `kids`, whereas Algolia showed 202 direct children. **VERIFIED** these sampled responses differed. The cause and whether Algolia omits comments needed by this product are **UNVERIFIED**. Do not call the endpoint a guaranteed complete mirror of Firebase without a data-quality check.

## Worst-case `research_hn` request budget

Assumptions: one Algolia search returns 15 story IDs; analyze up to 50 comments per story; no cache hits, redirects, or extra metadata calls. For Firebase, fetch each story once to obtain `kids`, then fetch 50 comment objects per story. For Algolia, fetch `/items/{id}` once per story, then select at most 50 comments locally from its nested tree. The latter downloads the entire tree; no endpoint-level 50-comment limit was observed. All counts are for **one inbound research request**, compared with Cloudflare's [50-subrequest Free limit](https://developers.cloudflare.com/workers/platform/limits/#subrequests).

| Strategy | First attempts | With one retry per GET | Cloudflare 50? | Sample-based serial wall clock | Idealized 6-wide wall clock |
| --- | ---: | ---: | --- | ---: | ---: |
| Firebase individual comments | `1 + 15 + (15 × 50) = 766` | `1,532` | **No**, both exceed 50; even omitting story fetches gives `1 + 750 = 751`. | `0.693511 + 15×0.953259 + 750×0.713027 = 549.763 s` (~9.2 min). | `0.693511 + ceil(15/6)×0.953259 + ceil(750/6)×0.713027 = 92.682 s` (~1.5 min; dependency stages honored). |
| Algolia nested items | `1 + 15 = 16` | `32` | **Yes** with max retries, `32 ≤ 50`; 18 calls remain before redirects/other calls. | `0.693511 + 15×2.427612 = 37.108 s`. | `0.693511 + ceil(15/6)×2.427612 = 7.976 s` (3 waves). |

The retry column models at most one retry on every GET (two attempts each); it does not assume that retries are used. Wall-clock columns model successful first attempts only and are **calculated**, not observed end-to-end. Firebase serial time uses the 15-hit Algolia search latency (`0.693511 s`), the large Firebase story latency (`0.953259 s`), and the new Firebase comment latency (`0.713027 s`). Six-wide Firebase time honors the dependency stages: story lookups precede comment lookups. Algolia uses the large tree sample (`2.427612 s`). Point estimates assume equal-duration calls per stage, no contention/upstream throttling, no retries, and no JSON processing. Cloudflare's published rule concerns connections waiting for response headers, so six is a conservative design cap rather than a measured throughput ceiling. Render has no six-connection figure in the sources used here; the same cap is shown only for comparison.

At the sampled large-tree size, `43,229 + (15 × 575,777) = 8,679,884 B` (~8.28 MiB) of decoded JSON would be transferred before selecting 50 comments from each tree. This is a **calculated sample scenario**, not a maximum payload. CPU and peak memory for 15 trees on either host are **UNVERIFIED**. The one-retry upper bound leaves 18 of Cloudflare's 50 subrequests for redirects/other calls on the Algolia path; disabling redirects and avoiding extra calls is part of the design.

## Decision matrix and Phase 2 plan

| Criterion | Cloudflare Workers Free | Render Free |
| --- | --- | --- |
| Strict $0 published path | **VERIFIED** free limits above; operational fit **UNVERIFIED**. | **VERIFIED** no card required in published Render material; no-card quota exhaustion suspends service/builds. Signup flow **UNVERIFIED**. |
| Python/FastAPI | **VERIFIED** FastAPI guide supports Python Workers; MCP SDK dependencies and mounted lifespan there **UNVERIFIED**. | **VERIFIED** Render [FAQ](https://render.com/docs/faq#which-languages-does-render-support) supports Python; local FastAPI/MCP integration passed in [technical validation](technical-validation.md). Hosted behavior **UNVERIFIED**. |
| Upstream count | Algolia `16 ≤ 50`; Firebase `766 > 50` by calculation. | No per-request subrequest quota was identified in the [Free docs](https://render.com/docs/free), but unusually high service-initiated API traffic can cause suspension; safe threshold **UNVERIFIED**. |
| Latency and availability | Actual Python Worker startup and client latency **UNVERIFIED**. | **VERIFIED** documented 15-min idle spin-down and ~1-min cold start; real MCP-client timeout behavior **UNVERIFIED**. |
| Resource fit | **VERIFIED** 10 ms CPU, 128 MB memory, 1 s startup, 64 MiB bundle limits; app fit **UNVERIFIED**. | **VERIFIED** 512 MB RAM and 750 free hours/month; project load and traffic threshold **UNVERIFIED**. |

**Recommendation:** use Algolia search plus one `/items/{id}` call per selected story as the primary retrieval strategy, with an explicit 15-story bound and local selection of at most 50 comments per tree. Treat missing or inconsistent Algolia comments as a quality risk, and keep any Firebase fallback bounded so it cannot silently become 750 per-comment calls. Render Free is the provisional native-Python hosting baseline; Cloudflare Workers Free remains conditional on a short compatibility experiment. This is a documentation decision, not a deployment or architecture change.

In Phase 2, time-box a Cloudflare Python Worker experiment to **one working day** (proposed project budget, not a provider limit). Stop at the first hard incompatibility rather than building workarounds that change the product architecture. Run the actual dependency set on Python `>=3.13`, mount FastAPI and MCP, exercise modern and legacy SDK clients, and measure the full 15-story path. The ten gates below were supplied for Prompt 0.3. Every gate is **UNVERIFIED on Cloudflare** until an authorized experiment executes it.

| Gate | Phase 2 pass criterion | Fail criterion / evidence to capture |
| --- | --- | --- |
| 1. Python runtime compatibility | Worker builds and runs on Python `>=3.13`; health request succeeds. | Runtime/import failure; capture build and request logs. |
| 2. Official MCP Python SDK v2 installation | A stable `mcp>=2,<3` resolves, bundles, and imports in the Worker. | Resolver, wheel, bundle, or import failure; record package and version. |
| 3. SDK dependency compatibility | Required packages, including `httpx2` and compiled dependencies, import and complete a real MCP call. | Unsupported pure/PyEmscripten wheel or runtime API; record dependency chain. |
| 4. FastAPI/ASGI execution | `workers.asgi.entrypoint(app)` serves `/health` and enters the host lifespan. | ASGI routing or lifespan failure; capture request and startup logs. |
| 5. MCP Streamable HTTP integration | `/mcp` responds with valid protocol traffic with the session manager active and a correct Host allowlist. | 421, 5xx, stalled session manager, or invalid transport response. |
| 6. MCP tool discovery | Official SDK client `tools/list` finds the expected tool via public Worker URL. | Empty, malformed, or failed listing; capture JSON-RPC exchange. |
| 7. MCP tool invocation | Official SDK client `tools/call` returns expected structured content and known tool errors. | Failed call or wrong result/error shape. |
| 8. JSON-RPC, streaming, request lifecycle | Modern and legacy flows complete; streaming and shutdown paths finish without stuck requests. | Malformed JSON-RPC, broken stream, missing lifespan, or cancellation failure. |
| 9. CPU, memory, subrequest, startup limits | Full scenario stays within [published limits](https://developers.cloudflare.com/workers/platform/limits/): CPU ≤10 ms/request, memory <128 MB/isolate, subrequests ≤50, at most 6 connections waiting for headers, startup ≤1 s, bundle ≤64 MiB. Record CPU, wall time, peak memory, request count, and bundle size. | Any provider limit exceeded; an unavailable metric leaves the gate **UNVERIFIED**, not passed. |
| 10. Public HTTPS and real client | Public HTTPS URL completes `tools/list` and `tools/call` from the official SDK client, including Host/security configuration. | TLS, routing, Host, client compatibility, or cold-start timeout failure. |

Open risks: Algolia tree completeness and response size; unmeasured Worker CPU/memory and Python dependency support; Render's undocumented outbound-traffic suspension threshold; Render cold starts against real MCP client timeouts. No platform is production-validated yet.
