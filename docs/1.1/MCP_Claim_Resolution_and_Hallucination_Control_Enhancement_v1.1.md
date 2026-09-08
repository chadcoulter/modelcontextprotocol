# MCP Claim Resolution & Hallucination Control Enhancement v1.1

**Work Product / Architecture Note**  
**Date:** 08 Sep 2026

A provenance-preserving state machine for stochastic claims, contradiction handling, retraction, re-resolution, and scoped authority in agent/MCP workflows.

## 1. Purpose

This enhancement manages hallucination risk by controlling how generated claims acquire evidentiary standing and authority. It does not attempt to eliminate stochastic generation. Instead, it preserves the lineage of a claim, tests its stability and support, and prevents unresolved material from silently crossing the interface into consequential action.

The design aligns with the Monadic Resolution Model: Potential is distinct from Present Resolution; provenance survives resolution; contradictory viable paths create branching potential; and Interface traversal governs distributed consequence.

> **Core invariant:** Generation ≠ Corroboration ≠ Resolution ≠ Authority. A claim may be explored internally without gaining permission to produce external consequence.

## 2. Emergence seed and stochastic lineage

Each agent receives an emergence-derived stochastic lineage. Emergence metadata influences the seed, but must not be the sole entropy source.

`Seed = f(emergence timestamp, environment, parent state, entropy)`

Preserve at minimum:

- `emergence_id`
- `origin_seed`
- `branch_id`
- `context_hash`

The seed enables replay, cross-seed comparison, branch provenance, quarantine of unsupported descendants, and regeneration from the last valid state. Cross-seed invariance is evidence of stability, not proof of truth.

## 3. Claim-confidence state machine

| State | Meaning | Permitted use | Authority |
|---|---|---|---|
| **POTENTIAL** | Generated or inferred; evidentiary standing not yet earned. | Internal exploration, comparison, hypothesis generation. | None |
| **CORROBORATED** | At least one genuinely independent support path survives. | Internal reasoning; may inform further validation. | None |
| **RESOLVED** | Support, provenance, contradiction limits, and resolution threshold are satisfied. | Adopt as current working state. | None by default |
| **AUTHORIZED** | Not a confidence state. Scoped permission granted after Interface validation. | Only the explicitly approved consequence and scope. | Active / scoped |

> **Authority law:** Authority is scoped, temporary, consequence-specific, and may fall faster than confidence. A Resolved claim does not automatically become Authorized.

## 4. Contradictions, retractions, and revoked authority

### Contradiction

A contradiction creates a competing branch rather than deleting the original claim. Mark the claim contested, attach the contradiction to provenance, and re-resolve.

- Weak challenges may leave the claim **Resolved**.
- Material challenges may demote it to **Corroborated**.
- Collapse of support returns it to **Potential**.

### Retraction

A retraction removes or weakens support contributed by the retracting lineage. It does not erase the historical assertion.

- If independent support survives, the claim may remain **Resolved** or become **Corroborated**.
- If foundational support disappears, it returns to **Potential**.

### Authority suspension and revocation

A material challenge to a consequentially Authorized claim suspends authority before confidence is recomputed.

- If re-resolution yields anything below **Resolved**, relevant authority is revoked.
- If the claim remains **Resolved**, authority stays suspended until a fresh Interface decision reissues it.
- Policy or consent may revoke authority without changing the claim’s confidence state.

## 5. Re-resolution algorithm

Re-resolution is a deterministic recomputation from surviving provenance. It never merely subtracts confidence points.

1. **Freeze consequence.** Suspend consequential authority when a material contradiction or meaningful retraction arrives.
2. **Rebuild support DAG.** Collapse derivative evidence into independent lineages. Five agents repeating one upstream source count as one lineage.
3. **Apply retractions.** Zero or weaken affected lineage weights while retaining the historical assertion and retraction event.
4. **Classify contradictions.** Classify as irrelevant, compatible, weak conflict, material conflict, or direct disproof; weight by credibility and independence.
5. **Cross-seed replay.** Re-run the normalized evidence state across independent seeds/resolvers and measure stable support. Treat agreement as corroboration, not truth.
6. **Verify provenance.** Require reconstructable provenance for Resolved status. Consensus with opaque lineage cannot manufacture resolution.
7. **Decide state.** Apply hard gates and thresholds to select Potential, Corroborated, or Resolved.
8. **Reauthorize separately.** If Resolved survives, rerun Interface policy. Never restore authority automatically.

## 6. Decision function

`E(C) = w_s*S + w_p*P + w_x*X - w_k*K`

Where:

- `S` = surviving independent support
- `P` = provenance integrity
- `X` = cross-seed/resolver stability
- `K` = credible contradiction strength

| Condition | Result |
|---|---|
| No surviving independent support | **POTENTIAL** |
| Unresolved direct disproof | **POTENTIAL** |
| Provenance below minimum for resolution | **CORROBORATED** or **POTENTIAL** |
| `E >= resolved_threshold` + minimum independent paths + conflict within limit | **RESOLVED** |
| `E >= corroborated_threshold` + at least one independent support path | **CORROBORATED** |
| Otherwise | **POTENTIAL** |

## 7. Competing resolutions

When incompatible claims each retain meaningful support, preserve both as competing branches. Do not demote one merely because another viable explanation exists. The parent question remains unresolved until one branch earns resolution or the conflict itself is resolved.

## 8. MCP claim envelope

```yaml
claim_id:
statement:
state: potential | corroborated | resolved
confidence:
contested:
retracted:
provenance:
  emergence_id:
  origin_seed:
  branch_id:
  context_hash:
  sources: []
validation:
  independent_support_paths:
  support_score:
  contradiction_score:
  provenance_integrity:
  cross_seed_stability:
  decision:
authority:
  - scope:
    status: none | pending | active | suspended | revoked | expired
    reason:
```

## 9. Governing invariants

1. Potential cannot acquire authority merely because one stochastic trajectory produced it.
2. Contradiction creates a branch, not deletion.
3. Retraction removes support, not provenance.
4. Resolution is reversible.
5. Authority can be revoked without changing truth status.
6. Any new authority requires a fresh Interface decision.
7. Re-resolution recalculates confidence from surviving provenance.

## 10. Implementation checkpoint

Minimum implementation requires:

- claim envelope
- provenance DAG
- seed/emergence lineage
- contradiction and retraction events
- deterministic re-resolution
- scoped authority records
- Interface policy hook

Thresholds and weights should be configuration, while the transition invariants remain protocol rules.
