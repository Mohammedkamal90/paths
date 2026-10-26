---
name: paths
description: Trace Solidity execution paths from entrypoints through internal calls, state reads/writes, external calls, callbacks/hooks, and sinks. Produce separate verified-complete and incomplete/uncertain path reports.
---

# Paths — Solidity Execution-Path Analysis

## Mission

Trace execution and data/state changes using repository evidence only. Do not invent functions, edges, values, call targets, protocol behavior, or reachability. Distinguish code facts, proven conclusions, and unknowns.

## 1. Establish Scope

1. Identify source roots, target contracts, imports, inheritance, libraries, interfaces, compiler pragma and compiler revision (if evidenced), and available tests/configuration.
2. Resolve imported files and inherited implementations recursively where source is available. Record missing, generated, external, or unresolved dependencies.
3. Classify the system from code evidence: token, AMM/pool/pair, token + AMM, or other protocol. Do not classify from names alone.
4. Pin analyzed source to commit/revision or stable hashes; cite it in reports. Identify concrete deployable contracts and runtime context (chain/fork, proxy/implementation/admin, constructor/deployer, initializer state) only from evidence; otherwise mark unknown.
5. List entrypoints per deployable contract, including fallback/receive, proxy dispatch, initializers, public/external functions, and privileged/callback paths. Record caller/role/value context. Log every excluded candidate and reason in `scope_and_limits`.

## 2. Trace Direction

Start at each supported entrypoint and trace in execution order:

`entrypoint -> modifiers/guards -> internal/library/inherited calls -> variable reads/writes -> external calls -> callback/hook/re-entry -> resumed caller -> checks/sinks -> return or revert`

This is a tracing guide, not a mandatory sequence: callbacks/hooks may occur before, between, or after state operations. Follow actual code order and dynamic dispatch. Trace all supported branches separately.

## 3. Call Resolution

- Resolve C3 inheritance/`super`, virtual dispatch, overload selection, constructors, storage layout/proxy slots, library calls and `using for`, and nested modifier bodies when source/compiler rules permit. Unresolved resolution/dependency marks dependent edge/path incomplete.
- Expand modifiers in execution order around every `_`; account for nested modifiers, multiple placeholders, early returns, and implicit modifier reverts. Record pre/post-placeholder effects and guards.
- For external calls, record caller, target expression, function/signature if resolvable, call type, return handling, and failure behavior.
- Bind interface/address targets to implementations only with cited evidence (e.g. immutable, constructor assignment, hardcoded address, deploy script, test config). Repository presence alone is not binding; never substitute remembered protocol behavior. Tag modeled edges `ASSUMED:<id>`; any dependent path belongs in report B.
- Account for `call`, `staticcall`, `delegatecall`, `create`, and `create2`: target/code and return/failure semantics; value/context; created address where derivable. `delegatecall` executes code in caller storage context and preserves `msg.sender`/`msg.value`; successful `staticcall` cannot write state. `create`/`create2` run init code as constructor in new account context; record creator, value, address derivation, and resulting state/code only when derivable.
- At every external-call site record state snapshot visible at call, re-enterable entrypoints, guard status, and whether caller-controlled callee can execute arbitrary code. Do not invent attacker/receiver code; hypothetical models are assumptions and dependent paths go to B.
- Follow callbacks, hooks, token receiver callbacks, flash-loan/swap callbacks, fallback/receive, and reentrant paths only when a code-backed edge or explicit modeled environment establishes them.
- If a target or callback implementation is unavailable or ambiguous, mark the edge unresolved; never fabricate its behavior.
- Preserve call/return nesting and resume caller after success. Revert discards writes, logs, and transient writes in reverting frame and callees; caller-frame state persists if it catches failure. Uncaught failure propagates to transaction root.

## 4. State and Data Lineage

For every encountered variable access and branch/loop condition on each traced path, record (do not filter as irrelevant):

- Contract and storage instance; variable/expression and storage location (storage, memory, calldata, transient storage, or local).
- Read/write kind, exact source location, execution order, and path condition.
- Value before/after only when derivable; otherwise record the expression and unknown dependency.
- Whether a write/log is tentative, externally observable, or committed on successful completion; apply frame-level revert behavior in §3.
- For every state-changing path, identify each affected variable, exact operation, and source location; record increase/decrease direction when determinable. Track attempted versus committed changes, including balance/allowance, reserve, staking/reward, and other relevant accounting updates. Do not infer exact values or directions unsupported by the code and execution path; apply revert/rollback behavior in §3.
- Data dependencies: which reads, parameters, return values, calls, or prior writes influence a later write, branch, external call, or sink.
  Track aliases and mapping/array keys when resolvable. If aliasing, key equality, or value is uncertain, preserve that uncertainty.

