# MCPS 1.1 — Claim Resolution, Secure Agent Identity & Distributed Targeting

**Work Product / Architecture Note**  
**Date:** 08 Sep 2026

A proposed secure extension to MCP combining dual-principal identity, provenance-preserving claim resolution, stochastic lineage, scoped authority, and cryptographically bound distributed-agent targeting.

## 1. Purpose

MCPS manages hallucination risk and distributed-agent accountability by controlling how generated claims acquire evidentiary standing and authority, while identifying both the human principal on whose behalf work is performed and the agent that actually produces a claim or action.

The design preserves lineage rather than attempting to eliminate stochastic generation. It separates identity, claim truth-state, targeting, and permission so none can silently substitute for another.

> **Core invariant:** Human Identity ≠ Agent Identity ≠ Target ≠ Claim Resolution ≠ Authority.

> **Claim invariant:** Generation ≠ Corroboration ≠ Resolution ≠ Authority.

## 2. MCPS trust model

MCPS is modeled as:

`MCP + Human Principal + Secure Agent Identity + Target Selector + Provenance + Claim Resolution + Scoped Authority`

The trust path is:

`Human Principal → Actor Agent → Target Selector → Claim/Instruction → Resolution → Authority → Interface`

The human principal answers **on whose behalf is this work being done?** The actor agent answers **which agent actually produced this claim or action?** The target selector answers **which authenticated agents are intended to receive or act upon it?** Claim state answers **what evidentiary standing has the assertion earned?** Authority answers **what consequence is permitted?**

## 3. Human principal

MCPS requires an authenticated human principal, represented through OpenID Connect (OIDC) identity context.

```yaml
principal:
  human:
    protocol: oidc
    issuer:
    subject:
    audience:
    session_id:
    auth_time:
```

OIDC establishes the human identity represented by the work. It does not establish claim truth, agent identity, or action authority by itself.

## 4. Secure Agent ID

Each agent receives an independently verifiable identity at emergence. The Secure Agent ID is the root of the agent's cryptographic provenance.

A conceptual derivation is:

`AgentID = H(agent_public_key, emergence_time, parent_identity, environment_hash, nonce)`

The agent receives an asymmetric keypair at emergence. The private key remains inside the agent/runtime trust boundary; the public key and emergence attestation identify the agent.

```yaml
agent_identity:
  version: "1.1"
  agent_id: "mcps:agent:..."
  public_key: "..."
  emerged_at: "..."
  parent_agent_id: "..."
  model_fingerprint: "..."
  environment_hash: "..."
  emergence_nonce: "..."
  issuer: "self | parent | trusted-runtime"
  signature: "..."
```

Every claim or consequential message binds the actor agent identity into its signed envelope. This permits verification that a particular agent produced a particular claim without treating that attribution as evidence that the claim is true.

### Agent ancestry

Distributed agents may spawn descendants:

`Agent A → Agent B → Agent C`

Each child may carry a verified `parent_agent_id`, parent signature, and emergence context hash. Parent relationships are attested edges, not self-asserted labels. This produces a cryptographic agent genealogy suitable for subtree targeting and provenance reconstruction.

## 5. Emergence seed and stochastic lineage

Identity is stable; stochastic lineage is branchable. The seed should be derived from, but never used as a substitute for, Secure Agent Identity.

`Seed = KDF(SecureAgentID, emergence_entropy, branch_nonce)`

Preserve at minimum:

- `emergence_id`
- `origin_seed`
- `branch_id`
- `context_hash`
- `actor_agent_id`

The seed enables replay, cross-seed comparison, branch provenance, quarantine of unsupported descendants, and regeneration from the last valid state. Cross-seed invariance is evidence of stability, not proof of truth.

## 6. Dual-principal binding

Every MCPS claim binds both identities:

