# Signal witness

This repository is the public witness of an append-only signal journal.
It holds no signals, no instruments and no prices. It holds only the
**head** of the journal's hash chain and the **number of records**, one line
per change.

The journal links every record to the one before it with a hash chain. A
single edited record breaks the chain. A complete recomputation of the
chain after an edit would still pass the check. What gives it away is a
head published earlier, here.

**The evidence is the history of this branch, not the current file.** A
branch rule blocks force pushes and deletion of `main`. It does not stop an
ordinary commit that changes `heads.jsonl`, and the owner, as
administrator, can lift the rule or delete the repository. Any such change
is visible to everyone who kept a copy of the history (a clone). So the
check below walks the history: no merge commits, and `heads.jsonl` only
ever grows - each version starts with the previous one.

## `heads.jsonl`

One JSON object per line:

```json
{"utc": "2026-10-02T16:00:05Z", "records": 0, "head": "0000000000000000000000000000000000000000000000000000000000000000", "schema": 4}
```

| Field | Meaning |
|---|---|
| `records` | the number of records in the journal chain at publication |
| `head` | the `chain_hash` of record number `records`; 64 zeros when there are no records |
| `schema` | the version of the journal format the hash formula belongs to |
| `utc` | the clock of the publishing machine |

**`utc` is a label, not evidence.** The publisher writes whatever its clock
says. The evidence is the order of commits in this branch and the times
GitHub received them. A line proves that a journal with this head and this
many records existed no later than the commit that added the line.

A new line is added only when the head changes. Before every publication
the history and every line are checked against the journal. A line or a
commit that contradicts it stops the publisher and stays here for a human to
see.

## Checking a copy of the journal

The check is `journal_witness.py`, which comes with the copy of the journal,
not with this repository. It uses only the Python standard library (3.9
or later) and needs no code of the journal's package.

The main check runs on a clone, because the evidence is the history:

```
git clone https://github.com/<owner>/<this repository>.git witness
python3 journal_witness.py --db <copy of the journal> --repo witness
```

The clone must be complete, without `--depth`: a shallow clone holds only
the last commits, and the check refuses it.

Keep the clone and pull it from time to time (`git pull --ff-only`). If a
pull ever refuses to fast-forward, the published history was rewritten.
Your clone is the proof of that.

`--heads witness/heads.jsonl` checks one file alone, without its history.
It cannot see a line that was replaced by a later commit.

- **Exit 0.** The copy's chain verifies, the history follows the rule, and
  every line agrees with the copy. The output is one line of JSON with
  `heads_checked`. It also has `heads_ahead`: lines with more records than
  the copy has, which happens when the copy is older than the latest
  publication.
- **Exit 1.** The chain is broken, the history breaks the rule, or a line
  contradicts the copy. The message names the record, the commit or the
  line number.
- **Exit 2.** The file is not a journal of schema 4, the clone is not a git
  repository, or there is no heads file.

The hash formula, for an independent implementation:

```
chain_hash = sha256((chain_prev + canonical).encode("utf-8")).hexdigest()
canonical  = json.dumps([table, row], sort_keys=True, separators=(",", ":"),
                        ensure_ascii=False, allow_nan=False)
```

- `table` is `signals` or `outcomes`.
- `row` holds every stored column except `chain_prev` and `chain_hash`.
- Records are taken in `chain_n` order across both tables.
- The first record follows 64 zeros.
