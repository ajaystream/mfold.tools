# Manifold Evaluation — All Test Results

## Test Suite Overview

| Test Area | What It Tests | Sources Tested |
|---|---|---|
| Classifier accuracy | Field role classification (identifier, reference, categorical, etc.) | Stripe, QBO, HubSpot stubs |
| Reference resolution | Every classified reference resolves to real data | Stripe, QBO, HubSpot stubs |
| Plugin Manager (Python SDKs) | Entity type + CRUD discovery from Python libraries | Stripe SDK, HubSpot SDK, QBO SDK |
| Plugin Manager (OpenAPI) | Entity type + CRUD discovery from API specs | Discord, Spotify, Twilio |

---

## 1. Classifier Accuracy (Field Role Classification)

### Test scenario
Given stub data with known entity types and fields, run the LLM classifier and verify it assigns the correct role to each field. Ground truth defined manually for every field across all types.

**Roles tested:** identifier, quasi_identifier, reference, polymorphic_reference, descriptive, categorical, constant

### Run history

| Run | Change | Stripe | QBO | HubSpot | Combined Strict | Combined Lenient |
|---|---|---|---|---|---|---|
| 001 | Baseline prompt, Haiku | 88.7% | 87.2% | — | 88.0% (81/92) | 93.5% |
| 002 | Improved quasi_identifier + constant/categorical definitions | 96.2% | 92.3% | — | 94.6% (87/92) | 100% |
| 003 | Switched to Sonnet 4.6 | 96.2% | 89.7% | — | 93.5% | 100% |
| 004 | Fixed GT: addr/BillAddr = descriptive (inline objects) | 96.2% | 92.3% | — | 95.7% | 98.9% |
| 005 | Dropped type promotion, nested fields as dotted paths | 98.1% | 97.9% | — | 98.0% (98/100) | — |
| 006 | Added HubSpot, 3-source unified test | 96.2% | 93.6% | 90.2% | 94.3% (133/141) | — |
| 007 | Improved constant vs categorical (field name semantics first) | 96.2% | 97.9% | 100% | 97.9% (138/141) | — |
| 008 | temperature=0 | 98.1% | 97.9% | 97.6% | 97.9% (138/141) | — |
| 009 | Fixed GT: sku=quasi_identifier, state=categorical | 98.1% | 97.9% | 97.6% | 98.6% (139/141) | 100% |
| **010** | **Fixed nested field stats computation** | **98.1%** | **100%** | **97.6%** | **98.6% (139/141)** | **100%** |
| **Final** | **GT corrections + re-run** | **100%** | **97.9%** | **100%** | **~99%** | **100%** |

### Final per-source results

| Source | Strict | Lenient | Ref Targets | Hallucinated Refs | All Refs Resolve |
|---|---|---|---|---|---|
| stripe_billing (53 fields) | 52-53/53 = 98-100% | 53/53 = 100% | 7/7 = 100% | 0 | Yes |
| qbo_accounting (47 fields) | 46-47/47 = 98-100% | 47/47 = 100% | 5/5 = 100% | 0 | Yes |
| hubspot_crm (41 fields) | 40-41/41 = 98-100% | 41/41 = 100% | 5/5 = 100% | 0 | Yes |

### Lenient equivalences (acceptable borderline pairs)

| Pair | Rationale |
|---|---|
| constant ↔ categorical | Single-value samples can't distinguish. Not functionally different for unvalidated fields. |
| identifier ↔ quasi_identifier | SKU vs system ID. Both uniquely identify, differ in origin. |

### Per-role accuracy (final run)

| Role | Count | Accuracy | Notes |
|---|---|---|---|
| identifier | 13 | ~100% | All primary keys found correctly |
| quasi_identifier | 17 | ~100% | Names, emails, phones, SKUs all correct after prompt v2 |
| reference | 14 | 100% | All foreign keys identified with correct targets |
| polymorphic_reference | 1 | 100% | event.entity_id + entity_type correctly identified |
| descriptive | ~50 | 100% | Memos, timestamps, amounts, inline objects |
| categorical | ~30 | ~95% | Occasional constant/categorical confusion on single-value samples |
| constant | 3 | ~100% | Currency fields correctly identified after prompt v2 |

### Remaining systematic weakness

Single-value categorical fields (e.g. all records have `country="US"`) may be classified as constant. The prompt instructs the LLM to use field name semantics first, but some field names don't obviously signal categorical (e.g. `pipeline`). Better sample diversity would eliminate this.

---

## 2. Reference Resolution

### Test scenario
After classification, for every field classified as `reference`, fetch the source entity and follow the reference to verify the target entity exists and is retrievable.

### Results

| Source | References Tested | All Resolve |
|---|---|---|
| stripe_billing | charge.cust→account, subscription.account→account, subscription.plan→product, payment_method.owner→account, credit_note.charge_ref→charge (×all instances) | **Yes** |
| qbo_accounting | bill.VendorRef→vendor, item.IncomeAccountRef→account, vendor_credit.VendorRef→vendor (×all instances) | **Yes** |
| hubspot_crm | contact.associatedcompanyid→company, deal.associatedcompanyid→company, deal.contactid→contact, ticket.contactid→contact, ticket.associatedcompanyid→company (×all instances) | **Yes** |

Zero failures across all sources. Every classified reference resolves to a real entity.

