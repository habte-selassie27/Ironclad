[![Ironclad](docs/images/explorer-contract-page.png)](docs/images/explorer-contract-page.png)

# Ironclad

*Evidence doesn't enter a contract on trust — it enters on a verdict.*

---

## What this is

A GenLayer Intelligent Contract whose single job is to sit between the open web and whatever contract wants to treat a webpage as evidence. You give it a URL. It gives you back a verdict you can build on, or it tells you the page can't be trusted.

Most evidence pipelines ask: *what does this page say?* Ironclad asks a different question first: *is this page trying to talk to the model?*

## The bet

Any page that will be read by an LLM is a potential attack surface. The same HTML that carries a useful fact can carry instructions aimed at the reader: override the governing prompt, impersonate a system role, exfiltrate secrets, invoke tools, transact, or hand off to a second instruction chain. Ironclad makes that a consensus problem instead of a prompting problem.

## How a verdict is produced

1. **Admission** — the URL is sanity-checked before anything fetches: public HTTPS only, no credentials, no ports, no private/loopback/link-local hosts, no obfuscated host spellings.
2. **Leader** — fetches the page, bounds the text, runs a deterministic phrase floor, and asks the classifier for structured JSON. Excerpts must appear verbatim in what the leader saw. If the leader's own verdict isn't clean, it releases nothing.
3. **Validators** — each one fetches and classifies the page *again*, in its own execution. They don't compare prose; they compare the security facts: reachability, whether hard risk exists, whether the literal floor fired, whether parsing failed, and the derived class. Every released excerpt is re-grounded in the validator's own snapshot, and each is independently judged passive and purpose-relevant.
4. **Evidence binding** — if the leader says "nothing to release," validators check that claim against their own snapshots too. A clean page with releasable evidence can never settle as empty on the leader's say-so alone.
5. **Settlement** — deterministic code maps the findings to a terminal state. The model never writes the state itself.

Possible end states: `SAFE`, `SUSPICIOUS`, `QUARANTINED`, `UNAVAILABLE`, `CANCELLED`.

## Reading the verdict

| You see | Meaning | What to do |
|---|---|---|
| `SAFE` + excerpts | validators agree the page posed no control attempt and released bounded excerpts | you may consume the excerpts |
| `SAFE`, no excerpts | clean but nothing purpose-relevant | not consumable — wait for validators to confirm |
| `SUSPICIOUS` | deterministic phrase floor fired | do not consume |
| `QUARANTINED` | semantic risk or unparseable analysis | do not consume |
| `UNAVAILABLE` | source unreadable | retry later |
| `CANCELLED` | requester aborted | — |

```python
if ironclad.view().is_consumable(capsule_id):
    excerpts = ironclad.view().get_capsule(capsule_id)["excerpts"]
```

`is_consumable` is the only gate that matters for automation. Risk bits are diagnostic labels — two honest validators may name the same attack differently, so never branch on an exact bit.

## On-chain facts

```text
network : Studionet
address : 0xdd641B5bdBE8D9C14783b458425da180946Fe41c
status  : FINALIZED, MAJORITY_AGREE
source  : contracts/ironclad.py @ 1dd86da0 (parity-checked against the chain)
```

An earlier deployment predates the evidence-availability fix and survives only as an audit reference.

## Test posture

- `python scripts/preflight.py` — standalone source/security gate, zero dependencies: **86/86**
- `pytest tests/direct/ -v -s` — Direct Mode, forged-leader adversarial coverage included: **26/26**
- `genvm-lint check contracts/ironclad.py` — AST lint + SDK validation, exit 0
- `pytest tests/integration/ -v -s --network studionet` — live Studionet consensus: **4/4**

Verified live: a safe page settles `SAFE` with a grounded excerpt and becomes consumable; the hostile fixture settles `QUARANTINED` with a risk mask of 265 and releases no excerpts.

```bash
pip install -r requirements-test.txt   # Direct Mode
pip install -r requirements.txt        # lint + integration extras
python scripts/deploy_studionet.py     # deployments
```

## Guarantee boundary

Ironclad decides one thing: *may this source be released forward as passive evidence without trying to control the reader?* It does not decide whether a claim is true, who is speaking, whether the source is fresh, or whether the excerpts are enough to move money. Those belong to layers that compose with it.

## Receipts

<details><summary>Explorer — deployed contract</summary>
<img src="docs/images/explorer-contract-page.png" alt="Explorer contract page"></details>

<details><summary>Run and Debug — methods beside the consensus log</summary>
<img src="docs/images/studio-run-and-debug.png" alt="Run and Debug panels"></details>

<details><summary>get_risk_dictionary() — the full flag set</summary>
<img src="docs/images/read-get-risk-dictionary.png" alt="risk dictionary"></details>

<details><summary>get_capsule(1) — while still pending</summary>
<img src="docs/images/read-get-capsule-pending.png" alt="pending capsule"></details>

<details><summary>is_consumable(1) — closed before settlement</summary>
<img src="docs/images/read-is-consumable-false.png" alt="not consumable"></details>

<details><summary>get_capsule(1) — settled SAFE with excerpt</summary>
<img src="docs/images/read-get-capsule-safe.png" alt="safe capsule"></details>

<details><summary>is_consumable(1) — open after settlement</summary>
<img src="docs/images/read-is-consumable-true.png" alt="consumable"></details>

<details><summary>open_inspection transaction</summary>
<img src="docs/images/write-open-inspection.png" alt="open_inspection"></details>

<details><summary>resolve transaction</summary>
<img src="docs/images/write-resolve.png" alt="resolve"></details>

<details><summary>second open_inspection (cancel flow)</summary>
<img src="docs/images/write-open-inspection-for-cancel.png" alt="second open_inspection"></details>

<details><summary>cancel transaction</summary>
<img src="docs/images/write-cancel.png" alt="cancel"></details>

<details><summary>open receipt — accepted</summary>
<img src="docs/images/receipt-open-accepted.png" alt="open accepted"></details>

<details><summary>resolve receipt — accepted despite dissent</summary>
<img src="docs/images/receipt-resolve-accepted.png" alt="resolve accepted"></details>

<details><summary>resolve receipt — header detail</summary>
<img src="docs/images/receipt-resolve-detail.png" alt="resolve detail"></details>

<details><summary>resolve receipt — finalized</summary>
<img src="docs/images/receipt-resolve-finalized.png" alt="resolve finalized"></details>

<details><summary>second open receipt — accepted</summary>
<img src="docs/images/receipt-open2-accepted.png" alt="second open accepted"></details>

<details><summary>cancel receipt — accepted</summary>
<img src="docs/images/receipt-cancel-accepted.png" alt="cancel accepted"></details>

<details><summary>cancel receipt — header detail</summary>
<img src="docs/images/receipt-cancel-detail.png" alt="cancel detail"></details>

<details><summary>cancel receipt — finalized</summary>
<img src="docs/images/receipt-cancel-finalized.png" alt="cancel finalized"></details>

## License

MIT — [`LICENSE`](LICENSE).
