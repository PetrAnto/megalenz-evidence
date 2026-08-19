# MegaLenz evidence

This repository publishes the daily MegaLenz `reader-observation/v1` evidence
record. It is not the MegaLenz application source; the implementation remains
private.

## What is here

| Path | Content |
| --- | --- |
| `latest.json` | The most recent published `reader-observation/v1` payload, byte-for-byte |
| `latest.transport.json` | The transport sidecar describing `latest.json` |
| `history/YYYY/MM/DD/<attemptId>.json` | The payload of one reader attempt, byte-for-byte |
| `history/YYYY/MM/DD/<attemptId>.transport.json` | The transport sidecar for that history payload |
| `docs/reader-observation-v1.md` | The payload contract |
| `docs/reader-observation-transport.md` | The transport and sidecar contract |

Read [`docs/reader-observation-transport.md`](docs/reader-observation-transport.md)
before relying on anything here. It explains how to verify a payload against its
sidecar, and what the publication guarantees are.

## How to read it

```text
https://raw.githubusercontent.com/PetrAnto/megalenz-evidence/main/latest.json
https://raw.githubusercontent.com/PetrAnto/megalenz-evidence/main/latest.transport.json
```

No token is required. Verify `payloadSha256` in the sidecar against the bytes you
fetched, and check that the sidecar's `attemptedAt` matches the payload's own
`observationFreshness.attemptedAt`, before treating either as evidence.

## What an observation is, and is not

> observation publication != evidence verification != scoring eligibility

An incomplete reader attempt is still a factual observation. Recording it does
not turn an unknown, missing or unavailable fact into a failure or a zero.
`outcome`, `operationalStatus` and per-protocol `scoringEligibility` live in the
payload and mean exactly what the payload contract says they mean.

Publication here is data transport. It is not a release of the MegaLenz
application, and a transport problem is never a protocol failure, a scoring
failure or a reader failure. Only three transport states are derivable from this
repository — `transport_ok`, `transport_stale` and `transport_unavailable` — and
the cause of a publication failure is deliberately not published here.

## History is append-only

History files are create-once: never rewritten, amended or deleted on the normal
path. `latest.json` only ever moves forward to a strictly newer attempt.
Corrections are additive — a later reader attempt is a new file, and any
correction notice is a new file added by a reviewed pull request.

## License

Data and documents in this repository are published under
[CC BY 4.0](LICENSE). Attribute to MegaLenz and link back to this repository.
