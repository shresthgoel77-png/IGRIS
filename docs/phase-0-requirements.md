# Phase 0 authoritative requirements

Source: user-provided authoritative specification, 2026-10-14. This file restores the source against which Phase 0 is audited.

## Prompt 0.1 — Repository initialization
- Initialize Git with main and create one initial commit.
- Create only docs/phase-status.md, docs/architecture.md, docs/deployment.md, README.md, .gitignore, and MIT LICENSE.
- The tracker must contain exactly these nine fields: Current Phase, Completed Prompts, Verified Features, Tests Passing, Files Added, Files Modified, Deployment Decision, Known Issues, Next Phase.
- No application code, dependency files, or test directories.

## Prompt 0.2 — MCP SDK v2 verification
- In a disposable environment outside the repository, install the latest stable MCP SDK v2, FastAPI, and Uvicorn.
- Record exact versions, requires-python, dependencies including httpx2, and compatible HTTP mocking tools.
- Verify MCPServer, structured-output tools, mounting at /mcp, /health, host lifespan with mcp.session_manager.run(), Host allowlisting and the 421 behavior, stateless HTTP, in-memory and real-HTTP clients, and error handling.
- Record executed commands and observed outputs. Mark every claim VERIFIED or UNVERIFIED. Remove the disposable spike afterward.

## Prompt 0.3 — Hosting and upstream request budget
- Verify current official Cloudflare Workers/Python and Render Free limits, with URLs and check dates.
- Measure a few live Algolia search, small/large /items/{id} tree, and Firebase requests using curl; record status, response size, and latency.
- Calculate worst-case requests for 15 discussions and up to 50 comments each under Firebase per-comment fetching versus Algolia tree retrieval, including concurrency and retry assumptions.
- Compare hosting options, document Render spin-down/payment-card requirements where verifiable, and define time-boxed Cloudflare pass/fail gates. No deployment or account creation.

## Prompt 0.4 — Binding architecture
- Use verified findings from 0.2 and 0.3.
- Document module boundaries, architecture diagram, shared services, all REST routes (/, /health, /api/search, /api/discussions/{id}, /api/research) and MCP tools (search_hn, get_hn_discussion, research_hn).
- Define field-level schemas for HNResult, HNComment, HNDiscussion, ResearchEvidence, ResearchTheme, ResearchResult, and the error envelope.
- Specify input/output contracts and bounds: query 1–300 characters, search limit 1–30, comment limit 1–50, research limit 1–15.
- Document retrieval strategy, request budgets, concurrency, timeouts/retries, Python version, HTTP client/mock library, MCP stateless/session mode, transport security, and lifespan.
- Specify ranking/normalization, fixed-reference-time testing, testable acceptance criteria for Phases 1–10, and unresolved risks. Keep under approximately 600 lines.

## Six mandatory completion checks
1. Required repository scaffold, clean Git working tree, and nine-field tracker verified.
2. technical-validation.md contains executed commands, observed outputs, and per-claim verification statuses.
3. SDK Python requirement, httpx2, HTTP client, and mocking choice are documented and verified.
4. deployment.md contains sourced hosting limits and explicit arithmetic for both retrieval strategies.
5. architecture.md contains schemas, bounds, retrieval decisions, and testable Phase 1–10 acceptance criteria.
6. All blueprint changes and outstanding decisions are explicitly listed for my approval.

## Execution rules
Audit existing files first; preserve existing work; never fabricate evidence. Make only minimal documentation changes. Do not create application code, deploy, or start Phase 1. Update phase-status.md only when evidence supports completion. Phase 0 is complete only when all six checks are satisfied or any remaining approval gate is clearly disclosed.

## Deliverable
A requirement-by-requirement PASS/PARTIAL/FAIL/UNVERIFIED report, changed-file list, Git status, all four completed documents, and explicit approval requests.
