# TrustState Python SDK

[![Python](https://img.shields.io/badge/python-3.9%2B-blue)](https://www.python.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![TrustState API](https://img.shields.io/badge/TrustState-API%20v1-green)](https://truststate-api.apps.trustchainlabs.com)

Python SDK for **[TrustState](https://trustchainlabs.com)** — real-time compliance validation, policy enforcement, and immutable audit trails for AI agents, financial systems, and automated workflows.

---

## What is TrustState?

TrustState is a compliance infrastructure platform. It lets you define **schemas** (what your data must look like) and **policies** (what rules it must follow), then validate any event or decision against them in real time — with cryptographic proof of every check.

Use it to:
- Ensure AI agent outputs comply with your SOPs before delivery
- Validate financial transactions against regulatory rules
- Prove that every decision in an automated workflow was checked and logged

---

## Installation

```bash
# Install from GitHub (PyPI listing coming soon)
pip install git+https://github.com/jasimp18/truststate-py.git
```

Or clone and install locally:
```bash
git clone https://github.com/jasimp18/truststate-py.git
cd truststate-py
pip install -e .
```

**Requirements:** Python 3.9+, `httpx>=0.27`

---

## Getting an API Key

1. Log in to your TrustState dashboard at [tstate.apps.trustchainlabs.com](https://tstate.apps.trustchainlabs.com)
2. Go to **Settings → API Keys**
3. Click **New API Key**, give it a label (e.g. `"My AI Agent"`)
4. Copy the key — it is shown **only once**

---

## Quick Start

```python
import asyncio
from truststate import TrustStateClient

ts = TrustStateClient(api_key="ts_your_key_here")

async def main():
    result = await ts.check(
        entity_type="AgentResponse",
        data={
            "responseText": "Your loan application is under review.",
            "confidenceScore": 0.92,
            "disclaimer": "This is not financial advice.",
            "language": "en"
        }
    )

    if result.passed:
        print(f"✅ Compliant — Record ID: {result.record_id}")
    else:
        print(f"❌ Blocked — {result.fail_reason}")

asyncio.run(main())
```

---

## Core Concepts

Before using the SDK, set up your schema and policy in the TrustState dashboard:

| Concept | What it is | Example |
|---|---|---|
| **Entity Type** | The name for the type of data you're validating | `AgentResponse`, `Transaction` |
| **Schema** | Defines the required fields and their types | `confidenceScore` must be a number |
| **Policy** | Defines the business rules | `confidenceScore` must be ≥ 0.7 |
| **Record** | A passed validation — cryptographically signed | Stored in Registry |
| **Violation** | A failed validation | Stored in Violations log |

---

## Usage

### 1. Single check

```python
from truststate import TrustStateClient

ts = TrustStateClient(
    api_key="ts_your_key_here",
    base_url="https://truststate-api.apps.trustchainlabs.com",  # default
    default_schema_version="1.0",
    default_actor_id="agent-001",
)

result = await ts.check(
    entity_type="AgentResponse",
    data={
        "responseText": "Here is your account summary.",
        "confidenceScore": 0.88,
        "disclaimer": "For informational purposes only.",
        "language": "en"
    },
    action="CREATE",                  # optional, defaults to CREATE
    entity_id="session-abc-turn-1",  # optional, auto-generated if omitted
    schema_version="1.0",             # optional, uses default_schema_version
    actor_id="agent-001",             # optional, uses default_actor_id
)

print(result.passed)       # True / False
print(result.record_id)    # "3fa85f64-..." (if passed)
print(result.fail_reason)  # "Policy: min-confidence violated" (if failed)
print(result.failed_step)  # 8 (schema) or 9 (policy)
```

### 2. Batch check

Submit multiple items in a single call — efficient for high-volume agents or transaction feeds.

```python
results = await ts.check_batch(
    items=[
        {
            "entity_type": "AgentResponse",
            "data": {"responseText": "...", "confidenceScore": 0.9, "disclaimer": "...", "language": "en"},
            "entity_id": "session-001-turn-1"
        },
        {
            "entity_type": "AgentResponse",
            "data": {"responseText": "...", "confidenceScore": 0.4, "disclaimer": "...", "language": "en"},
            "entity_id": "session-002-turn-1"
        }
    ],
    default_schema_version="1.0",
    default_actor_id="agent-batch"
)

print(f"Batch: {results.accepted}/{results.total} passed")
for r in results.results:
    status = "✅" if r.passed else "❌"
    print(f"  {status} {r.entity_id} — {r.fail_reason or r.record_id}")
```

### 3. Verify a past record

Retrieve and verify a previously submitted record by its ID:

```python
proof = await ts.verify(
    record_id="3fa85f64-5717-4562-b3fc-2c963f66afa6",
    bearer_token="your_jwt_token"
)
print(proof)
```

---

## The `@compliant` Decorator

Wrap any async function so its return value is automatically submitted to TrustState before being returned to the caller. The function is blocked if compliance fails.

```python
from truststate import TrustStateClient, compliant

ts = TrustStateClient(api_key="ts_your_key_here")

@compliant(client=ts, entity_type="AgentResponse", action="CREATE")
async def generate_customer_response(customer_id: str, query: str) -> dict:
    # Your AI agent logic here
    response = await llm.generate(query)
    return {
        "responseText": response,
        "confidenceScore": 0.91,
        "disclaimer": "This is not financial advice.",
        "language": "en",
        "customerId": customer_id
    }

# Usage — TrustState check happens automatically
result = await generate_customer_response("cust-001", "What is my balance?")
```

**`on_fail` options:**

| Value | Behaviour |
|---|---|
| `"raise"` (default) | Raises `TrustStateError` if check fails |
| `"warn"` | Logs a warning, returns the value anyway |
| `"return_none"` | Returns `None` silently on failure |

```python
@compliant(client=ts, entity_type="AgentResponse", on_fail="warn")
async def generate_response(query: str) -> dict:
    ...
```

---

## FastAPI Middleware

Automatically validate all requests to your API against TrustState before they reach your route handler.

```python
from fastapi import FastAPI
from truststate import TrustStateClient, TrustStateMiddleware

app = FastAPI()
ts = TrustStateClient(api_key="ts_your_key_here")

app.add_middleware(
    TrustStateMiddleware,
    client=ts,
    entity_type_header="X-Compliance-Entity-Type",  # set this header on requests
    action_header="X-Compliance-Action",
)

@app.post("/agent/respond")
async def respond(body: dict):
    # TrustState has already validated the request body
    # If it failed, a 422 was returned before reaching here
    return {"status": "ok"}
```

Requests must include the header `X-Compliance-Entity-Type: AgentResponse` to trigger validation. Requests without the header pass through untouched.

---

## Mock Mode

Develop and test without connecting to the TrustState API. Zero network calls.

```python
ts = TrustStateClient(
    api_key="",
    mock=True,
    mock_pass_rate=1.0   # 1.0 = always pass, 0.0 = always fail, 0.8 = 80% pass
)

result = await ts.check("AgentResponse", {"responseText": "Test"})
print(result.passed)  # True
print(result.mock)    # True
```

Useful for:
- Unit tests without API credentials
- CI/CD pipelines
- Local development before schemas/policies are set up

---

## Error Handling

```python
from truststate import TrustStateClient, TrustStateError

ts = TrustStateClient(api_key="ts_your_key_here")

try:
    result = await ts.check("AgentResponse", data)
    if not result.passed:
        # Validation ran, but data failed policy/schema
        handle_violation(result.fail_reason, result.failed_step)
except TrustStateError as e:
    # HTTP-level error (auth failure, network issue, server error)
    print(f"TrustState error {e.status_code}: {e}")
```

**`failed_step` values:**

| Step | Meaning |
|---|---|
| `8` | Schema validation failed — data shape/type is wrong |
| `9` | Policy evaluation failed — data violated a business rule |

---

## Running the Examples

```bash
git clone https://github.com/jasimp18/truststate-py.git
cd truststate-py
pip install -e .

# AI agent demo (uses mock mode if no API key set)
export TRUSTSTATE_API_KEY="ts_your_key"
python examples/ai_agent_demo.py

# CIMB-style batch transaction feed
python examples/batch_feed.py
```

---

## Running Tests

```bash
pip install pytest pytest-asyncio
pytest tests/
```

---

## API Compatibility

| SDK Version | TrustState API |
|---|---|
| `0.1.x` | API v1 |

---

## Roadmap

- [ ] Publish to PyPI (`pip install truststate`)
- [ ] Schema caching (avoid re-fetching on every call)
- [ ] Django middleware
- [ ] Sync client (non-async) for simpler integrations
- [ ] LangChain callback integration

---

## Links

- **TrustState Platform:** [tstate.apps.trustchainlabs.com](https://tstate.apps.trustchainlabs.com)
- **API Reference:** [truststate-api.apps.trustchainlabs.com/docs](https://truststate-api.apps.trustchainlabs.com/docs)
- **JS/TS SDK:** [github.com/jasimp18/truststate-js](https://github.com/jasimp18/truststate-js)
- **TrustChain Labs:** [trustchainlabs.com](https://trustchainlabs.com)

---

## License

MIT — see [LICENSE](LICENSE)
