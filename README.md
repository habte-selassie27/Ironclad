<h1 align="center">Ironclad</h1>

<p align="center"><b>A reusable GenLayer consensus firewall for hostile web evidence.</b></p>

<p align="center">
  <img src="https://img.shields.io/badge/GenLayer-Intelligent%20Contract-111111" alt="GenLayer Intelligent Contract" />
  <img src="https://img.shields.io/badge/Direct%20Mode-26%2F26%20passing-111111" alt="26 of 26 Direct Mode tests passing" />
  <img src="https://img.shields.io/badge/preflight-86%2F86%20passing-111111" alt="86 of 86 preflight checks passing" />
  <img src="https://img.shields.io/badge/Studionet-FINALIZED-111111" alt="Studionet deployment finalized" />
  <img src="https://img.shields.io/badge/license-MIT-111111" alt="MIT license" />
</p>

**Ironclad screens untrusted live web content before another Intelligent Contract or agent may treat it as evidence.** It re-observes hostile sources under GenLayer consensus, fails closed on malformed or unsafe analysis, and releases only bounded, validator-grounded passive excerpts.

Ironclad is deliberately **contract-only**: no frontend, dashboard, wallet flow, backend or indexer.

```text
open_inspection(url, purpose)
        |
        v
resolve(capsule_id)
        |
        +--> SAFE          source-anchored passive evidence may be consumed
        +--> SUSPICIOUS    deterministic control-language floor fired
        +--> QUARANTINED   semantic risk or unsafe/unparseable analysis
        +--> UNAVAILABLE   source could not be read
```