```yaml
claim:
  claim_id:
  statement:
  human_principal: "oidc:<issuer>:<subject>"
  actor_agent: "mcps:agent:..."
  branch_id:
  context_hash:
  signature:
```

The human can be valid while the agent claim remains Potential. A claim can remain Resolved while human authority expires. An unverified actor agent cannot borrow the human principal's identity as proof of agent identity.

For delegated agents, descendants retain the relevant human principal reference while carrying their own Secure Agent IDs and signed parent lineage.

## 7. Claim-confidence state machine

| State | Meaning | Permitted use | Authority |
|---|---|---|---|
| **POTENTIAL** | Generated or inferred; evidentiary standing not yet earned. | Internal exploration, comparison, hypothesis generation. | None |
| **CORROBORATED** | At least one genuinely independent support path survives. | Internal reasoning; may inform further validation. | None |
| **RESOLVED** | Support, provenance, contradiction limits, and resolution threshold are satisfied. | Adopt as current working state. | None by default |
| **AUTHORIZED** | Not a confidence state. Scoped permission granted after Interface validation. | Only the explicitly approved consequence and scope. | Active / scoped |

Authority is scoped, temporary, consequence-specific, and may fall faster than confidence. A Resolved claim does not automatically become Authorized.

## 8. Contradictions, retractions, and revoked authority

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
- Policy, consent, human authorization, or target-scope changes may revoke authority without changing claim confidence.

## 9. Re-resolution algorithm

Re-resolution is a deterministic recomputation from surviving provenance. It never merely subtracts confidence points.

1. **Freeze consequence.** Suspend consequential authority when a material contradiction or meaningful retraction arrives.
2. **Rebuild support DAG.** Collapse derivative evidence into independent lineages. Five agents repeating one upstream source count as one lineage.
3. **Apply retractions.** Zero or weaken affected lineage weights while retaining historical assertions and retraction events.
4. **Classify contradictions.** Classify as irrelevant, compatible, weak conflict, material conflict, or direct disproof; weight by credibility and independence.
5. **Cross-seed replay.** Re-run the normalized evidence state across independent seeds/resolvers and measure stable support. Agreement is corroboration, not truth.
6. **Verify provenance.** Require reconstructable provenance for Resolved status. Consensus with opaque lineage cannot manufacture resolution.
7. **Decide state.** Apply hard gates and thresholds to select Potential, Corroborated, or Resolved.
8. **Reauthorize separately.** If Resolved survives, rerun Interface policy. Never restore authority automatically.

### Decision function

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

## 10. Competing resolutions

When incompatible claims each retain meaningful support, preserve both as competing branches. Do not demote one merely because another viable explanation exists. The parent question remains unresolved until one branch earns resolution or the conflict itself is resolved.

## 11. Distributed target selectors

MCPS does not require a single target agent. The target is an **open hierarchy selector** over the authenticated agent graph.

`TargetSelector → resolve against agent graph → TargetSet`

The selector itself is signed as part of the MCPS semantic envelope. Transport routing does not establish target authority.

```yaml
target:
  version: "1.1"
  mode: subtree | cohort | capability | namespace | broadcast
  selector: {}
  constraints:
    include_self: false
    max_depth:
    require_verified_identity: true
    require_active: true
    max_targets:
    valid_until:
  exclusions: []
  resolution:
    policy: live | snapshot
```

An agent qualifies only when:

`Match(selector, agent) AND Constraints(agent) AND AuthorityAllows(actor, agent)`

Selector-relevant attributes must be cryptographically attributable through agent identity, emergence attestation, configuration, or signed delegation. Self-assertion is insufficient.

## 12. Subtree targeting

`subtree` selects an authenticated agent and eligible descendants connected through verified parent-child edges.

```yaml
target:
  mode: subtree
  selector:
    root_agent_id: "mcps:agent:A"
  constraints:
    include_self: false
    max_depth: 3
```

A descendant qualifies only when a verified parent chain exists from the root to that agent. `max_depth` bounds traversal. `include_self` controls whether the root itself is included.

