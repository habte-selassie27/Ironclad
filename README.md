<p align="center"><img src="docs/images/explorer-contract-page.png" alt="Ironclad on GenLayer Studio" width="640"></p>

<h1 align="center">Ironclad</h1>

<p align="center"><i>Don't let the page prompt the model.</i></p>

---

## What you get

A hostile webpage should never reach a downstream contract as raw "evidence". Ironclad sits in front of that handoff: you hand it a URL, it fetches the page inside GenLayer consensus, and it hands back a small, typed verdict you can gate on.

- every page is treated as an attack surface first;
- validators re-read the page themselves — one model's opinion is never enough;
- obvious "ignore your instructions" language can't settle safe, no matter the model;
- the model reports findings; deterministic code writes the final state.

## The pipeline

```text
you                Ironclad                       validators
 |                     |                              |
 |-- open_inspection ->|                              |
 |                     |-- fetch + classify (leader)  |
 |                     |----------------------------->|
 |                     |        fetch + classify again|
 |<--------------------|  agree on security dimensions
 |                     |                              |
 |-- resolve --------->|-- deterministic terminal state
 |                     |                              |
 |-- is_consumable? -->| true only if SAFE + grounded excerpts
```

Terminal states, and only these: `SAFE`, `SUSPICIOUS`, `QUARANTINED`, `UNAVAILABLE`, `CANCELLED`.

## Why the leader isn't trusted

The leader proposes. Validators dispose. Each validator:

1. renders the URL itself, in its own execution;
2. classifies that fresh snapshot on its own;
3. checks the leader's fields are well-typed (booleans are booleans, masks are bounded integers, no unknown bits);
4. compares security dimensions — reachability, hard-risk presence, literal-floor presence, parse failure, derived class — not prose;
5. grounds every released excerpt in its own snapshot, so nothing quoted is invented;
6. judges every excerpt as passive and relevant to the caller's purpose;
7. if the leader released nothing, checks whether its own snapshot held releasable evidence — if so, the proposal is rejected.

A proposal that disagrees on any security dimension is discarded. Rotation exhaustion leaves the capsule `PENDING`; it never force-settles.

## Risk flags

Ten bits, diagnostic only:

`PROMPT_OVERRIDE` · `ROLE_IMPERSONATION` · `TASK_REDIRECTION` · `SECRET_EXFILTRATION` · `TOOL_OR_ACTION_COMMAND` · `OBFUSCATED_INSTRUCTION` · `HIDDEN_INSTRUCTION` · `EXTERNAL_INSTRUCTION_CHAIN` · `LITERAL_CONTROL_PHRASE` · `UNPARSABLE_ANALYSIS`

Full dictionary on-chain via `get_risk_dictionary()`. Semantic bits, parse failures, or an unreadable source ⇒ quarantine. Literal floor only ⇒ suspicious. Clean ⇒ safe. Bits are labels — integrate on `status` / `is_consumable()`.

## Surface area

```python
open_inspection(url, purpose) -> u256   # create
resolve(capsule_id)                     # settle (anyone, once)
cancel(capsule_id)                      # requester only
get_capsule(capsule_id) -> dict         # inspect
is_consumable(capsule_id) -> bool       # the gate
get_risk_dictionary() -> dict           # diagnostics
```

## Where it runs

| | |
|---|---|
| Network | Studionet |
| Contract | `0xdd641B5bdBE8D9C14783b458425da180946Fe41c` |
| Deploy tx | `0x4e3dda328e0bfc325e45497944fd9c71b7ed898bc92571eba4bf0d12283b3b70` |
| State | `FINALIZED` / `MAJORITY_AGREE` |
| CLI | GenLayer `0.39.2` |

Source parity was checked against the chain: `genlayer code` returns `contracts/ironclad.py` at commit `1dd86da0` (chain copy CRLF, repo copy LF — identical after normalization). A previous address pre-dates the excerpt-availability binding and is kept for audit only.

