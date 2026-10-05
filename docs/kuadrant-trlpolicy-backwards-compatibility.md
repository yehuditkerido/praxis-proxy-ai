# Kuadrant `TRLPolicy` backward compatibility

**Status:** discussion draft for issue [#844](https://github.com/praxis-proxy/ai/issues/844)

This document is a design proposal for discussion. It does not implement a translator, change the Praxis filter, or open a production migration path.

## Why this issue exists

PR [#796](https://github.com/praxis-proxy/ai/pull/796) introduced a Praxis token rate limiting model that differs from Kuadrant's existing `TokenRateLimitPolicy` (called `TRLPolicy` in the issue). The compatibility question is how existing customers can keep their current policy and quota behavior after Praxis adds its implementation.

The Slack discussion adds an important requirement: compatibility includes counters and reservations already stored by Limitador in Redis. It also suggests evaluating whether extending or reusing Limitador is safer for production than creating a second storage and reservation model. The referenced Kuadrant design work also makes shared token usage fields, such as JSON paths, valuable to integrations such as MaaS Subscription.

The issue does not select an approach yet. The `Approach` section is empty, so this document should establish the gaps and the options before implementation begins.

## What “backward compatible” must cover

Compatibility has four layers. Passing one layer does not prove the others.

### 1. Resource and configuration compatibility

Can a customer keep applying the existing `TokenRateLimitPolicy` CRD?

A translator can satisfy this layer. A controller or deployment tool can read the existing CRD and produce the internal configuration Praxis needs. The customer-facing resource does not change, but the translator must define behavior for all meaningful fields, including:

- `targetRef` and `sectionName`
- route and request matching
- `limits`, multiple `rates`, and `counters`
- `when` expressions
- reservation settings
- response token extraction
- unsupported or ambiguous fields

Copying only a limit and a window is not sufficient. The translator must reject or report fields it cannot preserve.

### 2. Behavioral compatibility

Do Kuadrant and Praxis make the same observable decision for the same request and policy?

Differential tests can provide evidence for this layer. The same fixtures should run against both systems and compare:

- allow or reject decision, HTTP status, and headers
- selected route, policy rule, identity, and counter key
- quota remaining before and after a request
- reservation and final settlement
- window expiration and multiple rates
- concurrent requests
- upstream failures, missing usage, and infrastructure failures
- JSON and streaming responses
- supported Kuadrant and Limitador versions

A successful request is only a happy-path check. The compatibility matrix must include boundary, failure, and concurrency cases.

### 3. Data-at-rest compatibility

Do existing Limitador/Redis counters, TTLs, and active reservations continue to mean the same thing?

This is separate from configuration translation and behavior tests. Praxis must either read and update the existing Limitador state or document an explicit state strategy: migration, drain and expiry, temporary compatibility storage, or an intentional reset.

Using the same Redis server, database, or prefix does not prove compatibility. Limitador and Praxis currently use different key formats and different time-window representations.

### 4. Product integration compatibility

Can products above the limiter use one stable policy contract?

For example, MaaS Subscription could expose one token-usage JSON path. An adapter would map it to Kuadrant `dataExtraction` or Praxis `token_count`. The product integration should not need to know which enforcement engine is active.

This layer does not require Kuadrant and Praxis to enforce the same request at the same time. It requires a stable integration contract while the implementation underneath changes.

## Current gaps to investigate

| Area | Current gap | Risk | Evidence needed |
| --- | --- | --- | --- |
| CRD ingestion | Praxis has its own configuration and no confirmed `TRLPolicy` translator/controller. | Existing customers must rewrite policy or lose fields. | A field-by-field mapping, validation behavior, and generated Praxis configuration. |
| Targeting | Kuadrant attaches policy to `Gateway`/`HTTPRoute`; Praxis filters run for a selected `filter_chain` unless conditions narrow them. | Health, model, or unrelated routes can receive the limiter, or intended routes can be missed. | Effective route/filter mapping and positive/negative request matrix. |
| Policy hierarchy | Kuadrant can apply defaults, overrides, `atomic`, and `merge`; Praxis receives a flatter filter configuration. | A route can receive a different set of limits. | Effective Kuadrant policy per route and equivalent Praxis configuration. |
| Matching | Kuadrant uses CEL predicates and can apply multiple matching limits; Praxis conditions and internal rules have different matching and ordering semantics. | A request can select the wrong rule or only one of several intended limits. | Differential tests for predicates, missing fields, headers, methods, and multiple limits. |
| Rates | Kuadrant can enforce multiple rates in one limit; a Praxis rule has one algorithm and budget. | A short burst limit or long-term limit disappears. | Tests that exhaust each rate independently and together. |
| Identity | Kuadrant counters are defined per limit; Praxis currently has a filter-level key model. | Users or organizations share the wrong quota. | Counter vectors and generated keys for every supported identity case. |
| Reservation | Kuadrant version and cluster mode determine optimistic versus reservation behavior; Praxis estimates and reserves in its own filter. | Concurrent requests, failures, and final usage leave different quota balances. | Version-pinned tests for reserve, settle, timeout, and failure paths. |
| Token extraction | Kuadrant and Praxis expose related but different provider and JSON-path models. | Usage is counted as zero, an estimate, or a different value. | Fixtures for every provider, JSON response, and SSE response in scope. |
| Existing state | Limitador and Praxis use different keys, TTL semantics, and window models. | Quotas reset, double-count, or change meaning at rollout. | Snapshot and replay tests with counters and active reservations. |
| Product contract | Upper layers may need one set of policy and token-usage fields. | MaaS or other integrations become backend-specific. | A neutral contract and mappings for each supported engine/version. |

## Architectural options

| Option | What it preserves | Cost or risk | When it fits |
| --- | --- | --- | --- |
| Extend or reuse Limitador | Existing TRLPolicy concepts, counter state, and reservation path remain close to Kuadrant. | Praxis depends on the Limitador service and protocol; performance and operational ownership need review. | Strongest candidate when existing counters and semantics are the priority. |
| TRLPolicy compatibility adapter | Customers continue using the existing CRD; a controller or adapter produces Praxis configuration. | Every field and semantic difference must be defined. Translation alone does not solve counter compatibility. | When the Kubernetes API must remain stable but the enforcement path changes. |
| Shared policy model with multiple engines | MaaS and other products expose stable fields while selecting Kuadrant or Praxis underneath. | Requires a neutral semantic contract and mappings for every engine and version. | When both engines must be supported over time. |
| Praxis-native policy plus explicit migration | Simplifies long-term Praxis ownership. | Requires customer changes and a counter strategy; it is not transparent backward compatibility. | Only when an explicit upgrade/migration is acceptable. |

### Preliminary direction

The first option should be investigated seriously because it addresses both the CRD-era semantics and the data-at-rest concern raised in the discussion. If that coupling is not acceptable, the next option to evaluate is a `TRLPolicy` adapter paired with a defined Limitador state strategy. A Praxis-native migration should be treated as a deliberate product decision, not assumed to be the default interpretation of #844.

## Runtime and migration assumptions

The issue does not require Kuadrant and Praxis to enforce the same traffic indefinitely.

- **Normal target:** one enforcement source handles the request. Praxis may accept the existing `TRLPolicy` while Kuadrant is no longer the active enforcer for that traffic.
- **Shadow validation:** Praxis calculates decisions and metrics while Kuadrant remains the only blocker. Shadow does not transfer counters.
- **Dual enforcement:** both systems block or charge the same request. This should be avoided by default because it can create double rejection, double accounting, or two sources of truth.
- **Rolling migration:** different instances or traffic partitions use different engines. This requires explicit counter ownership and rollback rules.

If a state or backend change is selected, the operator can run an explicit process:

1. Export policies, effective routes, versions, and current state.
2. Translate the policy and report unsupported fields.
3. Validate with fixtures or shadow mode.
4. Choose state handling: shared Limitador, import, drain and expiry, or reset.
5. Cut over enforcement ownership and stop conflicting writes.
6. Verify counters, responses, and metrics; retain rollback.

If Praxis continues using the existing Limitador state, this cutover may be limited to enabling the adapter and changing enforcement ownership. There is no need for runtime detection based on traffic or Redis keys.

## Proposed work sequence and approval gate

The implementation should follow this order:

1. Lock the Kuadrant, Limitador, CRD, and Praxis versions in scope.
2. Build the full gap inventory and compatibility matrix.
3. Run focused comparison and data-at-rest experiments.
4. Write the design recommendation and state which gaps are supported, translated, or intentionally unsupported.
5. Review and approve the direction with the maintainers.
6. Only after approval, create the dedicated branch and build the prototype for the selected option.
7. Add the tests, example configuration, and operational instructions required by that prototype.

The dedicated branch is therefore a consequence of an approved design, not a parallel implementation of an undecided approach.

## Questions for review

- Is existing `TRLPolicy` required to work without edits, or is an explicit conversion step acceptable?
- Must existing Limitador counters and active reservations survive unchanged?
- Is Limitador an acceptable long-term dependency for Praxis?
- Which Kuadrant, Limitador, and CRD versions are supported?
- Must multiple rates, CEL predicates, defaults/overrides, and reservation mode be preserved exactly?
- Does MaaS need one neutral policy/token-usage contract for both engines?
- Is shadow validation required, and is rolling migration a real deployment requirement?
- Which gaps are acceptable for the first production release?

## References

- [Issue #844](https://github.com/praxis-proxy/ai/issues/844)
- [PR #796](https://github.com/praxis-proxy/ai/pull/796)
- [Kuadrant architecture PR #199](https://github.com/Kuadrant/architecture/pull/199)
- [Kuadrant token rate limit API compatibility RFC](https://github.com/Kuadrant/architecture/blob/main/rfcs/0024-token-rate-limit-api-format-compatibility.md)
- [Kuadrant Operator PR #2299](https://github.com/Kuadrant/kuadrant-operator/pull/2299)
- [Praxis token rate limit proposal](https://github.com/praxis-proxy/enhancements/blob/main/proposals/00121_token-rate-limiting.md)
