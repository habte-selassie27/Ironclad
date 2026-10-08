# Ironclad

> Hostile web evidence, screened by GenLayer consensus before it ever reaches your contract.

Ironclad is a standalone [GenLayer](https://genlayer.com) Intelligent Contract. Any contract can route a live webpage through Ironclad first and receive back a typed, consensus-backed verdict: the page is safe to quote, suspicious, quarantined, or unreadable — plus a small set of bounded, validator-anchored passive excerpts.

No frontend. No backend. No indexer. One contract, tests, and a deploy helper.

---

## The short version

Live pages can carry a useful fact *and* text written to manipulate the model reading it ("ignore your instructions", impersonate a system role, leak secrets, call tools, transact, follow a second instruction chain). Ironclad treats every page as hostile until proven otherwise:

- The **leader** fetches and classifies the page inside the non-deterministic path.
- **Validators independently re-fetch and re-classify** the same page and reject forged, malformed, or disagreeing leader output.
- A **deterministic lexical floor** guarantees obvious control phrases can never settle `SAFE`, no matter what the model says.
- Terminal state is **derived by deterministic code** — the LLM reports observations; it never writes `SAFE`/`SUSPICIOUS`/`QUARANTINED`.

```text
open_inspection(url, purpose)  -->  resolve(capsule_id)  -->  terminal capsule

                                  +--> SAFE         (usable passive evidence)
                                  +--> SUSPICIOUS   (literal control-language floor)
                                  +--> QUARANTINED  (semantic risk / unsafe analysis)
                                  +--> UNAVAILABLE  (source unreachable)
```

Downstream contracts should gate on:

```python
ironclad.view().is_consumable(capsule_id)   # True only for SAFE + grounded excerpts
```

---

## How a page settles

```text
PENDING
  |
  |-- resolve()  -->  SAFE | SUSPICIOUS | QUARANTINED | UNAVAILABLE
  |-- cancel()   -->  CANCELLED            (requester only, while pending)
```

Capsules are terminal and immutable: no owner, no admin override, no allowlist mutation, no way to un-quarantine evidence.

---

## Risk taxonomy

The classifier reports a bitmask built from ten diagnostic dimensions:

| Bit | Flag | What it catches |
|---:|---|---|
| 1 | `PROMPT_OVERRIDE` | attempts to replace governing instructions |
| 2 | `ROLE_IMPERSONATION` | claims system / developer / assistant authority |
| 4 | `TASK_REDIRECTION` | redirects the evidence task itself |
| 8 | `SECRET_EXFILTRATION` | requests hidden prompts, credentials, keys |
| 16 | `TOOL_OR_ACTION_COMMAND` | tells the reader to run code, call tools, transact |
| 32 | `OBFUSCATED_INSTRUCTION` | machine-directed instruction in disguise |
| 64 | `HIDDEN_INSTRUCTION` | non-visible instruction in rendered text |
| 128 | `EXTERNAL_INSTRUCTION_CHAIN` | sends the reader elsewhere to continue |
| 256 | `LITERAL_CONTROL_PHRASE` | deterministic lexical tripwire |
| 512 | `UNPARSABLE_ANALYSIS` | classifier output could not be safely parsed |

Any semantic bit, unparseable analysis, or unreadable source ⇒ `QUARANTINED`/`UNAVAILABLE`. Literal floor only ⇒ `SUSPICIOUS`. Clean ⇒ `SAFE`. Bits are diagnostic; settle on `status` / `is_consumable`.

---

## API surface

| Method | Kind | Purpose |
|---|---|---|
| `open_inspection(url, purpose)` | write | create a `PENDING` capsule from a bounded passive purpose |
| `resolve(capsule_id)` | write | run the consensus inspection once |
| `cancel(capsule_id)` | write | requester-only abort while pending |
| `get_capsule(capsule_id)` | view | full stored capsule |
| `is_consumable(capsule_id)` | view | the gate: `SAFE` + ≥1 grounded excerpt |
| `get_risk_dictionary()` | view | the fixed diagnostic dictionary |

---

## Live deployment (Studionet)

| Field | Value |
|---|---|
| Network | Studionet |
| Contract | `0xdd641B5bdBE8D9C14783b458425da180946Fe41c` |
| Deployment tx | `0x4e3dda328e0bfc325e45497944fd9c71b7ed898bc92571eba4bf0d12283b3b70` |
| State | `FINALIZED`, `MAJORITY_AGREE` |
| Source | `1dd86da00fff84344d3ff54e194c4b273ff013f1` |
| Tooling | official GenLayer CLI `0.39.2` |

`genlayer code` returns `contracts/ironclad.py` at this commit (chain copy is CRLF from a Windows upload; git serves LF — identical after normalization). A prior address, `0x86506D4017B5B47Ce8Cd03b3C561E3bd96cfA0e5`, predates the excerpt-availability fix and is retained for audit only.

---

## Verification at a glance

| Gate | Outcome |
|---|---|
| Source preflight (`scripts/preflight.py`) | PASS, 86/86 checks |
| Python compilation | PASS |
| Direct Mode (`tests/direct/`) | 26 passed, 0 failed |
| GenVM linter (`genvm-lint` 0.11.0) | exit 0 |
| Studionet integration (`tests/integration/`) | 4 passed |
| Live safe page | capsule `1`: `SAFE`, risk 0, consumable |
| Live hostile fixture | capsule `3`: `QUARANTINED`, risk 265, no excerpts |

```bash
python scripts/preflight.py
pip install -r requirements-test.txt
pytest tests/direct/ -v -s
pip install -r requirements.txt
genvm-lint check contracts/ironclad.py
pytest tests/integration/ -v -s --network studionet
```

Deploy with an unlocked GenLayer CLI account:

```bash
python scripts/deploy_studionet.py
```

---

## What the fold-closed rules guarantee

| Situation | Outcome |
|---|---|
| Page can't be rendered | `UNAVAILABLE` |
| Semantic control attempt detected | `QUARANTINED` |
| Classifier output malformed | `QUARANTINED` |
| Literal control phrase only | at least `SUSPICIOUS` |
| Non-SAFE capsule | zero excerpts released |
| SAFE but nothing grounded | not consumable |
| Forged / mistyped / disagreeing leader fields | proposal rejected |
| Leader withheld evidence validators found releasable | proposal rejected |
| Second resolve, or cancel by non-requester | rejected |

---

## Working CLI evidence

Screenshots from the official GenLayer Studio surfaces (Studionet contract `0xF5d253D05931adCC46C595Db42ed57E208d2e2d1`, captured 6 Oct 2026).

Explorer — deployed contract page:

![Explorer contract page](docs/images/explorer-contract-page.png)

Run and Debug — source and methods:

![Run and Debug source and method panels](docs/images/studio-run-and-debug.png)

`get_risk_dictionary()`:

![get_risk_dictionary result](docs/images/read-get-risk-dictionary.png)

`get_capsule(1)` while pending:

![get_capsule pending result](docs/images/read-get-capsule-pending.png)

`is_consumable(1)` before settlement:

![is_consumable false while pending](docs/images/read-is-consumable-false.png)

`get_capsule(1)` settled `SAFE`:

![get_capsule SAFE result with excerpt](docs/images/read-get-capsule-safe.png)

`is_consumable(1)` after settlement:

![is_consumable true after settlement](docs/images/read-is-consumable-true.png)

`open_inspection`:

![open_inspection transaction](docs/images/write-open-inspection.png)

`resolve`:

![resolve transaction](docs/images/write-resolve.png)

Second `open_inspection` for the cancel flow:

![open_inspection for cancel flow](docs/images/write-open-inspection-for-cancel.png)

`cancel`:

![cancel transaction](docs/images/write-cancel.png)

Open receipt:

![open receipt accepted](docs/images/receipt-open-accepted.png)

Resolve receipt under dissent:

![resolve receipt accepted under dissent](docs/images/receipt-resolve-accepted.png)

Resolve receipt header:

![resolve receipt header](docs/images/receipt-resolve-detail.png)

Resolve receipt finalized:

![resolve receipt finalized](docs/images/receipt-resolve-finalized.png)

Second open receipt:

![second open receipt accepted](docs/images/receipt-open2-accepted.png)

Cancel receipt:

![cancel receipt accepted](docs/images/receipt-cancel-accepted.png)

Cancel receipt header:

![cancel receipt header](docs/images/receipt-cancel-detail.png)

Cancel receipt finalized:

![cancel receipt finalized](docs/images/receipt-cancel-finalized.png)

---

## Boundaries

Ironclad answers one question: *can this source be released forward as passive evidence without itself trying to control the reader?* It does not decide truth, authority, freshness, or independence, and `SAFE` never authorizes a downstream payout on its own.

## License

MIT. See [`LICENSE`](LICENSE).
