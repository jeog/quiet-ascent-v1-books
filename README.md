# Quiet Ascent — V1 Book Commitments

A tamper-evident, timestamped record of what a set of systematic equity books held, published
**before** the outcome of each month was known.

## Why this exists

Published performance proves very little on its own. Anyone can select a track record in hindsight,
publish the survivors, and quietly drop the rest. The only thing that makes a forward record
credible to someone who does not trust the author is a commitment made **before** the outcome is
known and **provably not altered afterwards**.

That is all this repository is. Each month it records a cryptographic fingerprint of every book,
anchors it to a time nobody controls, and links it to the previous month so the sequence cannot be
rewritten.

**It is not a performance claim.** Nothing here asserts that any of these books made money.

## Which book is traded

Twelve rows are committed each month. Only one is traded.

| | what it is | status |
|---|---|---|
| **`unrestricted`** fully invested | the principal book: the same selection as the licensed tiers, with no position cap and no monthly rebalancing | **traded** |
| the other eleven | the licensed tiers and their treatments, and the principal book's other treatments | published, not traded |

**The account replicates the published book directly.** At the live start it buys the book as
the construction defines it on that date — the continuation of the backtest's own ladder — the way
an index fund replicates an index on launch. Every position it holds is a position the committed
row holds; the exit rule runs on the row's entry prices, not the account's fills; and the account's
realised return is reconciled against the committed row from month one. Divergence between the two
is tracking error, reported, not absorbed.

A cold start — an empty account deployed over six months so that every holding was selected after
the record began — was considered and withdrawn before anything was committed. It would have made
the account's first eighteen months match nothing published, and it bought nothing the record
needed: the committed rows are what a reader verifies, their starting state is itself timestamped
from month one, and a licensee replicating any of them would replicate it warm.

## What is committed, and when

- **All twelve rows, every month, run or not.** This is the part that matters. It is not possible
  to commit only the rows that later did well, because the roster is fixed and every row is recorded
  whether it was traded, published-but-not-traded, or not run at all.
- **Before the first fill.** The manifest is stamped on the morning of the trade date, after the
  books are fixed at the prior close.
- **A month that failed is still committed**, with `status: not-run` and a stated reason. A gap in
  the chain is the one thing it cannot explain away, so gaps are not permitted.

## Sealed books: the anchor and the key chain

The fingerprints above prove *when* a book was fixed. They do not let anyone read it — and
publishing holdings the moment they are held would hand a reader a forward schedule of what this
account must sell and roughly when, since the exit rule is public.

So each month publishes two things: the **hash** of the book, stamped immediately, and an
**encrypted copy** of it. The plaintext follows later, when a key is released.

`commit_anchor.json` in this repository is the root of that scheme. It contains `K_0`, published
once, before any month was sealed. Keys come from a reverse hash chain:

```
K_N  = a secret seed, held offline on paper      N = 1200 months
K_n  = SHA256(K_(n+1))
K_0  = the anchor in this repository             it decrypts nothing — month 1 uses K_1
```

When the key for month *m* is released, anyone can hash it forward and it must land exactly on
`K_0`. A released key is therefore **self-authenticating**: a fabricated one cannot reach the anchor.

The property that matters is what this makes impossible. Releasing `K_12` also yields `K_11 … K_1`,
one hash each — so **months cannot be disclosed selectively.** A good month cannot be revealed while
a bad one is withheld, because revealing the later key hands over every earlier one. Disclosure only
moves forward and never has holes. Going the other way, deriving `K_13` from `K_12`, requires
inverting SHA-256.

If the seed is ever lost, only convenience is lost: the plaintext is published directly and still
verifies against the hash that was stamped before the month began.

Each sealed row is an `AES-256-GCM` envelope under a key derived from `K_m` by HKDF-SHA256, with
the row id bound as authenticated data. The nonce is a **synthetic IV** — derived from the
plaintext as well as the key and the slot — so re-sealing an identical book reproduces its
ciphertext exactly, while two different books can never share a nonce — so a ciphertext moved to another row's filename fails to
open rather than opening into the wrong slot. The plaintext is exactly the bytes `book_sha256`
covers, so decrypting and hashing returns the manifest's own number with nothing to normalise
first. The manifest also records `sealed_sha256`, the hash of the envelope itself: the published
ciphertext is as fixed as the plaintext, and no second envelope can be produced later.

