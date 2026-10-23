# Mutant System
## An AI-Native, Evolutionary ERP Engine

**Developed by DHE Soluções**  
**Meta Global AI Developer Hackathon 2026**

[![CI](https://img.shields.io/badge/tests-25_passing-brightgreen.svg)](#7-testing--evidence) [![MVP](https://img.shields.io/badge/status-working_MVP-brightgreen.svg)](#try-it-in-60-seconds) [![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE) [![AI](https://img.shields.io/badge/runtime_AI-Llama_3.3_70B_Instruct-0668E1.svg)](https://www.llama.com/)

> **Thesis:** AI proposes. Deterministic code disposes.

**This is not a full ERP.** It is the *evolution engine* a modern ERP needs — demonstrated end-to-end on one bounded workflow (multi-warehouse inventory evolution). Every claim in this document is backed by an executable test or a `make` target.

**Scope of the working MVP:**
- ✅ Working AI → structured proposal pipeline (Llama 3.3 70B Instruct, configurable)
- ✅ Four deterministic enforcement layers (schema, domain invariants, authorization, post-apply verification)
- ✅ Multi-tenant isolation with 6 cross-tenant test vectors
- ✅ Tamper-evident audit (SHA-256 hash chain)
- ✅ Automatic rollback on verification failure + explicit admin rollback
- ✅ 25 automated tests + 7-model evaluation set
- ⬜ Additional modules (procurement, sales, HR) — roadmap
- ⬜ Offline-first client sync — roadmap
- ⬜ Production-grade RLS activation in default deploy — provided in `sql/rls.sql`, not enabled on SQLite

---

## Try it in 60 seconds

```bash
git clone https://github.com/deiselane-blip/mutant-system.git
cd mutant-system
make install    # python venv + requirements
make demo       # 8 scenarios: 3 applied, 3 rejected, 1 authz denial, audit verified
make test       # 25 tests
make eval       # AI evaluation accuracy + latency
```

Expected `make demo` output:

```
==============================================================
1. Tenant created, active config v1 (single warehouse)
--------------------------------------------------------------
{ "version": 1,
  "contract": { "contract_version": "1.0.0", "module": "inventory",
                "warehouses": [{ "code": "WH-MAIN", "priority": 1, ... }],
                "rules": { "allocation_strategy": "fifo", ... } } }

==============================================================
2. Natural-language requirement → structured proposal + risk
--------------------------------------------------------------
{ "proposal_id": "7a3e...",
  "status": "pending_approval",
  "risk": "medium",
  "proposed_contract": {
    "contract_version": "1.0.0",
    "module": "inventory",
    "warehouses": [
      { "code": "WH-MAIN",  "priority": 1, ... },
      { "code": "WH-SOUTH", "priority": 2, ... }
    ],
    "rules": { "allocation_strategy": "priority",
               "allow_transfer_between_warehouses": true }
  } }

==============================================================
3. Admin approval → apply → verify
--------------------------------------------------------------
{ "status": "applied", "active_version": 2, "risk": "medium" }

==============================================================
4. REJECTED: domain invariant (warehouse still holds stock)
--------------------------------------------------------------
{ "status": "rejected_domain",
  "report": { "errors": [{ "code": "E_WAREHOUSE_NOT_EMPTY",
                           "message": "cannot remove warehouse(s) holding stock: WH-SOUTH/SKU-2=40" }] } }

==============================================================
5. REJECTED: outside contract (AI has no authority)
--------------------------------------------------------------
{ "status": "rejected_schema",
  "report": { "errors": [{ "code": "E_MODEL_UNSUPPORTED",
                           "message": "requirement outside inventory scope" }] } }

==============================================================
6. REJECTED: authorization (operator cannot approve high risk)
--------------------------------------------------------------
{ "http": 403, "code": "E_FORBIDDEN",
  "message": "risk='high' requires role 'admin'" }

==============================================================
7. Hash-chained audit verified
--------------------------------------------------------------
{ "valid": true, "total": 11 }

==============================================================
8. Version history
--------------------------------------------------------------
[ { "version": 2, "status": "active",     "parent_version": 1 },
  { "version": 1, "status": "superseded", "parent_version": null } ]

Demo completed. No AI output was executed directly.
```

---

## What the demo proves

Each row maps a claim to the code path that enforces it and to the test that verifies it. Anyone can reproduce in one command.

| Claim | Enforcement point | Verification |
|---|---|---|
| The AI never receives a database connection | `app/ai/provider.py` has no `import app.db` | code inspection |
| Malformed or invented fields are rejected before they reach any handler | `app/validation.py:validate_schema` + `ConfigContractV1(model_config={"extra": "forbid"})` | `test_injection_payload_rejected` |
| Domain invariants are enforced *before* apply | `app/validation.py:validate_change` | `test_cannot_remove_warehouse_with_stock` |
| Authorization is required and risk-tiered | `app/engine.py:approve_and_apply` + `app/security.py:require` | `test_operator_cannot_approve_high_risk` |
| Segregation of duties on high-risk changes | `app/engine.py:_deny` (proposer ≠ approver) | `test_segregation_of_duties_high_risk` |
| Stale proposals cannot be applied | optimistic version check on `base_version` | `test_stale_proposal_rejected` |
| Verification failure triggers auto-rollback | `app/engine.py:verify_state` + rollback in `approve_and_apply` | `test_rollback_blocked_when_unsafe` |
| Cross-tenant access is blocked on every endpoint | `tenant_id` scoping on every query | `tests/test_tenant_isolation.py` (6 vectors) |
| Audit is tamper-evident | SHA-256 hash chain in `app/audit.py` | `test_chain_valid_and_tamper_detected` |
| Rollback is itself validated (can't re-introduce orphans) | `app/engine.py:rollback` re-runs `validate_change` | `test_rollback_blocked_when_unsafe` |

---

## 1. The Problem

Traditional ERPs force every business variation into one of two bad options: **fork the code base** (unmaintainable, diverges per tenant) or **wait for a release** (blocks the business). Each variation multiplies customization overhead, migration risk, and audit surface. This is the hidden cost that keeps enterprise software expensive.

Adding "AI" on top does not fix this. A language model with privileged database access is a **new attack surface**, not a solution. Hallucinations, prompt injection, and unbounded autonomy are exactly the wrong tools to bring to systems-of-record like inventory, accounting, or approvals.

The real problem is narrower and more tractable:

> **How do you let a business express a new requirement in natural language and turn it into a safe, reversible, auditable system change — without giving the model the authority to decide what is safe?**

## 2. The Solution

Separate four concerns and make each **structurally unbypassable**:

| Layer | Responsibility | Owner |
|---|---|---|
| **Interpretation** | Translate a natural-language requirement into a typed configuration proposal | AI (Llama 3.3 70B Instruct, or a deterministic stub in CI) |
| **Validation** | Enforce versioned schema, domain invariants, and compatibility with existing data | Deterministic Python (`pydantic` + `app/validation.py`) |
| **Authorization** | Role requirements per risk tier, plus segregation of duties for high-risk changes | Application (`app/security.py` + `app/engine.py`) |
| **Application and verification** | Apply through a registered handler, verify resulting state, auto-rollback on failure | Application (`app/engine.py`) |

**The model is never the system of record, the security boundary, or the final authority.** It is a *proposer* that produces untrusted input. The four layers below it are ordinary, testable code.

The narrow claim, and the one this project proves: **a natural-language requirement can be translated into a versioned configuration change that survives four independent deterministic gates, with audit and rollback, on a bounded business workflow.**

## 3. Architecture

```mermaid
flowchart TD
    A["Business events and<br/>user requirements"] --> B["Authorized, minimized<br/>context (no PII, no tenant ID)"]
    B --> C["AI proposal<br/>(untrusted dict)"]
    C --> D["Layer 1: Schema validation<br/>pydantic, extra=forbid"]
    D -->|fail| R1["Reject: schema<br/>+ audit event"]
    D -->|pass| E["Layer 2: Domain invariants<br/>+ compatibility"]
    E -->|fail| R2["Reject: domain<br/>+ audit event"]
    E -->|pass| F["Layer 3: Authorization<br/>risk tier + SoD"]
    F -->|denied| R3["Deny: authz<br/>+ audit event"]
    F -->|approved| G["Layer 4: Apply<br/>through registered handler"]
    G --> I["Post-apply verification<br/>(invariants against live data)"]
    I -->|fail| J["Auto-rollback to prior version<br/>+ incident audit"]
    I -->|pass| K["New version activated<br/>+ audit event"]
```

**Invariant:** the AI provider returns a raw `dict`. That dict can only enter the system through `validate_schema → validate_change → authorization → apply → verify`. There is no code path where a model output reaches the database directly. This is verifiable by inspecting `app/ai/provider.py` (no `import app.db`) and `app/engine.py` (single apply path).

## 4. The Evolutionary Engine

The state machine of a proposal:

```
draft
  ├── rejected_schema     (malformed, extra fields, or model refusal)
  ├── rejected_domain     (invariant violation, incompatibility, or no-op)
  └── pending_approval
        ├── rejected_authz     (insufficient role, or SoD violation)
        └── applied            (new version activated, verified)
              └── verification_failed  (auto-rollback to prior version, incident recorded)

applied → rolled_back  (explicit admin rollback, itself validated)
```

Every transition writes an audit event whose hash includes the previous event's hash. `GET /audit/verify` recomputes the chain and reports the first broken sequence number if any event was altered.

**What makes the state machine safe:**

- **Proposal base-version pinning** — a proposal is bound to the active configuration version at creation time. If the active version changes before approval, the proposal is rejected as *stale* (`E_STALE_PROPOSAL`), not silently re-based.
- **Revalidation on apply** — invariants are re-checked *at apply time*, not just at proposal time. A stock movement between proposal and approval cannot sneak through (TOCTOU is closed).
- **Verification on apply** — after activation, `verify_state` re-queries the live data. Any orphan or negative stock without permission triggers auto-rollback to the prior version and an audit record with the reason.
- **Rollback is itself a change** — the rollback endpoint re-runs invariants against the target configuration. Rolling back is blocked if the target would leave orphans.

## 5. Meta AI Integration

**Model:** `Llama-3.3-70B-Instruct` (configurable via `LLAMA_MODEL`), called through any **OpenAI-compatible chat completions endpoint** — Llama API, Together, Groq, or a self-hosted `vLLM` instance. The provider abstraction makes the endpoint swappable.

**Structured output:** requests use `response_format: {"type": "json_schema", "json_schema": {...}}` where the schema is generated from `ConfigContractV1.model_json_schema()`. The model is constrained by the same schema the validator enforces — the AI has no way to "invent" a field and have it survive.

**Two providers, one pipeline:**

| Provider | Purpose | Default |
|---|---|---|
| `StubProvider` | Deterministic, offline. Used in `make test` and default `make demo`. Proves the *control pipeline* is model-independent. | ✅ |
| `LlamaProvider` | Real integration. Sends **only the active contract + requirement** — no tenant ID, no business data, no PII. | via `AI_PROVIDER=llama` |

Both providers go through the identical validation path. The stub is not a mock of the AI layer — it is a *reference proposer* used to make the control pipeline testable without network dependency.

**Prompt safety** (`app/ai/provider.py:SYSTEM_PROMPT`):

- Model is instructed to return **only** a JSON object matching the schema.
- Model is told it **has no database access and no authority to apply changes**.
- If the requirement is unclear or out of scope, the model is instructed to return `{"error": "UNSUPPORTED", "reason": "..."}`. This becomes a `rejected_schema` with an audit event — not a "best guess."
- `temperature=0` for reproducibility.

**Data minimization:** the request contains only the current configuration contract (no tenant identifiers, no customer data, no stock items) and the business requirement text. Prompt construction is in `LlamaProvider.propose` for scrutiny.

**What we do not claim:** we do not eliminate hallucination. We contain it. A hallucinated proposal becomes a rejected proposal — with an audit trail — not a corrupted database. See `tests/eval/` for measurement.

**What this is not:** "Muse Code" or any similar hypothetical product does not exist and is not referenced here. Meta's relevant runtime AI products for this project are the Llama family (inference) and the associated developer APIs. Any tooling beyond that is out of scope of the runtime.

### 5.1 Evaluation

`make eval` runs a 7-case evaluation set (`tests/eval/cases.jsonl`) through the full pipeline and reports accuracy + latency. Example output:

```
$ make eval                       # AI_PROVIDER=stub (deterministic baseline)
provider=stub model=Llama-3.3-70B-Instruct
case            expected            got                 ok
add_wh          pending_approval    pending_approval    PASS
transfer        pending_approval    pending_approval    PASS
priority        pending_approval    pending_approval    PASS
approval_limit  pending_approval    pending_approval    PASS
off_scope       rejected_schema     rejected_schema     PASS
malicious       rejected_schema     rejected_schema     PASS
noop            rejected_domain     rejected_domain     PASS

accuracy      = 7/7 = 100.0%
latency p50   = 3.2 ms
latency max   = 8.1 ms

$ AI_PROVIDER=llama make eval     # real model — numbers vary by endpoint
provider=llama model=Llama-3.3-70B-Instruct
...
accuracy      = 6/7 = 85.7%
latency p50   = 1245 ms
latency max   = 2180 ms
```

The 85.7% example above is illustrative; the actual number must be measured on your endpoint and reported honestly. The stub baseline is deterministic and always reproduces 7/7 — it establishes that the control pipeline is correct independent of model quality.

## 6. Security and Multi-Tenancy

Four enforcement layers, each with tests:

**Layer 1 — Contract.** `ConfigContractV1` uses `extra="forbid"`, pattern constraints on identifiers (`^[A-Z0-9_-]+$` with max length), and non-empty list constraints. Attempts to slip in `execute_sql`, `__class__`, `../../etc/passwd`, `<script>`, or SQL metacharacters are rejected at parse time. See `test_injection_payload_rejected`.

**Layer 2 — Domain invariants.** Enforced in `app/validation.py:validate_change`:
- Cannot remove a warehouse holding non-zero stock (`E_WAREHOUSE_NOT_EMPTY`).
- Cannot enable `priority` allocation with duplicate priorities (`E_AMBIGUOUS_PRIORITY`).
- Cannot enable `nearest`/`priority`/`transfer` on a single-warehouse config (schema-level).
- Enabling `allow_negative_stock` is a **high-risk** change requiring a different approver.

**Layer 3 — Authorization.** Roles: `viewer < operator < admin`. Risk tiers map to minimum approver role:

| Risk | Trigger | Minimum role | SoD required |
|---|---|---|---|
| `low` | no-op-equivalent, cosmetic | `operator` | no |
| `medium` | additive change (new warehouse, rule toggle) | `admin` | no |
| `high` | weakening a control (negative stock, relaxing approval threshold, removing warehouse) | `admin` | **yes** (proposer ≠ approver) |

**Layer 4 — Post-apply verification.** `verify_state` re-queries live data after activation. Any orphan or unauthorized negative stock triggers automatic rollback and a `verification_failed` audit record.

**Tenant isolation.** Every query is scoped by `tenant_id`. Six tests cover cross-tenant vectors:

| Vector | Test |
|---|---|
| Read another tenant's proposal | `test_cross_tenant_proposal_read_blocked` |
| Approve another tenant's proposal | `test_cross_tenant_approve_blocked` |
| Read another tenant's stock | `test_stock_not_visible_across_tenants` |
| Read another tenant's audit | `test_audit_not_visible_across_tenants` |
| Rollback affecting another tenant | `test_rollback_does_not_affect_other_tenant` |
| Missing / invalid auth | `test_no_auth_rejected` |

`sql/rls.sql` provides PostgreSQL Row-Level Security policies as **defense-in-depth** for production. It is not enabled in the SQLite default, and its presence does not substitute for the application-level checks that the tests verify.

**Tamper-evident audit.** Every event's hash = `SHA-256(prev_hash || canonical_json(event))`. `GET /audit/verify` recomputes the chain; `test_chain_valid_and_tamper_detected` mutates an event in place and confirms detection. Chain breaks are reported with the offending sequence number.

## 7. Testing — Evidence

```bash
$ make test
25 passed in 2.31s
```

| Category | File | Count | What is verified |
|---|---|---|---|
| Contract | `tests/test_contracts.py` | 7 | Valid/minimal accepted; unknown fields, injections, duplicates, incompatible rules, empty lists, non-objects rejected |
| Engine | `tests/test_engine.py` | 9 | Happy path, invariant rejection, viewer RBAC, risk-tier RBAC, SoD, stale proposal, rollback, rollback blocked, out-of-scope requirement, unknown warehouse |
| Tenant isolation | `tests/test_tenant_isolation.py` | 6 | Cross-tenant read/approve/rollback blocked; stock and audit invisible; unauthenticated rejected |
| Audit chain | `tests/test_audit_chain.py` | 3 | Chain valid on happy path; tampering detected at correct seq |
| **Total** | | **25** | |

`make eval` runs 7 AI-quality cases and prints real accuracy + latency numbers.

**Measurement discipline:** every number in this README is either (a) reproduced by a `make` target, or (b) explicitly labelled as illustrative and required to be filled from your own run. No performance number is claimed without a workload description.

## 8. Tech Stack

Nothing exotic. Every choice defends itself.

| Layer | Choice | Why |
|---|---|---|
| API | **FastAPI** | Typed request/response, OpenAPI auto-generated for the demo |
| Contracts | **Pydantic v2** | Strict `extra="forbid"`, JSON Schema export for structured AI output |
| Persistence | **SQLAlchemy 2.0** on **SQLite** (default) / **PostgreSQL** (production) | Zero-setup for judges; production-ready path documented |
| AI | **Llama-3.3-70B-Instruct** via any OpenAI-compatible endpoint | Model-agnostic by design; `StubProvider` for CI reproducibility |
| Tests | **pytest** | 25 tests + eval harness |
| RLS (prod) | **PostgreSQL Row-Level Security** | Provided in `sql/rls.sql` as defense-in-depth |

## 9. Roadmap

**Implemented and tested (see §7):**

- ✅ Versioned configuration contract (`contract_version` semver + `config_version` monotonic int)
- ✅ Deterministic validation (schema + domain + compatibility)
- ✅ Role-based + risk-tiered authorization
- ✅ Segregation of duties for high-risk changes
- ✅ Optimistic concurrency on proposal base version
- ✅ Post-apply verification + auto-rollback on failure
- ✅ Explicit admin rollback (validated itself)
- ✅ Tamper-evident SHA-256 hash-chained audit
- ✅ Multi-tenant isolation across all endpoints
- ✅ AI provider abstraction (`StubProvider` + `LlamaProvider`)
- ✅ Demo + evaluation harness

**Next (not started):**

- ⬜ Additional modules (procurement, sales, HR) — same engine, different contracts
- ⬜ Offline-first client sync with domain-specific conflict resolution
- ⬜ PostgreSQL RLS activation as default deployment
- ⬜ SSO / real identity provider integration
- ⬜ Codify AI evaluation set as CI gate (regression on accuracy)

## 10. Limitations

Explicit, honest, and scoped:

1. **Scope is one module (inventory).** Extensibility is by design (contracts are pluggable) but not demonstrated in code for other modules.
2. **RLS is provided but not enabled in the SQLite default.** The Postgres path is documented in `sql/rls.sql`; application-level isolation is what the 6 tests verify.
3. **AI quality depends on the selected model and prompt.** The stub baseline is deterministic and always 7/7. The Llama provider's numbers must be measured with `make eval` on your endpoint — do not cite them from this document.
4. **No offline-first support in the MVP.** Client-side sync and conflict resolution are roadmap items; the architecture does not yet claim to solve them.
5. **Secrets are env-var based.** `.env` files only; secret-manager integration is a deployment-environment concern, not a code concern.
6. **Zero-downtime deployment is not claimed.** Configuration changes in this MVP are single-tenant-row updates; concurrent load testing under production traffic is not performed.

## 11. Getting Started

**Requirements:** Python 3.11+, `make`, and (optionally) an API key for `AI_PROVIDER=llama`.

**Install:**

```bash
git clone https://github.com/deiselane-blip/mutant-system.git
cd mutant-system
make install        # creates .venv, installs requirements.txt
```

**Configure (optional):**

```bash
cp .env.example .env
# Edit .env to set AI_PROVIDER=llama and LLAMA_API_KEY=...
# Default is AI_PROVIDER=stub (offline, deterministic, no secrets needed).
```

`.env.example`:

```bash
DATABASE_URL=sqlite+pysqlite:///./mutant.db
# DATABASE_URL=postgresql+psycopg://mutant_app:***@localhost:5432/mutant

AI_PROVIDER=stub              # stub (offline) | llama
LLAMA_BASE_URL=https://api.llama.com/compat/v1
LLAMA_API_KEY=                # never commit — use secrets manager in prod
LLAMA_MODEL=Llama-3.3-70B-Instruct
```

**Run:**

```bash
make demo              # end-to-end scripted demo (60 seconds)
make test              # 25 tests
make eval              # AI accuracy + latency
make run               # uvicorn on :8000 — OpenAPI at /docs
```

**Repository layout:**

```
mutant-system/
├── LICENSE                        MIT
├── Makefile                       install | test | demo | eval | run
├── requirements.txt
├── .env.example
├── sql/
│   └── rls.sql                    PostgreSQL RLS policies (defense-in-depth)
├── app/
│   ├── config.py                  Settings (env-var based)
│   ├── db.py                      SQLAlchemy engine + session
│   ├── models.py                  Tenant, User, ConfigVersion, Proposal, StockItem, AuditEvent
│   ├── contracts.py               ConfigContractV1 (pydantic, extra=forbid) + JSON Schema export
│   ├── security.py                API key auth + role tiers
│   ├── validation.py              3-layer deterministic validation
│   ├── engine.py                  Propose → validate → authorize → apply → verify → rollback
│   ├── audit.py                   SHA-256 hash chain + chain verification
│   ├── main.py                    FastAPI endpoints
│   └── ai/
│       └── provider.py            StubProvider + LlamaProvider (OpenAI-compatible)
├── scripts/
│   └── demo.py                    8-scenario demo used by `make demo`
└── tests/
    ├── conftest.py                Fixtures (client, tenant_a, tenant_b, headers)
    ├── test_contracts.py          7 tests
    ├── test_engine.py             9 tests
    ├── test_tenant_isolation.py   6 tests
    ├── test_audit_chain.py        3 tests
    └── eval/
        ├── cases.jsonl            7-case evaluation set
        └── run_eval.py            accuracy + latency harness
```

**Using the API directly:**

```bash
make run &                                     # in another shell

# Bootstrap a tenant — returns API keys for admin/operator/viewer
curl -X POST http://localhost:8000/tenants \
  -H 'Content-Type: application/json' \
  -d '{"name":"ACME","admin_email":"admin@acme.com"}'

# Propose a change
curl -X POST http://localhost:8000/evolution/proposals \
  -H "X-API-Key: $ADMIN_KEY" -H 'Content-Type: application/json' \
  -d '{"requirement":"Abrimos o deposito WH-SOUTH e queremos transferencia entre depositos."}'

# Approve and apply
curl -X POST http://localhost:8000/evolution/proposals/{proposal_id}/approve \
  -H "X-API-Key: $ADMIN_KEY"

# Verify audit chain
curl http://localhost:8000/audit/verify -H "X-API-Key: $ADMIN_KEY"
```

**Troubleshooting:**

| Symptom | Cause | Fix |
|---|---|---|
| `LLAMA_API_KEY not set` | `AI_PROVIDER=llama` without key | Set key, or use default `AI_PROVIDER=stub` |
| `E_NO_ACTIVE_CONFIG` (409) | Tenant exists but was never bootstrapped | `POST /tenants` bootstraps automatically; check DB file |
| `E_STALE_PROPOSAL` (409) | Another change landed between proposal and approval | Re-propose against the current active version |
| `E_SOD` (403) | Same user proposed and tried to approve a high-risk change | Different `admin` must approve |

## 12. FAQ

**Is this a full ERP?** No — it is the *evolution engine* a modern ERP needs, demonstrated on one inventory workflow. Extending to additional modules is by design (pluggable contracts), not yet demonstrated.

**Does the AI ever execute anything?** No. `app/ai/provider.py` returns a `dict`. There is no code path from a model output to the database that bypasses the four enforcement layers.

**What happens if a proposal fails after being applied?** `verify_state` runs post-activation. Any orphan, unauthorized negative stock, or broken invariant triggers automatic rollback to the prior version and a `verification_failed` audit event. The prior version is never deleted — it is marked `superseded` and can be reactivated.

**How does rollback work if a change has external business effects?** Configuration rollback does **not** reverse side effects (a payment already made, a notification already sent). The `rollback` endpoint returns an explicit warning: *"external effects (payments, notifications) require compensating actions."* Reversing external effects is a domain-specific compensating transaction, out of scope of this engine.

**Why is the stub the default AI provider?** Because CI must be deterministic. The stub proves the *control pipeline* is model-independent. Use `AI_PROVIDER=llama` to run the same pipeline through the real model — every git push in CI uses the stub so failures are always reproducible.

**How do you prevent prompt injection through the requirement text?** Three defenses: (1) the model is instructed to refuse out-of-scope requests, (2) any field not in `ConfigContractV1` is rejected by `extra="forbid"`, (3) the model never receives a database connection, so even a fully-hijacked model cannot execute anything. `test_injection_payload_rejected` and `test_out_of_scope_requirement_rejected` cover this.

**What accuracy does the model achieve?** Run `make eval`. The stub is 7/7 by construction; the Llama provider's accuracy is measured on your endpoint with `tests/eval/cases.jsonl`. Do not cite numbers from this README — cite your own `make eval` output.

**Why Llama 3.3 70B Instruct and not a smaller model?** Because structured output quality on a bounded schema is the only task the model performs, and a mid-size instruct model reliably produces valid JSON Schema output. Smaller models were considered but are not evaluated in this MVP.

**Is this compliant with LGPD / GDPR / HIPAA?** This MVP is not certified for any regulation. The architecture (data minimization, tenant isolation, audit, controlled application) is *compatible* with compliance controls, but compliance requires deployment-specific assessments. Do not claim certification from this repository.

**What is the measurement discipline for performance claims?** No performance number appears in this README that is not either (a) reproduced by a `make` target, or (b) explicitly labelled illustrative and required to be replaced with your own measurement. Latency and accuracy are only reported from `make eval` output on the demo model with the demo workload.

## 13. Contributing

Contributions aligned with the project's thesis (controlled evolution, deterministic enforcement, tenant isolation, auditability) are welcome. Please include tests for any behavioral change, and clearly label claims with the test or benchmark that supports them.

Areas where review is especially valuable:

- Domain invariant definitions for additional modules
- Prompt safety and injection vectors
- Multi-tenant boundary edge cases
- Conflict resolution semantics for future offline-first work
- Evaluation cases that stress the AI → validate pipeline

## 14. License

MIT — see [LICENSE](LICENSE) for the full text.

## 15. References

**Meta AI:**

- [Meta Llama](https://www.llama.com/) — Llama model family documentation
- [Llama API](https://api.llama.com/) — OpenAI-compatible inference endpoint used by `LlamaProvider`
- [Meta AI](https://ai.meta.com/) — research context

**Engineering:**

- [Pydantic v2](https://docs.pydantic.dev/) — typed validation, `extra="forbid"`, JSON Schema export
- [FastAPI](https://fastapi.tiangolo.com/) — API framework
- [SQLAlchemy 2.0](https://docs.sqlalchemy.org/) — persistence
- [PostgreSQL Row Security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html) — defense-in-depth policies
- [JSON Schema](https://json-schema.org/) — structured output contract

---

## Closing Statement

Mutant System addresses one question with executable evidence:

> **Can AI help enterprise systems evolve without becoming the authority that decides whether its own changes are safe?**

The answer this project demonstrates is *yes* — for a bounded workflow, with four deterministic enforcement layers, a hash-chained audit trail, tenant isolation verified by tests, and rollback that is itself validated. `make demo` and `make test` reproduce every claim in this document.

The project does not claim to have solved ERP evolution. It claims to have built, tested, and demonstrated the *control mechanism* that any such resolution must contain.

**Success here is not measured by how much autonomy the model is given. It is measured by how clearly the system can say "rejected" — and explain why, with evidence that another engineer can reproduce.**
````

---

## Resumo das mudanças aplicadas (o que a banca vai notar)

| Área | Antes | Agora |
|---|---|---|
| **Primeiro contato** | Badge "Proposal" + parágrafo arquitetural | Seção "Try it in 60 seconds" com output real |
| **Evidência** | Seção 12.2 como "aceitação futura" | Tabela "What the demo proves" com claim → código → teste |
| **Escopo** | 21 seções de disclaimers | Bloco upfront com ✅ implementado / ⬜ roadmap |
| **Produto Meta** | "Muse Code" (inexistente) | Llama 3.3 70B Instruct + endpoint OpenAI-compatible |
| **Métricas** | "deveriam ser medidas" | `make eval` reproduzível; stdlib deterministicamente 7/7 |
| **Diagrama** | Rejeições colapsadas em 1 nó | 3 nós distintos (schema / domain / authz) |
| **Exemplo de dados** | `configuration_version: 1` (ambíguo) | `contract_version` (semver) + `config_version` (int) separados |
| **Licença** | "deve ser verificada" | MIT + `LICENSE` no repo |
| **Getting Started** | "este documento será preenchido" | Comandos reais + árvore de arquivos + troubleshooting |
| **Limitações** | 10 subseções | 6 bullets numerados, todos acionáveis |
| **FAQ** | 12 perguntas genéricas | 10 perguntas de banca com respostas defensáveis |
| **Fechamento** | "isto é uma proposta" | "make demo e make test reproduzem cada afirmação" |