Subtree describes **agent genealogy**, not administrative membership.

## 13. Cohort targeting

`cohort` selects agents sharing an attested emergence classification.

```yaml
target:
  mode: cohort
  selector:
    cohort_id: "resolver-generation-17"
```

Cohort membership belongs to the emergence envelope or another signed authority. A cohort need not share ancestry. It can represent agents born during a run, experiment, generation, or other emergence grouping.

`Cohort ≠ Subtree`.

## 14. Capability targeting

`capability` selects agents whose verified capability manifests satisfy the requested capability predicate.

```yaml
target:
  mode: capability
  selector:
    capability:
      name: "causal-resolution"
      version: ">=2.1"
```

A capability selector means deliver to agents **attested as capable**, not agents that merely claim the capability. Capability manifests may be signed by a trusted runtime, issuer, parent, certification authority, or other configured authority.

Selectors may compose capabilities using `all` and `any` predicates.

## 15. Namespace targeting

`namespace` selects agents within a signed logical administrative or functional domain.

```yaml
target:
  mode: namespace
  selector:
    namespace: "finance/reconciliation"
    namespace_match: descendants
```

Namespaces are hierarchical. `exact` matches only the named namespace; `descendants` includes eligible child namespaces.

Namespace describes **administrative topology**. Subtree describes **agent genealogy**. The two are intentionally independent.

Agents cannot self-enroll into privileged namespaces; membership must derive from signed delegation or trusted configuration.

## 16. Broadcast targeting

`broadcast` targets every qualifying agent inside an explicitly bounded authority domain. It never means every agent everywhere.

```yaml
target:
  mode: broadcast
  selector:
    scope:
      subtree_root: "mcps:agent:A"
  constraints:
    require_verified_identity: true
    max_targets: 500
```

A broadcast scope may be defined by subtree, namespace, human principal, or another explicitly authorized domain. Broadcast is higher consequence because membership may be large or dynamic and therefore requires explicit broadcast authority and bounded constraints.

## 17. Selector composition

Selectors may be composed as intersections, unions, and exclusions.

```yaml
target:
  selector:
    all:
      - type: subtree
        root_agent_id: "mcps:agent:A"
        max_depth: 4
      - type: capability
        capability:
          name: "causal-resolution"
          version: ">=2.0"
    any: []
    exclude:
      - type: namespace
        namespace: "experimental"
```

Conceptually:

`TargetSet = selector composition - exclusions`, with every candidate still subject to identity, constraint, and authority validation.

Composition permits targets such as:

- resolver-capable descendants of Agent A
- members of Cohort C17 in the research namespace
- agents supporting PostgreSQL or MySQL
- all authorized descendants except experimental agents

## 18. Live and snapshot resolution

### Snapshot

`resolution.policy: snapshot` freezes membership when the selector is resolved. Agents emerging later do not receive the instruction. Snapshot is appropriate for approvals, financial actions, fixed workflows, and audit-sensitive operations.

### Live

`resolution.policy: live` keeps the selector active. A newly emerged agent that later satisfies the selector and authority constraints becomes eligible. Live selectors behave like secure subscriptions and support distributed research, monitoring, swarm processes, and long-running orchestration.

For auditability, selector resolution records:

```yaml
resolution_result:
  selector_hash:
  graph_version:
  resolved_at:
  target_set_hash:
  target_count:
```

## 19. Delegation inheritance

A child agent does not automatically inherit all targeting or action authority from its parent. Delegation is explicit and bounded.

```yaml
delegation:
  from_agent: "mcps:agent:A"
  to_agent: "mcps:agent:B"
  target_scope:
    subtree: "mcps:agent:A/*"
  permissions:
    - send
    - delegate
  max_delegation_depth: 2
  expires_at:
  signature:
```