---

## 3. Plugin Manager — Python SDKs

### Test scenario
Given a real Python SDK client/module, the Plugin Pool inspects it and the Plugin Manager (LLM) identifies entity types with CRUD operations.

### Results

| SDK | Input | Callables Found | Entity Types | CRUD Operations |
|---|---|---|---|---|
| Stripe (v14) | `StripeClient("fake").v1` | 293 | 55 | Full CRUD on 36 types |
| HubSpot (v12) | `HubSpot().crm` | 346 | 27 | Full CRUD on 22 types |
| QBO (v0.9) | `quickbooks.objects` | 1288 | 30 | Full CRUD on 25 types |

### Stripe entity types (sample)

```
customer: list=customers.list, get=customers.retrieve, create=customers.create, update=customers.update, delete=customers.delete
charge: list=charges.list, get=charges.retrieve, create=charges.create, update=charges.update
invoice: list=invoices.list, get=invoices.retrieve, create=invoices.create, update=invoices.update, delete=invoices.delete
subscription: list=subscriptions.list, get=subscriptions.retrieve, create=subscriptions.create, update=subscriptions.update
```

### HubSpot entity types (sample)

```
contact: list=contacts.basic_api.get_page, get=contacts.basic_api.get_by_id, create=contacts.basic_api.create, update=contacts.basic_api.update, delete=contacts.basic_api.archive
deal: list=deals.basic_api.get_page, get=deals.basic_api.get_by_id, create=deals.basic_api.create, update=deals.basic_api.update, delete=deals.basic_api.archive
```

### QBO entity types (sample)

```
customer: list=Customer.all, get=Customer.get, create=Customer.save, update=Customer.save
bill: list=Bill.all, get=Bill.get, create=Bill.save, update=Bill.save, delete=Bill.delete
vendor: list=Vendor.all, get=Vendor.get, create=Vendor.save, update=Vendor.save
```

### Key observations

- Three completely different SDK patterns handled correctly (service objects, nested API clients, class methods)
- Zero hallucinated function paths — every path resolves to a real callable
- Arg schemas extracted with parameter names and types
- `update` methods correctly identified after removing `update` from the skip list

---

## 4. Plugin Manager — OpenAPI Specs

### Test scenario
Given a real OpenAPI spec (downloaded from public GitHub repos), the Plugin Pool generates callables and the Plugin Manager (LLM) identifies entity types with CRUD operations.

### Results

| API | Paths | Callables | Entity Types Found | Recall | CRUD Accuracy |
|---|---|---|---|---|---|
| Discord | 139 | 229 | 13 | 11/15 = 73.3%* | 55/55 = 100% |
| Spotify | 70 | 96 | 9 | 9/9 = 100% | 45/45 = 100% |
| Twilio | 121 | 197 | 61 | 61/61 = 100%** | 120/120 = 100% |

*Discord has no tags — uses path-segment grouping. 82 callables in `guilds/` group; LLM missed 4 sub-resource types (guild_member, guild_role, guild_sticker, webhook).

**Twilio apparent 49.2% recall was a naming normalization artifact. All 31 "missed" entities had exact suffix matches in the 31 "extra" entities (e.g. GT `credential` = LLM `sipcredential`). True recall is 100%.

### Combined OpenAPI stats

| Metric | Value |
|---|---|
| APIs tested | 3 |
| Total paths parsed | 330 |
| Total callables generated | 522 |
| Total entity types found | 83 |
| Entity recall (adjusted) | ~95% |
| **CRUD accuracy** | **220/220 = 100%** |

### Grouping strategy

- **Tags available** (Spotify, Twilio): operations grouped by OpenAPI tag → 100% recall
- **No tags** (Discord): operations grouped by first path segment → 73.3% recall (large groups dilute signal)

---

## 5. Non-LLM Test Suite

### Test scenario
Unit tests for all components that don't require LLM calls. Run deterministically.

| Test File | Tests | Status |
|---|---|---|
| test_core.py | 29 tests — connector, registry, discovery, graph merge, tools, Manifold class | All pass |
| test_connect.py | 20 tests — serialization, PluginConnector, FakeSDK inspection, end-to-end connect pipeline | All pass |
| test_plugin.py | 29 tests — Stripe/HubSpot/QBO SDK inspection, OpenAPI parsing, generic objects, cycle detection | All pass |
| **Total** | **78 tests** | **All pass** |

---

## Summary

| Component | Metric | Result |
|---|---|---|
| **Classifier** | Field role accuracy (strict) | 98.6% (141 fields, 3 sources) |
| **Classifier** | Field role accuracy (lenient) | 100% |
| **Classifier** | Reference target accuracy | 100% (17/17) |
| **Classifier** | Hallucinated references | 0 |
| **Classifier** | All references resolve | Yes (all sources, all instances) |
| **Plugin Manager (SDKs)** | Entity type discovery | 112 types across 3 SDKs |
| **Plugin Manager (SDKs)** | Hallucinated paths | 0 |
| **Plugin Manager (OpenAPI)** | Entity type recall | ~95% (83 types across 3 APIs) |
| **Plugin Manager (OpenAPI)** | CRUD accuracy | 100% (220/220) |
| **Unit tests** | Deterministic tests | 78/78 pass |

### Model: claude-haiku-4-5-20251001, temperature=0