## 5. Protocol-Specific Analysis

- For token systems, trace relevant transfer/transferFrom/mint/burn/approval and fee/tax/limit logic only where present in source.
- If code establishes an AMM/pool/pair integration, trace token paths together with the actual pool/pair implementation and its swap/skim/sync/callback/hook flow where applicable. Resolve caller identity, transfer direction, reserves, callbacks, updates, and relevant state changes from the actual cross-contract execution. Follow available source; mark unresolved dependencies or unsupported behavior without inventing assumptions. Resolve token0/token1 and caller direction only from evidence. Do not substitute a generic AMM model for the actual implementation.
- If a staking contract is involved, trace token behavior through the actual Token–Staking interaction and available cross-contract source, accounting for caller context and relevant state changes. Mark unresolved dependencies or unsupported behavior without inventing assumptions.
- For non-token, non-AMM protocols, analyze entrypoints and reachable internal/external execution without forcing a token/pool model.
- Treat events as evidence of emitted logs, not proof by themselves that the intended state transition succeeded.

## 6. Reachability and Sink

- Enumerate branch outcomes and loop iterations within explicit budgets; state truncation bounds. A path is one entrypoint-to-terminal sequence for a fixed branch-decision vector. IDs: `P-<entry>-<n>`; list unexamined branches and excluded entrypoints in scope.
- Assign `reach_tier`: `source_possible` (source-derived; feasibility unproven), `witness_feasible` (documented concrete caller/role, calldata, value, state/context satisfy predicates), or `runtime_confirmed` (execution evidence). Tier describes reachability evidence, not trace completeness; never infer execution from source.
- Put fully traced source paths in A even when tier is `source_possible`, provided all relevant edges/semantics resolve. Put only truncated/unresolved/unsupported paths in B; preserve confirmed prefixes and mark the exact unresolved edge. A missing witness alone never makes a path incomplete. Runtime claims cite exact command, source revision, and relevant output.
- Identify sinks only when relevant to the task and code-supported (e.g. state-changing operation, token/pool transfer, low-level call, privileged action, accounting check, revert, or return). Do not assume a sink is vulnerable.

## 7. Required Final Reports

Produce exactly two JSON path reports, keeping each path ID stable across references. Save `complete_reachable_paths.json` and `incomplete_or_uncertain_paths.json` at the analyzed repository root, beside `src/`; both must be in the same repository as the analyzed source, never only in an external workspace/output folder. After creating and validating both JSON reports, also create `Flow.md` there as the flow-diagram report.

### A. `complete_reachable_paths.json`

JSON object schema: `{"token_classification": null | {"tokens": [{"contract": "...", "types": [...], "evidence": [...]}]}, "pricing": null | {"mechanics": [...], "path_ids": [...], "evidence": [...]}, "entrypoints": [...], "table_text": [...], "paths": [...]}`. Put `token_classification` first and `pricing` immediately after it. For a token or AMM/token repository, classify each relevant token contract from source evidence and cite that evidence; allow multiple types and record `unknown` or unresolved traits rather than guessing. Consider Standard ERC-20 Tokens, Fee-on-Transfer (FOT) Tokens, Reflection (RFI) Tokens, Rebasing/Elastic Supply Tokens, Dividend-Yield/Reward-Paying Tokens, Upgradable Proxy Tokens, Blacklist/Pausable Tokens, Mintable & Burnable Tokens, ERC-4626 Vault-Wrapped Tokens, Deflationary Tokens, ERC-404 (Hybrid ERC-20/ERC-721) Tokens, LayerZero OFT Tokens, Bridge-Mintable Tokens, and other evidenced types. For AMM/token repositories, describe how token price is determined (`pricing.mechanics`) from code-backed formulas, inputs, and state/dependencies, and list the complete path IDs that establish those mechanics in `pricing.path_ids`; cite source evidence. Explicitly mark unavailable or unresolved pricing behavior rather than infer it. Highlight these pricing paths in `Flow.md`. Use `null` for token classification or pricing when not applicable.

