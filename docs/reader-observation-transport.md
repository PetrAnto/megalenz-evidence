# Public evidence transport contract — `reader-observation-transport/v1`

This document describes how a `reader-observation/v1` payload reaches the public
evidence repository, and how to read what you find there. It is published
alongside the payloads so a third party can interpret them without access to the
MegaLenz implementation, which stays private.

Companion document: [`reader-observation-v1.md`](reader-observation-v1.md) defines
the payload itself.

Governing distinctions, unchanged by this transport:

> observation publication != evidence verification != scoring eligibility
>
> reader publication != application deployment

## What is published

| Path | Content |
| --- | --- |
| `latest.json` | The most recent published `reader-observation/v1` payload, byte-for-byte |
| `latest.transport.json` | The transport sidecar describing `latest.json` |
| `history/YYYY/MM/DD/<attemptId>.json` | The payload of one reader attempt, byte-for-byte |
| `history/YYYY/MM/DD/<attemptId>.transport.json` | The transport sidecar for that history payload |
| `README.md`, `LICENSE`, `docs/` | Human-maintained, changed by reviewed pull request only |

`latest.json` is a copy, not a symlink, so plain HTTP consumers can fetch it
without resolving git links.

## The payload is transport-neutral

`reader-observation/v1` carries **no** publication metadata. The bytes published
here are identical to the bytes committed in the private implementation
repository — no wrapper, no added field, no re-pretty-print, no key reordering,
no trailing-newline change. Provenance lives only in the sidecar.

If the exact bytes cannot be reproduced, publication fails and nothing is
written.

## Sidecar shape

```json
{
  "schema": "reader-observation-transport/v1",
  "payloadSchema": "reader-observation/v1",
  "payloadPath": "latest.json",
  "payloadSha256": "64-lowercase-hex",
  "attemptId": "20260818T065027.338Z",
  "attemptedAt": "2026-08-18T06:50:27.338Z",
  "publishedAt": "2026-08-18T07:12:04.120Z",
  "source": {
    "repository": "PetrAnto/megalenz",
    "ref": "refs/heads/main",
    "commitSha": "40-character-hex-sha"
  },
  "publisher": {
    "kind": "github-app-installation",
    "appSlug": "megalenz-evidence-publisher"
  }
}
```

| Field | Meaning |
| --- | --- |
| `payloadSha256` | SHA-256 of the exact payload bytes at `payloadPath`, lowercase hex. Verify it. |
| `attemptId` | `attemptedAt` with `-` and `:` removed. Fractional seconds are preserved exactly. |
| `attemptedAt` | Observation time, copied from the payload. **Not** publication time. |
| `publishedAt` | Publisher UTC stamp taken immediately before the publication commit object is created. It is **not** the GitHub ref-update receipt time: a single atomic commit cannot contain an instant that only exists after it lands. |
| `source.repository` | A stable `owner/name` string, not a fetchable URL. The implementation repository is private and the SHA is not publicly cloneable; the name exists so the SHA is not ambiguous. |
| `source.commitSha` | The private commit containing the published payload. Never a workflow-run URL, artifact URL or private file path. |
| `publisher` | The identity that wrote the commit. |

A sidecar never carries secrets, tokens, private hosts or endpoints, workflow
logs, agent traces or raw snapshot internals.

## Append-only semantics

- History files are **create-once**. They are never rewritten, amended, rebased
  or deleted on the normal path.
- Republishing an attempt that already landed with identical bytes is an
  idempotent no-op: the existing sidecar and its `publishedAt` are preserved.
- The same history path with **different** bytes is a hard failure. The
  publisher refuses; it does not overwrite, rewrite or rewind.
- `latest` moves only to a strictly newer `attemptedAt`. An older attempt that
  arrives late may add its history file but can never rewind `latest`.
- The `main` ref is updated fast-forward-only: every ref update is sent with
  `force: false`, and the publisher has no history-rewriting capability at all.

Corrections are additive: a later reader attempt is a new `attemptedAt` and a new
history file, and any human correction notice is a new file added by a reviewed
pull request. There is no silent replacement path.

### A payload and its sidecar are one unit

A payload without a coherent sidecar is not valid published evidence. Before
publishing, the publisher reads the public tree at one exact commit and checks
every already-published pair it is about to rely on:

