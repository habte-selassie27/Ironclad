<h1 align="center">Ironclad</h1>

<p align="center"><b>A GenLayer intelligent-contract firewall that screens hostile live web content before any other contract may consume it as evidence.</b></p>

Ironclad is a single deployable GenLayer Intelligent Contract. It fetches live web pages inside consensus, classifies machine-directed prompt attacks with semantic review, enforces a deterministic literal tripwire, and only releases bounded, validator-anchored passive excerpts. It ships no frontend, indexer, or backend — just the contract, tests, and deployment tooling.

## What it does

1. A caller opens an inspection with a public HTTPS URL and a bounded passive purpose.
2. A leader renders and classifies the source, and proposes reachability, a risk mask, a reason, and excerpts.
3. Validators independently re-render and re-classify the same URL, reject forged or malformed leader fields, and ground every released excerpt in their own snapshot.
4. Deterministic contract code — never the model — derives the terminal status.

Each inspection ends in exactly one terminal state:

```text
PENDING
  |
  +-- resolve() ----------> SAFE | SUSPICIOUS | QUARANTINED | UNAVAILABLE
  |
  +-- requester cancel() -> CANCELLED
```

Only a `SAFE` capsule carrying at least one validator-grounded excerpt reports `is_consumable(capsule_id) == true`.

## Risk taxonomy

| Bit | Name | Meaning |
|---:|---|---|
| `1` | `PROMPT_OVERRIDE` | attempts to replace governing instructions |
| `2` | `ROLE_IMPERSONATION` | claims system/developer/assistant authority |
| `4` | `TASK_REDIRECTION` | redirects the evidence task |
| `8` | `SECRET_EXFILTRATION` | requests hidden prompts, credentials, or keys |
| `16` | `TOOL_OR_ACTION_COMMAND` | asks the reader to execute code, call tools, or transact |
| `32` | `OBFUSCATED_INSTRUCTION` | disguised machine-directed instruction |
| `64` | `HIDDEN_INSTRUCTION` | non-visible machine-directed instruction in rendered text |
| `128` | `EXTERNAL_INSTRUCTION_CHAIN` | sends the reader elsewhere to continue instructions |
| `256` | `LITERAL_CONTROL_PHRASE` | deterministic lexical floor |
| `512` | `UNPARSABLE_ANALYSIS` | classification could not be safely parsed |

Status derivation is deterministic: semantic-risk bits, `UNPARSABLE_ANALYSIS`, or an unreadable source lead to `QUARANTINED`/`UNAVAILABLE`; the literal floor alone yields `SUSPICIOUS`; no risk dimensions yields `SAFE`. Category bits are diagnostic labels — downstream settlement should use `status` or `is_consumable()`, never a single exact bit.

## Public API

- `open_inspection(url, purpose) -> u256` — create a pending capsule
- `resolve(capsule_id)` — permissionless, single-use settlement
- `cancel(capsule_id)` — requester-only while pending
- `get_capsule(capsule_id) -> dict` — stored capsule
- `is_consumable(capsule_id) -> bool` — the gate downstream contracts should require
- `get_risk_dictionary() -> dict` — the fixed diagnostic dictionary

## Live Studionet deployment

| Evidence | Value |
|---|---|
| Network | Studionet |
| Contract | `0xdd641B5bdBE8D9C14783b458425da180946Fe41c` |
| Deployment tx | `0x4e3dda328e0bfc325e45497944fd9c71b7ed898bc92571eba4bf0d12283b3b70` |
| Deployment state | `FINALIZED`, `MAJORITY_AGREE` |
| Deployment source | `1dd86da00fff84344d3ff54e194c4b273ff013f1` |
| Deployment method | official GenLayer CLI `0.39.2` |

`genlayer code` returns `contracts/ironclad.py` at this commit (the chain copy is CRLF from a Windows upload; git serves LF; after normalization they are identical). The earlier address `0x86506D4017B5B47Ce8Cd03b3C561E3bd96cfA0e5` (commit `594d3243`) predates the excerpt-availability fix and is kept for audit only.

## Testing and verification