Inventory only entrypoints represented in this report. Include fully traced paths regardless of reachability tier. Each record: stable ID; `reach_tier`; terminal outcome (`completes`, `reverts`, or named terminal); ordered call chain/edges; branch forks with location, condition, and source-level outcome; guards passed/denied; reads and delta writes; external target binding, call type/return/failure; callbacks/reentrancy; semantics/uncertainty; evidence for every edge. Mark source-only paths `source_possible`, never runtime-confirmed.

### B. `incomplete_or_uncertain_paths.json`

JSON object schema: `{"scope_and_limits": {...}, "entrypoints": [...], "table_text": [...], "paths": [...]}`. Begin scope with source revision/hash, classification/evidence, dependencies, exclusions, budgets, unexamined branches and unsupported semantics. Include only truncated/unresolved paths. Each record preserves resolved prefix, ID/tier, terminal/frontier, branch/guard evidence, state trace, external-call binding/return and callback status, exact unresolved edge or limit, and next evidence needed. Never discard prefix because later edge is unresolved.

Both files require `table_text`: JSON string array, one aligned header then one row per entrypoint with paths in that report. Header: `entrypoint                  access          paths   groups  diverg  sinks  status`; columns/order are fixed. Use concise IDs (e.g. `Sky.transferFrom`), access `permissionless`, `restricted`, `conditional`, or `constructor`, numeric counts, and evidence-backed status (`ok`, `INCOMPLETE`, `CONDITIONAL`, `UNRESOLVED`). Generate rows from that report's path records/inventory only; do not duplicate entrypoints without paths in this report. Any analyzed entrypoint omitted from both reports must have a concrete scope exclusion reason.

Counts per report only: `paths` = path records for entrypoint; `groups` = unique keys of ordered call-chain + branch/control-flow outcomes + terminal class; `diverg` = paths taking a distinct non-normal outcome (guard denial/revert or materially different route); ordinary successful branch variants are not divergences by default. `sinks` = unique `(contract/storage instance, source location, operation/effect)` identities reached; repeats count once. Generate counts mechanically from that file's records. Example header/row strings: `"entrypoint                  access          paths   groups  diverg  sinks  status"`, `"Sky.constructor             constructor     1       1       0       0      ok"`. Keep report/path schemas distinct; never put Markdown table markup in JSON.

Do not force every candidate into report A. Do not silently omit unresolved paths; report them in B. If a report has no entries, state that it is empty and why.

### C. `Flow.md`

Use `/home/glitch/.agents/skills/archify` and follow its authoring and finalization workflow to draw a source-backed sequence or dataflow diagram from `complete_reachable_paths.json` only. Include only paths with at least one state-changing or variable-changing operation. Show included path IDs and the ordered operations, including transfer direction, reserves, callbacks/hooks, updates, `balanceOf`, and other relevant state changes when present. Identify affected variables and operations; show change direction only when supported by the path evidence. List `pricing.path_ids` as pricing paths in `Flow.md` and visually highlight those that are included in the diagram; read-only pricing paths remain listed but are not added to the diagram. Save `Flow.md` at the analyzed repository root with the included path IDs and a link to the finalized standalone HTML diagram. Do not include paths from report B or infer behavior. If no complete path qualifies, state that in `Flow.md` and do not invent a diagram.

## 8. Evidence and Output Rules

- Cite repository-relative file paths and exact line ranges for every material claim. Quote only the minimum code needed to establish the claim.
- Tag material claims `[FACT]`, `[ASSUMPTION]`, `[INFERENCE]`, or `[UNKNOWN]`; keep facts separate from conclusions. Put assumption-dependent paths in B. Runtime claims cite exact command, source revision, and relevant output; otherwise do not claim execution.
- Record relevant Solidity/EVM semantics by pinned compiler/revision: checked/`unchecked` arithmetic, bounds failures, division by zero, ABI decoding failures, implicit reverts, `extcodesize`, and low-level calls to EOAs, as applicable. Do not overstate behavior beyond evidence.
- Record analysis limits, unsupported syntax/semantics, missing dependencies, and unexamined branches.
- Validate both output files at the analyzed repository root as JSON; check distinct schemas, `table_text` header/order/alignment, one row per inventoried path entrypoint, and recompute every count from that report's actual path records.
- Verify `Flow.md` links to the finalized Archify diagram and lists only qualifying path IDs from `complete_reachable_paths.json`.
- Never claim exhaustive coverage if dependencies/targets are unresolved, semantics unsupported, or branches/loops excluded; enumerate those limits.
- Never label a path safe merely because no issue was found. This skill traces paths; it does not independently prove security.