Every delegation edge may narrow target scope, action scope, duration, and further delegation depth. A descendant cannot expand authority beyond the intersection of its inherited delegation chain.

## 20. Receiver-side target validation

Every receiving agent evaluates semantic inclusion independently of transport routing:

`AmIIncluded(selector)`

If infrastructure delivers a valid MCPS envelope to an agent outside the resolved target set, that agent rejects the envelope for action. This makes targeting a security boundary rather than networking metadata.

The governing rule is:

> **Transport destination does not establish target authority. Only membership in the signed, authorized MCPS target selector does.**

## 21. Authority gate

A consequential action requires all relevant dimensions to succeed independently:

`AuthorizedAction = HumanAuthenticated AND HumanScopeAllows AND ActorAgentVerified AND TargetAuthorized AND ClaimResolved AND ClaimSignatureValid AND InterfaceApproved`

For messages that are instructions rather than factual claims, the applicable policy may replace `ClaimResolved` with the appropriate instruction/intent validation state, but identity, targeting, signature, and authority gates remain separate.

## 22. Integrated MCPS envelope

```yaml
mcps:
  version: "1.1"

  principal:
    human:
      protocol: oidc
      issuer:
      subject:
      audience:
      session_id:
      auth_time:

    actor_agent:
      agent_id:
      public_key:
      emerged_at:
      parent_agent_id:
      model_fingerprint:
      environment_hash:
      signature:

  target:
    selector:
      all: []
      any: []
      exclude: []
    resolution:
      policy: snapshot
    constraints:
      require_verified_agent: true
      require_active_agent: true
      max_targets:
      valid_until:
    proof:
      selector_hash:
      actor_signature:

  claim:
    claim_id:
    statement:
    state: potential | corroborated | resolved
    confidence:
    contested:
    retracted:

    provenance:
      human_principal:
      actor_agent_id:
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
      human_principal:
      actor_agent_id:
      target_selector_hash:
      reason:
      expires_at:

  signature:
    signer_agent_id:
    algorithm:
    value:
```

## 23. Governing invariants

1. Human identity, agent identity, target identity, claim truth-state, and authority are independent dimensions.
2. Potential cannot acquire authority merely because one stochastic trajectory produced it.
3. An authenticated human does not make an agent claim true.
4. A verified agent does not make its claim true.
5. A resolved claim does not automatically grant action authority.
6. Contradiction creates a branch, not deletion.
7. Retraction removes support, not provenance.
8. Resolution is reversible.
9. Authority can be revoked without changing truth status.
10. Any new authority requires a fresh Interface decision.
11. Re-resolution recalculates confidence from surviving provenance.
12. Secure Agent ID is stable; stochastic seed lineage is derived and branchable.
13. Parent-child agent relationships require attested edges.
14. Selector membership cannot be established by self-assertion alone.
15. Transport routing cannot establish target authority.
16. Broadcast is always bounded by an explicit authorized domain.
17. Delegated authority can narrow but cannot silently expand down the hierarchy.
18. Live selectors may admit future agents only when those agents satisfy identity, selector, constraint, and authority requirements.

## 24. Implementation checkpoint

Minimum MCPS 1.1 implementation requires:

- OIDC human-principal binding
- Secure Agent ID and asymmetric agent signing
- emergence attestation and parent-agent lineage
- seed/emergence stochastic lineage
- claim envelope
- provenance DAG
- contradiction and retraction events
- deterministic re-resolution
- scoped authority records
- signed Target Selector
- authenticated agent graph
- subtree, cohort, capability, namespace, and bounded broadcast selectors
- selector composition and exclusions
- live and snapshot target resolution
- delegation constraints
- receiver-side target validation
- Interface policy hook

Thresholds, weights, permitted issuers, capability attestors, namespace authorities, selector limits, and cryptographic algorithms should be configuration or negotiated profile parameters. The transition, identity-separation, provenance, target-binding, and authority invariants remain protocol rules.