- **Both or neither.** A payload and its sidecar are written in the same commit.
  Finding exactly one of them means the tree is damaged, and publication stops.
  A lone sidecar is never treated as "nothing is there".
- **The sidecar must describe its payload.** Transport schema, payload schema,
  a `payloadPath` pointing at that exact file, and the SHA-256 of the bytes
  actually present there — not of the bytes anyone expected to find. That digest
  is computed over the exact octets the repository serves, never over text
  decoded and re-encoded along the way: a UTF-8 round trip strips a byte-order
  mark and rewrites malformed sequences, so it would hash something the
  repository does not hold. Public JSON that is not valid UTF-8, or that carries
  a byte-order mark this publisher never writes, is refused rather than
  normalised.
- **Ordering comes from the payload, not the sidecar.** Which attempt `latest`
  carries is read from `latest.json`'s own
  `observationFreshness.attemptedAt`, and the sidecar must agree with it. A
  correct digest alone is not enough: a sidecar that hashes the right bytes but
  back-dates `attemptedAt` would otherwise make newer public evidence look older
  and eligible for replacement.
- **`latest` must agree with its own history entry.** `latest.json` is a copy of
  one history payload. The history entry it names must exist, be byte-identical
  to it, and carry its own coherent sidecar. A public tree that contradicts
  itself orders nothing.
- **Provenance is checked, not assumed.** `source.repository`, `source.ref`,
  `source.commitSha`, `publisher.kind` and `publisher.appSlug` must all name the
  authorized publisher. A sidecar attributing evidence to some other source or
  App is refused. The publisher writes the identity of the App that actually
  minted its token, not a hard-coded literal.
- **Timestamps must be real, and ordering is exact.** Every field the contract
  calls a UTC instant is validated component by component: impossible calendar
  dates such as `2026-02-30T00:00:00Z` are rejected, never silently rolled
  forward. Comparison runs over those components rather than through a
  millisecond-truncating parser, so `…00.0001Z` is correctly older than
  `…00.0009Z`. Two spellings of the same instant (`…00.1Z` and `…00.10Z`) would
  file one observation under two history paths, so they are refused rather than
  silently reconciled.

Any of these failing stops publication. None of them is ever treated as a
successful no-op, and none is used to decide which attempt is newer.

The publisher also refuses to write at all unless GitHub reports the target
repository as public. This transport publishes evidence; writing the same bytes
into a private repository would look like success while publishing nothing.

Consumers should apply the same rules: verify `payloadSha256` against the bytes
you fetched, check that the sidecar's `attemptedAt` matches the payload's own and
that `latest.json` matches the history entry it names, and treat a mismatch as
unusable data rather than as evidence.

### Security incidents

Public disclosure cannot be undone — forks, clones, caches and indexes retain
copies. The incident path is containment plus an additive public notice, not
history surgery. No rewrite would make already-public bytes unpublished, and this
repository will not claim otherwise.

## Reading transport health

Only three transport states are derivable from this repository:

| State | How you detect it | What it does **not** mean |
| --- | --- | --- |
| `transport_ok` | Sidecar fetched, `publishedAt` younger than 36 hours | — |
| `transport_stale` | Sidecar fetched, `publishedAt` older than 36 hours | Protocol failure, score failure, reader error |
| `transport_unavailable` | `latest.json` or its sidecar could not be fetched | Protocol failure, score failure, reader error |

36 hours is one missed daily 06:00 UTC run plus margin. It is a transport SLA,
not a scoring rule.

When a publication fails, this repository simply stays on its previous successful
commit. **The cause is not published here.** Publisher failure reasons — history
collision, exhausted fast-forward retries, authentication failure, safety
refusal, invalid timestamp — remain private operational diagnostics. Do not infer
a cause from staleness.

Observation and scoring states live in the payload, not in the sidecar:
`outcome == "incomplete"`, `outcome == "error"` / `operationalStatus ==
"reader_error"`, and per-protocol `scoringEligibility`. A transport problem must
never be presented as `not_eligible`, `fail`, `0` or a protocol error.

## Consuming the transport

```text
https://raw.githubusercontent.com/PetrAnto/megalenz-evidence/main/latest.json
https://raw.githubusercontent.com/PetrAnto/megalenz-evidence/main/latest.transport.json
```

No token is required. The publisher runs daily, so a consumer cache of 5–15
minutes is sufficient. Verify `payloadSha256` against the bytes you fetched
before relying on either file.
