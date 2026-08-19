# Reader observation public contract — `reader-observation/v1`

## Status

This document defines the first transport-neutral public observation contract for the daily MegaLenz on-chain reader. It changes no methodology rule and no score. The implementation repository remains private.

The governing distinction is:

> observation publication != evidence verification != scoring eligibility

An incomplete reader attempt is still a factual observation. Recording that observation must not turn an unknown, missing or unavailable fact into a failure or zero.

## Four independently dated concepts

### Observation freshness

What the reader actually attempted and observed in the latest structured run. `observationFreshness.attemptedAt` is the exact run-wide timestamp already written by the reader store.

`completedAt` is nullable in v1 because the current snapshot store does not persist an exact run-end timestamp. The contract does not invent one. `lastObservedAt` is the latest timestamp among allow-listed public source observations (for example Blockscout or Sourcify) and is not a substitute for completion time.

### Evidence state

Per-address evidence is represented on separate axes:

1. `onchainPresence`
2. `officialAttribution`
3. `publishedSource`
4. `reproducibleBuild`
5. `verifiedBytecode`
6. `teamSubmitted` provenance when applicable

The axes do not imply one another. An officially attributed address is not thereby source-verified; an explorer source record is not thereby a reproducible build; a reproducible-build claim would not itself be accepted bytecode verification without the strict verification gate.

### Scoring eligibility

`scoringEligibility` describes whether the latest reader evidence may advance the scoring-side override under the existing reader gates. It is not a new scoring rule.

Promotion is a per-protocol fact: a complete protocol whose snapshot became the published scoring basis is `eligible`, even inside a partial run. `eligible_not_promoted` marks the residual case of a complete observation whose snapshot did not become the published basis (e.g. historical pre-advancement store states).

An incomplete protocol is `not_eligible` with machine-readable insufficiency reasons. Its unknown or missing evidence is never coerced to `fail`, `false`, `0`, a tier or a score.

### Last-known scoreable state

`lastKnownScoreableState` reports the snapshot-store prior only when `lastVerifiedReaderSnapshot` exists. Its timestamp is independent of the latest attempt timestamp.

`lastPublishedReaderState` separately reports the timestamp/read-source carried by the currently generated `src/lib/onchain-data.ts` override. This makes a stale scoring/public-override basis visible next to a fresh reader observation without pretending the two were produced at the same time.

## Root shape

The canonical machine-readable object is JSON with this top-level shape:

```json
{
  "schema": "reader-observation/v1",
  "methodologyVersion": "v0.8",
  "observationFreshness": {
    "attemptedAt": "2026-08-07T07:22:38.656Z",
    "completedAt": null,
    "lastObservedAt": "2026-08-07T07:24:00.000Z",
    "outcome": "complete | incomplete | error",
    "operationalStatus": "ok | reader_error",
    "error": null,
    "errorPhase": null
  },
  "sourcesConsulted": {},
  "protocols": [],
  "semantics": {
    "observationPublicationIsScoringValidation": false,
    "missingEvidenceIsFailure": false,
    "incompleteObservationCanBePublished": true,
    "scoringGateRemainsStrict": true
  }
}
```

Serialization is recursively canonicalized and deterministic.

## Per-protocol observation

Each protocol record contains:

- slug and name;
- latest reader outcome and attempted timestamp;
- explicit insufficiencies;
- allow-listed reader diagnostics (contract coverage, dynamic resolution coverage, explorer coverage, anomalies, permissions-book reachability and Safe-probe coverage);
- resolution/discovery notes already produced by the reader;
- addresses actually present in the latest attempt payload;
- criterion-evidence facts from the reader payload when present;
- scoring eligibility;
- last promoted strict reader snapshot, if one exists;
- last generated scoring/public-override timestamp, if one exists.

The projection does not fabricate configured addresses that were not actually observed in the attempt. A fatal or pre-read path therefore cannot become a list of apparently checked contracts merely because those addresses exist in reader configuration.

## Address origin and provenance

For an address actually observed by the reader, the projection classifies its origin as one of:

- `configured` — matched to the reader's explicit contract seed;
- `on_chain_discovery` — carried by the reader's discovered-contract output;
- `dynamic_resolution` — tied to a current-run resolution note such as a storage-slot or getter-derived implementation;
- `reader_observed` — observed but not safely attributable to one of the above from the current structured fields.

Configured contracts inherit only the existing `verification_source` provenance tag and purpose. No provenance is invented from an address match alone.

Official attribution uses the same explicit taxonomy already used by the contract-evidence ladder. `on_chain_probe`, discovery and third-party provenance are not silently promoted to protocol attribution.

## Public source records

### Blockscout

The reader snapshot persists Blockscout facts, not the exact request endpoint. Therefore v1 exposes `provider: blockscout` and sets `endpoint: null` rather than reconstructing or guessing a URL.

The allow-list includes observation time, fetch success/error, contract/source-verification status, proxy/implementation information, contract/compiler metadata when present, creator/creation transaction and public tags.

### Sourcify

The reader persists the exact Sourcify endpoint and identity-check record, so v1 may expose those fields directly along with match status, refusal reason, raw match classifications and timestamps.

A Sourcify refusal is not a generic failure. `not-found`, timeout, network error and an unsupported/invalid response remain distinguishable.

## Evidence-state vocabulary

The contract uses factual states including:

- `established`
- `not_established`
- `not_found`
- `unavailable`
- `unknown`

`not_found` is always scoped to the sources actually consulted. For example, when Blockscout reports an address unverified and Sourcify reports `not-found`, the projection may say no source was found in those two consulted verification services. It may not claim that source code does not exist anywhere else.

`reproducibleBuild` remains `unknown` today. Compiler version or optimizer settings alone are useful partial metadata but are not a reproducible-build proof; an exact deployed source commit and the complete compiler input are still missing from the current canonical schema.

## Exit semantics

| Reader exit | Private audit store | `reader-observation/v1` | Scoring override (`onchain-data.ts`) | Workflow signal |
| --- | --- | --- | --- | --- |
| `0` | advances | advances | fresh entries for every target | green if validation passes |
| `2` partial | advances | advances with insufficiencies | complete targets advance; non-scoreable targets carried forward byte-for-byte (or absent when never published) | green with a visible non-scoreable warning |
| `3` reader error | advances | advances with error/not-read facts | preserved byte-for-byte | green with a visible non-scoreable warning |
| `10+` fatal | no structured new store write | **no new observation is generated** | preserved | fatal red |

Per-protocol advancement (owner decision 2026-08-11): what is verified is published; what is not stays explicitly on its last published state, keeping its old `lastReadAt` visible. A red workflow now always signals a technical failure, never a structured data outcome.

## Confidentiality and allow-list policy

The projection is built field-by-field. It does **not** serialize the raw snapshot store and then redact a blacklist.

That direction matters: an unknown future private field remains private by default until explicitly reviewed and mapped into the public contract.

The projection must never expose:

- secrets, tokens or credentials;
- private RPC or infrastructure endpoints;
- internal prompts;
- private agent traces/logs;
- anti-abuse implementation;
- arbitrary unknown snapshot fields;
- private implementation details that are not themselves public evidence facts.

Tests inject representative private-looking fields into the input and assert that none survives serialization.

## Current publication boundary

The workflow writes the deterministic projection to:

`src/data/reader-observation.json`

During this first PR that file is still committed only to the **private** implementation repository. This establishes the stable data contract and daily generation semantics; it does **not** yet make the observation publicly reachable every day.

A separate owner-authorized transport must publish the allow-listed projection without turning each reader run into an application deployment. See `docs/adr-reader-observation-transport.md`.
