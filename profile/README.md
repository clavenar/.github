<p align="center">
  <img src="./clavenar-mark-256.png" alt="Clavenar" width="96" height="96">
</p>

# Clavenar

**Your AI agents are privileged users. Treat them like one.**

AI agents hold API keys, move money, and touch production data — yet they're governed by none of the controls you'd put on a human with that access. An agent with a hallucinated tool call and a real API key is indistinguishable from a malicious insider, except faster. By the time a person notices the agent is wiring funds or dropping tables, the action has already cleared upstream.

**Clavenar is the control plane between your agents and production.** It's an mTLS-fronted MCP (Model Context Protocol) proxy that inspects every tool call before any side effect, parks the risky ones for human approval, and hash-chains the lot into a forensic ledger your auditor can replay.

> Inspect every tool call. Approve the risky ones. Prove all of it happened.

**Who it's for:** engineers and security leads running autonomous or semi-autonomous agents against production systems — CRMs, finance APIs, infrastructure, customer data — where someone with a clipboard cares what the agent did last Tuesday at 4 p.m. Most teams land here after a prompt-injection scare, a credential near-miss, a runaway loop, or a regulator asking how AI decisions get reviewed. Not yet for pre-prototype experimentation or read-only agents on a single laptop — come back when the agent gets a credit card or a database write.

---

## Architecture — the four-layer model

Every agent request flows through four independent layers. Each can veto; each leaves a record.

| Layer | Component | Role |
|-------|-----------|------|
| **L1 — Data Plane** | proxy | Single mTLS MCP ingress (`:8443`). Terminates mTLS, parses the SPIFFE SAN, runs L2 then L3 to completion, gates Yellow-tier at HIL, mints/redeems A2A actor tokens, forwards upstream. Everything else is defense-in-depth behind it. |
| **L2 — Semantic Evaluation** | brain | `POST /inspect` on the hot path: intent classification, persona-drift, indirect-injection, malicious-code, and compromised-package detection (Haiku-backed). Stateless; fail-open, since L3/L4 retain independent veto/record. |
| **L3 — Governance** | policy-engine | Pure-Rust Rego evaluator (regorus) over `policies/*.rego`. The deterministic governance anchor with independent veto and a per-agent velocity tracker. Sandboxed, no host bridge. |
| **L4 — Forensic Store** | ledger | SHA-256 hash-chained, SQLite-backed append-only audit. Subscribes to `clavenar.forensic` on NATS JetStream; `GET /verify` walks the chain, so any single-row edit invalidates every later entry. |

### The wire path

An agent connects over mTLS to **L1** (`:8443`), which terminates the connection and parses the SPIFFE SAN. On every tool-call request it runs **L2** (`Brain POST /inspect`, semantic verdict) then **L3** (Rego governance) to completion, deriving the final verdict from `authorized && policy_decision.allow` — **fail-closed**. Yellow-tier calls (wires, prod writes, mass emails) hold at human-in-the-loop for approve/deny before `forward_upstream` fires. The proxy then publishes a forensic event over NATS (`clavenar.forensic`) that **L4** appends to its hash-chained store, keyed on a UUIDv4 `correlation_id` and verifiable via `GET /verify`.

---

## Start here

**See it work in minutes — no control plane to stand up:**

- [**Hosted demo**](https://demo.clavenar.com) — fire curated attack scenarios at a live stack and watch the verdict, the human-approval gate, and the hash-chained ledger build in your browser.
- [**clavenar.com**](https://clavenar.com) — what it is, who it's for, the editions, and the compliance story.

**Run it yourself:**

- [**clavenar-lite**](https://github.com/clavenar/clavenar-lite) — single-binary OSS edition. A drop-in MCP proxy that inspects requests, evaluates Rego policy, and writes a SHA-256 hash-chained forensic ledger without the multi-service control plane.
- [**clavenar-shadow-scanner**](https://github.com/clavenar/clavenar-shadow-scanner) — free 10-minute discovery tool. Scans GitHub orgs, Slack workspaces, and local filesystems for leaked agent credentials (AI provider, cloud, CI/deploy, dev-platform, database, messaging) with redacted / JSON / SARIF output.
- [**clavenar-charts**](https://github.com/clavenar/clavenar-charts) — Helm charts + Terraform modules for sidecar deployment on AWS, GCP, and Azure.

**Integrate your agents:**

- [**clavenar-ai-py**](https://github.com/clavenar/clavenar-ai-py) — Python SDK. Wrap your async Anthropic / OpenAI client; every tool call is inspected before it runs.
- [**clavenar-ai-sdk**](https://github.com/clavenar/clavenar-ai-sdk) — TypeScript SDK. Wrap your Anthropic client; every `tool_use` is inspected by `clavenar-lite` before your code runs it.
- [**clavenar-sdk**](https://github.com/clavenar/clavenar-sdk) — async Rust SDK. Typed client over the proxy `POST /mcp` and ledger audit/verify endpoints; consumed by the console, the CLI, and external integrators.
- [**clavenar-ctl**](https://github.com/clavenar/clavenar-ctl) — operator CLI (binary `clavenarctl`). Thin client over `clavenar-sdk` for ledger queries, HIL decisions, and chain verification.

**Read the contracts:**

- [**clavenar-specs**](https://github.com/clavenar/clavenar-specs) — `TECH_SPEC.md`, the source of truth for every wire contract across the `clavenar-*` repos. Read this before integrating.
- [**clavenar-chaos-catalog**](https://github.com/clavenar/clavenar-chaos-catalog) — pure-data attack catalog driving the red-team and demo flows; the corpus the hosted demo fires at the proxy.

> The control-plane services (proxy, brain, policy-engine, ledger, HIL, identity, console) are not yet public. The repos above are the public building blocks and integration surface.

---

## Why teams land here

- **Inspect every tool call before any upstream side effect** — a five-signal Brain (intent classification, persona-drift, indirect-injection, malicious-code, and compromised-package detection) plus pure-Rust Rego policy and a per-agent velocity tracker.
- **Human-in-the-loop for the dangerous bits** — Yellow-tier tools (wires, prod writes, mass emails) park as Pending and wait for an approver's approve/deny decision; expired requests fail closed.
- **Cryptographic proof, not log scraping** — every verdict, approval, and outcome is written in canonical JSON and SHA-256 hash-chained. Tamper a byte and `/verify` tells you exactly which row broke.
- **Maps to the frameworks your auditor opens with** — signed verdicts, chained transitions, and regulatory export bundles on demand. Three editions: a free Shadow Scanner, OSS `clavenar-lite`, and the full multi-layer control plane.

---

### Inspect. Approve. Prove.