**Status: the anchor is published and the sealing pipeline is built and tested. No month has been
committed yet.** The first manifest will be the first record in the chain.

## The pre-registration, hashed every month

Every manifest carries `lineage.pre_registration` — the SHA-256 of the project's register **and its
length in bytes**. The register is append-only, so those two fields make each month's version
checkable forever without keeping a single old copy:

```python
import hashlib, json
m = json.load(open("manifests/manifest_2026-08.json"))["lineage"]["pre_registration"]
raw = open("PRE_REGISTRATION.md", "rb").read()          # today's file, whatever it has grown to
assert hashlib.sha256(raw[:m["bytes"]]).hexdigest() == m["sha256"]
```

Truncate the current file to the recorded length and hash it. This is not merely a convenience: it
**fails** if any earlier text was edited — a typo fix, a reformat, a tidied section — and it fails
for that month and every month before it at once. So the register cannot be quietly rewritten after
the fact, and a correction to an earlier entry has to be a new appended entry saying what was
wrong, which is how it already works.

The register itself is not published here. What is published is a running proof of what it said and
when, which grows with it and needs no reveal.

## Layout

```
commit_anchor.json                   K_0 — the root of the key chain, published once
manifests/manifest_YYYY-MM.json      the month's commitment
manifests/manifest_YYYY-MM.json.ots  its OpenTimestamps proof
sealed/YYYY-MM/<row id>.seal.json    the month's books, encrypted under K_m
keys/key_month_N.txt                 a released key — appears only when a month is disclosed
book_commitments.jsonl               every manifest in order — the chain, in one file
allowed_signers                      the key commits are signed with
```

The sealed books are published **with** the manifest, not later: the hash proves when a book was
fixed, but only the ciphertext carries the book itself, and a ciphertext handed over after the fact
would leave a reader depending on us to produce a plaintext. Keys are published separately and
deliberately — a routine monthly push can never disclose the record as a side effect.

A **return ledger does not exist yet**. When realised returns are published they will appear here
and this section will say so; until then nothing in this repository reports performance.

## How to verify

Everything below can be run by anyone. None of it requires trusting the author, GitHub, or any
service still being online.

### 1. The commits are signed

```
git config gpg.ssh.allowedSignersFile allowed_signers
git verify-commit HEAD
```

Expect `Good "git" signature for jeog.dev@gmail.com with ED25519 key
SHA256:wXuM+ivfsGtTir2h479KucezBHLbx5V6pSGFAXWsbN8`.

That fingerprint is the whole of the identity claim — check it against the key published on the
author's GitHub account. **The signature proves who; the timestamp below proves when.** Neither
proves the other, which is why both are here.

### 2. The timestamp is real

```
pip install opentimestamps-client
ots verify manifests/manifest_YYYY-MM.json.ots
```

A proof reading `pending` is anchored only by calendar servers and still depends on them. An
upgraded proof carries a **Bitcoin block header** and depends on nobody. Proofs are upgraded and
re-committed within a few days of stamping; a proof still pending long after its month is a defect
worth asking about.

### 3. The chain has not been rewritten

Each manifest carries `prev_manifest_sha256` — the previous month's `chain` — and
`chain = sha256(prev || body_sha)`. The first month's `prev` is 64 zeros. Recompute the sequence:

```python
import json, hashlib, glob
prev = "0" * 64
for p in sorted(glob.glob("manifests/manifest_*.json")):
    m = json.load(open(p))
    keys = ("protocol", "schema", "chain_index", "month", "stamp_date", "sigdig",
            "roster", "roster_changes", "primitives", "book_files", "lineage", "missing")
    if "not_run_acknowledgement" in m:          # a month that did not run carries its reason
        keys += ("not_run_acknowledgement",)
    body = {k: m[k] for k in keys}
    body_sha = hashlib.sha256(
        json.dumps(body, sort_keys=True, separators=(",", ":")).encode()).hexdigest()
    assert body_sha == m["body_sha"], p
    assert m["prev_manifest_sha256"] == prev, p
    assert hashlib.sha256((prev + body_sha).encode()).hexdigest() == m["chain"], p
    prev = m["chain"]
print("chain intact, head", prev)
```

Altering, inserting, deleting or reordering **any** month breaks every hash after it.

### 4. A released key really belongs to this chain