## Proof it works

| Check | Result |
|---|---|
| preflight | 86/86 |
| Direct Mode | 26/26 |
| genvm-lint 0.11.0 | exit 0 |
| Studionet integration | 4/4 |
| live safe page | SAFE, risk 0, consumable |
| live hostile page | QUARANTINED, risk 265, empty excerpts |

```bash
python scripts/preflight.py
pip install -r requirements-test.txt && pytest tests/direct/ -v -s
pip install -r requirements.txt && genvm-lint check contracts/ironclad.py
pytest tests/integration/ -v -s --network studionet
python scripts/deploy_studionet.py
```

## Fail closed, always

Unreadable source → unavailable. Bad model JSON → quarantined. Literal tripwire → at least suspicious. Not SAFE → no excerpts. Forged types or unknown bits → rejected. Leader hid releasable evidence → rejected. Second resolve → rejected. No admin, no overrides, no exceptions.

## On-chain captures

<sub>GenLayer Studio, Studionet, 6 Oct 2026</sub>

<p align="center"><img src="docs/images/studio-run-and-debug.png" alt="Run and Debug" width="480"><br><b>Run and Debug</b></p>

<p align="center"><img src="docs/images/read-get-risk-dictionary.png" alt="risk dictionary" width="480"><br><b>get_risk_dictionary()</b></p>

<p align="center"><img src="docs/images/read-get-capsule-pending.png" alt="pending capsule" width="480"><br><b>get_capsule(1) — pending</b></p>

<p align="center"><img src="docs/images/read-is-consumable-false.png" alt="not consumable yet" width="480"><br><b>is_consumable(1) — before settlement</b></p>

<p align="center"><img src="docs/images/read-get-capsule-safe.png" alt="safe capsule" width="480"><br><b>get_capsule(1) — SAFE</b></p>

<p align="center"><img src="docs/images/read-is-consumable-true.png" alt="consumable" width="480"><br><b>is_consumable(1) — after settlement</b></p>

<p align="center"><img src="docs/images/write-open-inspection.png" alt="open" width="480"><br><b>open_inspection</b></p>

<p align="center"><img src="docs/images/write-resolve.png" alt="resolve" width="480"><br><b>resolve</b></p>

<p align="center"><img src="docs/images/write-open-inspection-for-cancel.png" alt="open for cancel" width="480"><br><b>second open_inspection</b></p>

<p align="center"><img src="docs/images/write-cancel.png" alt="cancel" width="480"><br><b>cancel</b></p>

<p align="center"><img src="docs/images/receipt-open-accepted.png" alt="open receipt" width="480"><br><b>open receipt</b></p>

<p align="center"><img src="docs/images/receipt-resolve-accepted.png" alt="resolve receipt" width="480"><br><b>resolve receipt (dissent)</b></p>

<p align="center"><img src="docs/images/receipt-resolve-detail.png" alt="resolve header" width="480"><br><b>resolve receipt header</b></p>

<p align="center"><img src="docs/images/receipt-resolve-finalized.png" alt="resolve finalized" width="480"><br><b>resolve finalized</b></p>

<p align="center"><img src="docs/images/receipt-open2-accepted.png" alt="open2 receipt" width="480"><br><b>second open receipt</b></p>

<p align="center"><img src="docs/images/receipt-cancel-accepted.png" alt="cancel receipt" width="480"><br><b>cancel receipt</b></p>

<p align="center"><img src="docs/images/receipt-cancel-detail.png" alt="cancel header" width="480"><br><b>cancel receipt header</b></p>

<p align="center"><img src="docs/images/receipt-cancel-finalized.png" alt="cancel finalized" width="480"><br><b>cancel finalized</b></p>

## Honestly, what it doesn't do

- prove a claim true or a source authoritative;
- catch every novel prompt-injection trick;
- survive a malicious validator majority;
- replace validator-side egress controls;
- settle money on its own.

## License

MIT — see [`LICENSE`](LICENSE).