| Gate | Result |
|---|---|
| SDK-free source preflight | PASS — `86/86` checks |
| Python compilation | PASS |
| Direct Mode | PASS — `26 passed, 0 failed`, `genlayer-test v0.29.2` |
| GenVM linter | PASS — `genvm-lint` `0.11.0`, exit `0` |
| Studionet integration | PASS — `4 passed` |
| Live safe / hostile evidence | PASS — capsule `1`: `SAFE`, consumable `true`; capsule `3`: `QUARANTINED`, risk `265`, consumable `false` |

```bash
python scripts/preflight.py
pip install -r requirements-test.txt
pytest tests/direct/ -v -s
pip install -r requirements.txt
genvm-lint check contracts/ironclad.py
pytest tests/integration/ -v -s --network studionet
```

## Deployment

The contract takes no constructor arguments:

```bash
python scripts/deploy_studionet.py
# or: genlayer deploy --contract contracts/ironclad.py --rpc https://studio.genlayer.com/api
```

## Working CLI evidence

Captures from the official GenLayer Studio surfaces, taken 6 Oct 2026 against Studionet contract `0xF5d253D05931adCC46C595Db42ed57E208d2e2d1`.

**Explorer contract page**

![Explorer contract page](docs/images/explorer-contract-page.png)

**Run and Debug — source and method panels**

![Run and Debug source and method panels](docs/images/studio-run-and-debug.png)

**`get_risk_dictionary()` result**

![get_risk_dictionary result](docs/images/read-get-risk-dictionary.png)

**`get_capsule(1)` while pending**

![get_capsule pending result](docs/images/read-get-capsule-pending.png)

**`is_consumable(1)` before settlement**

![is_consumable false while pending](docs/images/read-is-consumable-false.png)

**`get_capsule(1)` after SAFE settlement**

![get_capsule SAFE result with excerpt](docs/images/read-get-capsule-safe.png)

**`is_consumable(1)` after settlement**

![is_consumable true after settlement](docs/images/read-is-consumable-true.png)

**`open_inspection` transaction**

![open_inspection transaction](docs/images/write-open-inspection.png)

**`resolve` transaction**

![resolve transaction](docs/images/write-resolve.png)

**Second `open_inspection` (cancel flow)**

![open_inspection for cancel flow](docs/images/write-open-inspection-for-cancel.png)

**`cancel` transaction**

![cancel transaction](docs/images/write-cancel.png)

**Open receipt accepted**

![open receipt accepted](docs/images/receipt-open-accepted.png)

**Resolve receipt — accepted under dissent**

![resolve receipt accepted under dissent](docs/images/receipt-resolve-accepted.png)

**Resolve receipt header**

![resolve receipt header](docs/images/receipt-resolve-detail.png)

**Resolve receipt finalized**

![resolve receipt finalized](docs/images/receipt-resolve-finalized.png)

**Second open receipt accepted**

![second open receipt accepted](docs/images/receipt-open2-accepted.png)

**Cancel receipt accepted**

![cancel receipt accepted](docs/images/receipt-cancel-accepted.png)

**Cancel receipt header**

![cancel receipt header](docs/images/receipt-cancel-detail.png)

**Cancel receipt finalized**

![cancel receipt finalized](docs/images/receipt-cancel-finalized.png)

## Fail-closed summary

| Condition | Result |
|---|---|
| Source cannot be read | `UNAVAILABLE` |
| Semantic machine-control risk | `QUARANTINED` |
| Classifier output unparseable | `QUARANTINED` |
| Literal floor only | at least `SUSPICIOUS` |
| Capsule not SAFE | no excerpts released |
| SAFE but no grounded excerpt | `is_consumable == false` |
| Forged, wrongly typed, or disagreeing leader fields | proposal rejected |
| Leader withheld releasable evidence | proposal rejected |
| Repeat resolution or non-requester cancellation | rejected |

## Scope

Ironclad decides whether a source can be released forward as passive evidence without attempting to control the reader. It does not verify truth, freshness, authority, or independence, and it is not a complete defence against every future prompt-injection technique or network-level SSRF vector.

## License

MIT — see [`LICENSE`](LICENSE).