Only a `SAFE` capsule with at least one validator-grounded excerpt returns `is_consumable(capsule_id) == true`. Whether that list is empty is itself decided by validators, not the leader — see [evidence availability](#evidence-availability-is-validator-bound).

## Live Studionet deployment

| Evidence | Value |
|---|---|
| Network | Studionet |
| Contract | `0xdd641B5bdBE8D9C14783b458425da180946Fe41c` |
| Deployment tx | `0x4e3dda328e0bfc325e45497944fd9c71b7ed898bc92571eba4bf0d12283b3b70` |
| Deployment state | `FINALIZED`, `MAJORITY_AGREE` |
| Deployment source | `1dd86da00fff84344d3ff54e194c4b273ff013f1` |
| Deployment method | official GenLayer CLI `0.39.2` |

Source parity is verified: `genlayer code` returns `contracts/ironclad.py` at this commit (chain copy is CRLF from a Windows upload; git serves LF). Digests, parity check and every smoke transaction are in [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md). The earlier address `0x86506D4017B5B47Ce8Cd03b3C561E3bd96cfA0e5` (commit `594d3243`) predates the excerpt-availability fix and is kept for audit.

## 30-second reviewer version

Ironclad answers one narrow, reusable question: **can a downstream Intelligent Contract consume live web evidence without letting the page itself instruct or redirect the model reading it?**

The caller supplies a URL and a bounded passive purpose. The leader fetches and classifies it; validators independently re-fetch, reject forged leader fields, ground every excerpt in their own snapshot and judge passive relevance. Deterministic code derives the terminal status — the model never writes `SAFE`, `SUSPICIOUS` or `QUARANTINED`.

## The problem

GenLayer contracts can read the live web and interpret it with LLMs — a boundary ordinary oracle designs lack: one page may hold a useful fact and text aimed at controlling the reader (ignore governing prompts, impersonate system authority, expose secrets, call tools, transact, continue elsewhere). If every builder reinvents that defence, downstream contracts inherit inconsistent prompt-injection handling. Ironclad makes evidence intake a composable primitive. It does **not** sanitise the internet or prove a claim true; its question is narrower:

> Can this source be released forward as passive evidence without itself attempting to control the reader?

Truth, freshness, authority, policy and settlement remain separate layers.

## Why this needs GenLayer

A central content-safety API replaces one trust problem with another operator. Ironclad needs live web access (the contract observes the source, not the caller), semantic reasoning (manipulation is not substring-shaped) and independent consensus (one model invocation cannot decide what others may trust). Delete GenLayer and it fails open: one operator becomes the trust authority, the caller reports model output selectively, a regex or format-only check misses paraphrased attacks while accepting forged verdicts, or the caller supplies the evidence itself.

## Architecture

```text
untrusted live URL
       |
       v
deterministic admission -> leader fetch + classify
       |
       v
validators independently fetch + classify, agree on security dimensions
       |
       v
excerpt grounding + passive/relevance verification
       |
       v
deterministic terminal status -> typed capsule
       |
       v
downstream truth / policy / settlement primitive
```

## State model

Each inspection is one-way and terminal:

```text
PENDING
  |
  +-- resolve() ----------> SAFE | SUSPICIOUS | QUARANTINED | UNAVAILABLE
  |
  +-- requester cancel() -> CANCELLED
```

A terminal capsule cannot be re-resolved or overwritten. Capsules store `requester`, `url`, `purpose`, `status`, `risk_mask`, `reason`, `excerpts` and transaction-observed timestamps — with no owner, administrator, mutable allowlist or privileged safety override.

## Risk taxonomy

| Bit | Name | Meaning |
|---:|---|---|
| `1` | `PROMPT_OVERRIDE` | tries to replace governing instructions |
| `2` | `ROLE_IMPERSONATION` | claims system/developer/assistant authority |
| `4` | `TASK_REDIRECTION` | redirects the evidence task |
| `8` | `SECRET_EXFILTRATION` | requests hidden prompts, credentials or keys |
| `16` | `TOOL_OR_ACTION_COMMAND` | asks the reader to execute code, call tools or transact |
| `32` | `OBFUSCATED_INSTRUCTION` | disguised machine-directed instruction |
| `64` | `HIDDEN_INSTRUCTION` | non-visible machine-directed instruction in rendered text |
| `128` | `EXTERNAL_INSTRUCTION_CHAIN` | sends the reader elsewhere to continue instructions |
| `256` | `LITERAL_CONTROL_PHRASE` | deterministic lexical floor |
| `512` | `UNPARSABLE_ANALYSIS` | classification could not be safely parsed |

Status derivation is deterministic: any semantic-risk bit, `UNPARSABLE_ANALYSIS` or an unreadable source → `QUARANTINED` / `UNAVAILABLE`; only the literal floor → `SUSPICIOUS`; no risk dimensions → `SAFE`. The LLM never writes the terminal state.

### Stable integration rule

Category bits are **diagnostic labels**, not the settlement API — honest validators can label the same attack differently. Consensus requires agreement on reachability, hard-risk presence, literal-floor presence, parser failure and derived class. Downstream automatic decisions should use `status` or `is_consumable()`; never one exact category bit.

## Consensus design

Ironclad uses one custom `gl.vm.run_nondet_unsafe(leader_fn, validator_fn)`.

**Leader.** Renders the admitted URL in text mode, bounds the source, computes the lexical floor, sends JSON-framed purpose and hostile source to the classifier, keeps only short verbatim excerpts, drops evidence when its own classification is not `SAFE`, and proposes reachability, risk, reason and excerpts.

**Validator.** Independently renders and classifies the same URL, requires typed `bool` reachability and bounded integer risk fields, rejects unknown bits, checks that security dimensions and derived class agree, grounds every excerpt in **its own snapshot** (no second-fetch TOCTOU gap), rejects evidence on a non-`SAFE` derivation, and independently judges each released excerpt as passive and relevant — not a schema check: the Direct Mode suite injects forged leader payloads via `direct_vm.run_validator(leader_result=...)`.

Validators need not agree on prose or fine-grained category bits, which can vary between runs; they must agree on reachability, hard-risk presence, literal-floor presence, parser failure, derived class and whether releasable evidence existed. Full design in [`docs/CONSENSUS.md`](docs/CONSENSUS.md).

## Evidence availability is validator-bound

`is_consumable` is `SAFE` plus a non-empty excerpt list. Verifying only what a leader chose to release would leave the empty list as the one consensus-visible field nobody checked, letting two honest leaders turn the same SAFE page consumable or not by choosing whether to speak. Both directions therefore share one acceptance test, `judge_excerpt_release`: a leader must either ground every released excerpt in the validator's snapshot and pass its release judgment, or — when proposing nothing — have no grounded candidate that passes that judgment.

Evidence rides on `SAFE` only: `excerpts_for_class` is shared by the leader path, the validator and settlement. This binds availability, not selection — which releasable span a leader picks remains its choice, and every pick is judged independently. Where honest models disagree the proposal is rejected and the capsule stays `PENDING`.

## Deterministic gates

**Before consensus.** Only conservative public HTTPS hostnames are admitted: no non-HTTPS URLs, embedded credentials, explicit ports, `localhost`/`.local`/`.internal`, private/link-local/loopback IPv4, numeric or leading-zero IP spellings, encoded ambiguous hosts, malformed DNS labels, DNS wrappers over private IPv4, or blank/oversized purposes that become a second control channel — defence in depth, not a replacement for validator egress policy.

**During classification.** `risk_mask` must be an integer or decimal integer string; booleans, floats, hex-like strings and unsupported bits fail closed; literal control phrases set a deterministic floor; source and purpose are JSON-framed in prompts.

**During validator verification.** Unknown bits and wrongly typed fields are rejected; every excerpt must be a canonical string anchored in the validator's own snapshot and pass an independent passive/relevance judgment; excerpts on a non-`SAFE` derivation are rejected; an empty list is rejected when the validator's snapshot held releasable evidence.

**After consensus.** Only `status == SAFE and len(excerpts) > 0` is consumable, reachable only when validators independently agreed nothing was releasable.

## Fail-closed matrix

| Condition | Result |
|---|---|
| Source cannot be read | `UNAVAILABLE` |
| Semantic machine-control risk | `QUARANTINED` |
| Classifier cannot be safely parsed | `QUARANTINED` |
| Deterministic literal floor only | at least `SUSPICIOUS` |
| Capsule not SAFE | no excerpts released |
| SAFE but no grounded excerpt | `is_consumable == false` |
| Forged, wrongly typed or disagreeing leader fields | proposal rejected |
| Leader withheld evidence the validator found releasable | proposal rejected |
| Repeat resolution or non-requester cancellation | rejected |

## Public API

### `open_inspection(url, purpose) -> u256`

Creates a `PENDING` evidence capsule from a bounded passive purpose (e.g. `Extract factual evidence about whether ACME announced version 3.0.`).

Other entrypoints: `resolve(capsule_id)` (permissionless, single-use), `cancel(capsule_id)` (requester only, while pending), `get_capsule(capsule_id) -> dict` (stored capsule), `is_consumable(capsule_id) -> bool` (preferred downstream gate: `SAFE` plus a grounded excerpt), and `get_risk_dictionary() -> dict` (fixed diagnostic risk dictionary).

## Validation results

| Gate | Result | Evidence |
|---|---|---|
| SDK-free source preflight | PASS | `86/86` checks |
| Python source compilation | PASS | contract and scripts compile |
| Direct Mode | PASS | `26 passed, 0 failed, 0 skipped`, `genlayer-test v0.29.2`, Python 3.12.13 |
| GenVM linter | PASS | `genvm-lint check contracts/ironclad.py --json`, `0.11.0`, exit `0` |
| Studionet integration | PASS | `4 passed`, fresh disposable contract per test |
| Studionet deployment | PASS | finalized contract and tx recorded above |
| Live safe / hostile evidence | PASS | capsule `1`: `SAFE`, risk `0`, consumable `true`; capsule `3`: `QUARANTINED`, risk `265`, consumable `false`, fixture [`fixtures/hostile_evidence.txt`](fixtures/hostile_evidence.txt) |

### Commands

```bash
python scripts/preflight.py                    # zero-dependency source/security gate
pip install -r requirements-test.txt
pytest tests/direct/ -v -s                     # state, consensus, forged-leader coverage
pip install -r requirements.txt
genvm-lint check contracts/ironclad.py         # AST lint + SDK validation
pytest tests/integration/ -v -s --network studionet
```

Direct Mode covers safe grounded evidence, prompt-like purpose rejection, literal and semantic attacks, malformed/fractional/hex-like risk fields, unsupported bits, invented excerpts, forged leader payloads, non-boolean reachability, class disagreement, cancellation, single-resolution and the excerpt-availability binding in both directions.

## Studionet deployment

Ironclad has no constructor arguments. With an active/unlocked GenLayer CLI account:

```bash
python scripts/deploy_studionet.py    # or: genlayer deploy --contract contracts/ironclad.py --rpc https://studio.genlayer.com/api
```

## Working UI and CLI evidence

Ironclad ships no frontend. These are the official GenLayer surfaces used to deploy, call and verify it on Studionet (`0xF5d253D05931adCC46C595Db42ed57E208d2e2d1`), captured 6 Oct 2026.

### Studio

**Deployed contract — Explorer page.** Deploy tx `0x74bd817a…becb7943`, `FINALIZED`, consensus `Accepted`.

![Explorer contract page](docs/images/explorer-contract-page.png)

**Source and method panels — Run and Debug.** Three read and three write methods beside the consensus log.

![Run and Debug source and method panels](docs/images/studio-run-and-debug.png)

### Read methods

**`get_risk_dictionary()` — full taxonomy.** All 10 risk bits and masks.

![get_risk_dictionary result](docs/images/read-get-risk-dictionary.png)

**`get_capsule(1)` — pending state.** `status: 0`, empty `excerpts`, not consumable.

![get_capsule pending result](docs/images/read-get-capsule-pending.png)

**`is_consumable(1)` — gated before settlement.**

![is_consumable false while pending](docs/images/read-is-consumable-false.png)

**`get_capsule(1)` — settled SAFE.** `status: 1`, `risk_mask: 0`, excerpt, `consumable: true`.

![get_capsule SAFE result with excerpt](docs/images/read-get-capsule-safe.png)

**`is_consumable(1)` — evidence released.**

![is_consumable true after settlement](docs/images/read-is-consumable-true.png)

### Write methods

**`open_inspection` — capsule created.** Capsule `1` on `https://example.com/`; five validators `agree`.

![open_inspection transaction](docs/images/write-open-inspection.png)

**`resolve` — consensus inspection run.** Fetch and classification for capsule `1`.

![resolve transaction](docs/images/write-resolve.png)

**`open_inspection` — disposable capsule for the cancel flow.**

![open_inspection for cancel flow](docs/images/write-open-inspection-for-cancel.png)

**`cancel` — requester-only abort.** Four `agree`, one `idle`.

![cancel transaction](docs/images/write-cancel.png)

### Transaction receipts

**Open — accepted.** `MAJORITY_AGREE`, five `AGREE`, `ACCEPTED`.

![open receipt accepted](docs/images/receipt-open-accepted.png)

**Resolve — accepted under dissent.** `MAJORITY_AGREE` despite `AGREE, DISAGREE, AGREE, AGREE, IDLE`.

![resolve receipt accepted under dissent](docs/images/receipt-resolve-accepted.png)

**Resolve — receipt header.** Sender, contract, calldata `{"args":[1],"method":"resolve"}`.

![resolve receipt header](docs/images/receipt-resolve-detail.png)

**Resolve — finalized.** `status_name: 'FINALIZED'`.

![resolve receipt finalized](docs/images/receipt-resolve-finalized.png)

**Second open — accepted.** `MAJORITY_AGREE`, five `AGREE`.

![second open receipt accepted](docs/images/receipt-open2-accepted.png)

**Cancel — accepted.** `MAJORITY_AGREE`, `IDLE, AGREE, AGREE, AGREE, AGREE`.

![cancel receipt accepted](docs/images/receipt-cancel-accepted.png)

**Cancel — receipt header.** Calldata `{"args":[2],"method":"cancel"}`.

![cancel receipt header](docs/images/receipt-cancel-detail.png)

**Cancel — finalized.** Confirmed by the network.

![cancel receipt finalized](docs/images/receipt-cancel-finalized.png)

## Security properties

- **Caller text is not evidence** — the source is fetched inside the nondeterministic path.
- **Hostile source text is data** — purpose, source and excerpts are JSON-framed before reasoning.
- **The leader cannot invent evidence** — excerpts must occur in an independent validator snapshot.
- **Obvious control language cannot become SAFE** — a deterministic lexical floor sits outside model judgment.
- **Malformed fields fail closed** — ambiguous risk representations cannot coerce a safe mask.
- **Forged leader fields are validated** — unknown bits, wrong types or disagreement reject the proposal.
- **The model does not settle state** — deterministic code derives status and the consumption gate.
- **No privileged bypass** — nobody can rewrite a terminal capsule or un-quarantine evidence.

## What Ironclad does not claim

No classifier can prove arbitrary content harmless in every future context. Ironclad does not claim:

- perfect detection of every prompt-injection technique;
- protection against a malicious GenLayer validator majority;
- complete DNS-rebinding/network-level SSRF prevention;
- that `SAFE` means the claim is true, or that a source is authoritative, fresh or independent;
- that a safe excerpt alone can settle a market, escrow, insurance policy or governance action;
- inspection of content absent from GenLayer's rendered text representation.

See [`docs/SECURITY.md`](docs/SECURITY.md) for the full threat model.

## Integration

Treat Ironclad as an intake gate: refuse a source unless `is_consumable(capsule_id)` is true, then consume only the released excerpts and apply your own truth, corroboration, freshness and policy checks. Composition example in [`docs/INTEGRATION.md`](docs/INTEGRATION.md).

## Repository layout

```text
.
├── contracts/ironclad.py          the deployable contract
├── tests/direct/                  state, consensus, forged-leader tests
├── tests/integration/             live Studionet tests
├── scripts/preflight.py           zero-dependency source/security gate
├── scripts/deploy_studionet.py    deployment helper
├── fixtures/hostile_evidence.txt  public hostile source
├── docs/                          CONSENSUS, SECURITY, INTEGRATION, DEPLOYMENT, APPEAL, images/
├── requirements-test.txt          Direct Mode deps
├── requirements.txt               optional tooling, including linter
├── gltest.config.yaml
├── SUBMISSION.md
├── LICENSE
└── README.md
```

## Reviewer fast path

1. Read the deployment table and thesis above.
2. Inspect [`contracts/ironclad.py`](contracts/ironclad.py), especially `_inspect`, the custom validator, terminal status derivation and `is_consumable`.
3. Inspect the forged-leader tests in [`tests/direct/test_ironclad_hardening.py`](tests/direct/test_ironclad_hardening.py).
4. Run [`tests/integration/test_ironclad_studionet.py`](tests/integration/test_ironclad_studionet.py) for live consensus evidence.
5. Read [`docs/CONSENSUS.md`](docs/CONSENSUS.md), [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) and [`docs/SECURITY.md`](docs/SECURITY.md).

## Builder submission

**Category:** Standalone GenLayer Intelligent Contract  
**Primitive:** Ironclad  
**Purpose:** consensus-backed hostile-web-evidence intake  
**Repository:** `https://github.com/habte-selassie27/Ironclad`  
**Studionet contract:** `0xdd641B5bdBE8D9C14783b458425da180946Fe41c`

Copy-ready notes in [`SUBMISSION.md`](SUBMISSION.md).

## Licence

MIT. See [`LICENSE`](LICENSE).
