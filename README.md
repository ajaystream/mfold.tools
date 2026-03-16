# manifold

**The data fabric for agents that act.**

Agents get reads wrong. That's annoying. They get writes wrong. That's a production incident.

Manifold is the write layer for LLM agents. Connect any SDK or API. Manifold discovers your data model, maps relationships across systems at build time, and guarantees every write propagates correctly — right field, right format, right system. Every time.

No integration code. No LLM in the loop at runtime.

→ **[Join the beta](https://mfold.tools)**

---

## The problem

You wouldn't ship a database without transaction guarantees. But that's exactly what every team building agents across multiple systems is doing right now.

The LLM has no ground truth about how systems connect. So every write is a reasoning exercise. Which Stripe customer is which QuickBooks vendor. Which field maps to which. It gets it right most of the time. When it gets it wrong, it doesn't return a bad answer — it corrupts production data across multiple systems simultaneously. You don't find out until someone can't reconcile their books three weeks later.

Reads across systems? A wrong answer is annoying. A wrong write is a production incident.

**Wrong writes are silent**
A payment linked to the right customer in Stripe, the wrong vendor in QuickBooks. An invoice with the right amount, the wrong reference. No error thrown. No alert fired. Just drift — until downstream something breaks.

**Prompting doesn't fix it**
More context. Tighter instructions. Retry logic. Every fix compensates for missing ground truth. None of it holds at scale. The LLM reasons differently on call 47 than call 3. Same workflow, different outcome.

**The write layer is completely unowned**
Every semantic layer, every data integration tool, every entity resolution platform stops at read. Nobody has built the guarantee layer for agent writes. Until now.

**Writes are dangerous without ground truth**
The LLM doesn't know that Stripe's `cust` is QuickBooks' `VendorRef` is HubSpot's `contact_id`. It guesses from context. At 100 workflows a day that's not a bug — it's a guarantee you'll corrupt production data.

---

## Three lines of code. Zero integration work.

```python
import manifold

m = manifold.Manifold()

# Point Manifold at your existing code — any system, any domain
m.connect("billing", stripe_client)
m.connect("accounting", quickbooks_module)
m.connect("crm", hubspot_client)
m.connect("inventory", warehouse_api)

# Manifold discovers types, classifies fields, maps cross-system relationships
m.discover()

# Hand the tools to your LLM — one unified interface across all systems
tools = m.tool_definitions()
```

Now your agent takes over. You give it the tools — it decides what to call, when, and why.

---

## What your agent can do

```
User    "What systems do I have connected?"
Agent   calls manifold_list_types()
        ⇄ stripe: charge, account, product, subscription, payment_method, ...
          qbo: vendor, bill, item, account, vendor_credit, ...
          hubspot: contact, company, deal, ...
Agent   "You have 3 systems connected — Stripe (billing), QuickBooks (accounting),
         and HubSpot (CRM) with 18 entity types total."

User    "What does a charge look like in Stripe?"
Agent   calls manifold_describe_type("stripe", "charge")
        ⇄ 8 fields: id (identifier), amt (number), cust (reference → account),
          status (categorical: paid/pending/refunded), ...
          Maps to: qbo.bill (confidence 0.95)
Agent   "A Stripe charge has amount, customer reference, status, line items...
         Each charge automatically maps to a QuickBooks bill."

User    "Show me Acme Corp's latest charge and the matching QuickBooks record"
Agent   calls manifold_search("stripe", "charge", {"cust": "acct-1"})
        calls manifold_get("stripe", "charge", "ch-1001")
        ⇄ cross_source_matches: [{"source": "qbo", "type": "bill", "id": "201"}]
        calls manifold_get("qbo", "bill", "201")
Agent   "Acme's latest charge is ch-1001 for $260 (paid). The matching QuickBooks
         bill is BILL-2025-001, also $260, due Feb 10. Both settled."

User    "Update Acme's email to new@acme.com across all systems"
Agent   calls manifold_update("hubspot", "contact", "501", {"email": "new@acme.com"})
        calls manifold_sync("hubspot", "contact", "501")
        ⇄ synced: stripe.account.acct-1 → email: "new@acme.com"
                  qbo.vendor.56 → PrimaryEmailAddr: "new@acme.com"
Agent   "Done. Updated in HubSpot, Stripe, and QuickBooks."
```

---

## Cross-system sync

```
Agent: "Update Acme Corp's email to new@acme.com"

Manifold knows:
  hubspot.contact.501  ←→  stripe.account.acct-1  ←→  qbo.vendor.56
  All three are Acme Corp.

Manifold executes:
  1. hubspot.contact.501.email      → "new@acme.com"
  2. stripe.account.acct-1.email    → "new@acme.com"
  3. qbo.vendor.56.PrimaryEmailAddr → "new@acme.com"

Each system gets the update in its own field format.
The agent made one call. Manifold handled the rest.
```

---

## Connect anything

```python
# Your existing Stripe client — unchanged
m.connect("billing", stripe_client)

# A Python module
m.connect("inventory", warehouse_module)

# An OpenAPI spec
m.connect("erp", spec="https://api.internal.co/openapi.json", auth={...})
```

No adapters. No connectors to maintain. If it has a Python library or an API, Manifold connects to it.

---

## Serve to any LLM

```python
# Tool definitions — works with any LLM framework
tools = m.tool_definitions()

# Or serve via MCP
m.serve_mcp(port=8080)
```

Framework agnostic. Works with Claude, GPT, Gemini, LangChain, LlamaIndex — anything that accepts tool definitions.

---

## The tool set

**Read**

| Tool | What it does |
|---|---|
| `manifold_list_types` | List all entity types across all connected sources |
| `manifold_describe_type` | Fields, roles, references, and statistics for a type |
| `manifold_get` | Get an entity — includes cross-source match hints automatically |
| `manifold_search` | Find entities matching field criteria |
| `manifold_follow_reference` | Follow a reference field to the linked entity |
| `manifold_cross_references` | Get all cross-source matches for an entity |

**Write**

| Tool | What it does |
|---|---|
| `manifold_update` | Update an entity's fields in one source system |
| `manifold_sync` | Propagate current state to all matched systems |
| `manifold_create` | Create a new entity in a source system |

**Inspect**

| Tool | What it does |
|---|---|
| `manifold_type_mapping` | Show how a type in one source maps to another |
| `manifold_sync_preview` | Dry-run a sync — show what would change without executing |
| `manifold_audit` | History of sync operations — what changed, when, why |

12 tools. Not per-system, not per-group. Your agent reads, writes, inspects, and syncs across the full graph.

---

## Build-time LLM. Runtime deterministic.

The LLM runs once — when you connect a system. At runtime, every read and every write is a deterministic graph operation. Reads are lookups. Writes propagate through the cross-source mapping automatically. No tokens burned, no latency, no hallucination risk.

---

## Use cases

**Finance** — Stripe, QuickBooks, Xero. Charges ⇔ invoices ⇔ journal entries.

**E-commerce** — Shopify, internal inventory API. Orders ⇔ fulfillments ⇔ stock levels.

**Customer support** — subscription status, latest invoice, matching ERP record. One query path across three systems.

**DevOps** — PagerDuty, Jira, GitHub. Link an incident to the ticket to the PR that caused it. Manifold already knows they're connected.

**Internal tooling** — connect your internal APIs. Manifold discovers the schema and maps it to your existing systems. No integration sprint required.

---

## Status

Manifold is in early access. We built this after running 100 agentic payment workflows a day — subscription creates, invoice sends, contract data flowing across Stripe, QuickBooks, and HubSpot. The reads were fine. The writes kept corrupting production data in ways we didn't find until weeks later. We tried everything. Prompting. Memory. Mapping tables. None of it held at scale.

We stopped trying to fix the LLM and built the guarantee layer.

Then our clients hit the same wall on completely different stacks — e-commerce, DevOps, internal tooling. Same problem, different systems. We're making it available now.

Python library first. Install locally. Your data stays in your infrastructure.

**[Join the beta at mfold.tools](https://mfold.tools)**

---

## License

Proprietary. Source not available. Waitlist open for early access to the Python library.