```python
import hashlib, json
m = 12                                                  # the month index the key is for
k = bytes.fromhex(open("keys/key_month_12.txt").read().strip())
for _ in range(m):
    k = hashlib.sha256(k).digest()
assert k.hex() == json.load(open("commit_anchor.json"))["anchor_k0"]
```

If the walk does not land on the anchor, the key is not from this chain. Every earlier month's key
appears along the way, which is why disclosure cannot skip a month.

### 5. The sealed books are the ones committed — and, once released, what they contain

Before any key exists, the ciphertext can already be checked against the record: each roster row's
`sealed_sha256` names the exact envelope bytes in `sealed/<month>/`.

Once a key is released, decrypt and confirm the plaintext is the book the manifest hashed:

```python
import base64, hashlib, json
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
from cryptography.hazmat.primitives.kdf.hkdf import HKDF

K = bytes.fromhex(open("keys/key_month_1.txt").read().strip())
m = json.load(open("manifests/manifest_2026-08.json"))
for r in m["roster"]:
    if not r["sealed_sha256"]:
        continue
    raw = open(f"sealed/{m['month']}/{r['id']}.seal.json", "rb").read()
    assert hashlib.sha256(raw).hexdigest() == r["sealed_sha256"], r["id"]
    e = json.loads(raw)
    d = lambda s, n: HKDF(algorithm=hashes.SHA256(), length=n, salt=None,
                          info=s.encode()).derive(K)
    pt = AESGCM(d(f"QA-SEAL-2|{e['month']}|{e['id']}|key", 32)).decrypt(
        base64.b64decode(e["nonce"]), base64.b64decode(e["ct"]),
        f"QA-SEAL-2|{e['month']}|{e['chain_index']}|{e['id']}".encode())
    assert hashlib.sha256(pt).hexdigest() == r["book_sha256"], r["id"]
    print(r["id"], "OK", json.loads(pt)["long"])
```

`sha256` binds the pre-image: no other file can produce that hash. The nonce is derived rather than
random, so re-sealing a month reproduces its ciphertext byte for byte — an honest re-run does not
look like a changed record.

## The canonical form

Hashes are taken over content, not bytes, because the generator is not byte-deterministic — a hash
over raw floats would change on an honest re-run.

- `json.dumps(obj, sort_keys=True, separators=(",", ":"))`
- every float rounded to **8 significant digits**
- negative zero normalised to `0.0`

Changing the serialisation, the rounding, the chain rule, or the roster definition is a **new
protocol version**, not an edit to this one. A rule that can be revised in place is not a
commitment. Every manifest states its protocol version.

## Row statuses

| status | meaning |
|---|---|
| `run` | the account actually held this book |
| `published-not-traded` | a published product's own treatment, committed but not held |
| `reference-not-run` | a published treatment shown for comparison, not traded |
| `not-run` | no book existed; a reason is required |

Only `run` asserts a live position. The distinction is deliberate: a manifest claiming a product was
traded when it was not would be exactly the misstatement this record exists to prevent.

## What this does NOT prove

Stated here rather than left to be discovered.

- **The published history is a backtest, and it is not verifiable by anyone — including the author.**
  Everything behind these books was fitted on one 238-month sample with no holdout. This chain
  exists to accumulate the out-of-sample record that history does not have.
- **It does not prove no other strategy was running alongside.** A timestamp proves that something
  existed at a time; it cannot prove it was the only thing. Nothing published here rules out that
  other chains were kept and abandoned. What constrains it is that this one is public and
  attributable, that the roster is fixed and cannot be pruned to the survivors, and that the
  configuration hash in every manifest would make a substituted specification visible. The only
  complete answer to this objection is a third-party calculation agent, which this is not.
- **Return verification is not open to everyone.** Anyone can check the hashes, the signature, the
  timestamp and the key chain. Recomputing returns from committed weights additionally requires the
  same licensed vendor data, so that step is available only to someone who holds it.
- **It says nothing about execution.** The books record intended weights. Realised fills differ, and
  nothing here yet records that difference — when a return ledger exists it will, and this line will
  say so.

## Book disclosure

Books are sealed at commitment and disclosed by releasing the key for a month, which discloses that
month and every earlier one. **The lag between commitment and disclosure is not yet fixed**, and must
be settled before the first commitment and stated here permanently — a disclosure rule that changes
later is not a rule.

---

*Manifests are generated by `commit_books.py` in the author's private repository. This repository
receives manifests, proofs, the anchor and released keys only; no code and no data are published
here.*
