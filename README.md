
# Cadence Compact Format (CCF)

Cadence Compact Format (CCF) is a data format designed for compact, efficient, and deterministic encoding of [Cadence](https://github.com/onflow/cadence) external values.  CCF is defined in [ccf_specs.md](ccf_specs.md).

Cadence is a resource-oriented programming language that introduces new features to smart contract programming.  It's used by [Flow](https://github.com/onflow/flow-go) blockchain and has a syntax inspired by Swift, Kotlin, and Rust. Its use of resource types maps well to the Move language.

CCF can be used as a hybrid data format.  CCF-based messages can be fully self-describing or partially self-describing.  Both are more compact than JSON-based messages.  CCF-based protocols can send Cadence metadata just once for all messages of that type.  Malformed data can be detected without Cadence metadata and without creating Cadence objects.

CCF obsoletes [JSON-Cadence Data Interchange Format](https://developers.flow.com/cadence/json-cadence-spec) for use cases that do not require JSON.

### Introduction

Cadence external values (e.g. transaction arguments, events, etc.) have been encoded using JSON-Cadence Data Interchange format, which is human-readable, verbose, and doesn't define deterministic encoding.

CCF is a binary data format that allows more compact, efficient, and deterministic encoding of Cadence external values.  Consequently, the CCF codec in Cadence is faster, uses less memory, encodes deterministically, and produces smaller messages than the JSON-CDC codec.

A real `FeesDeducted` event can encode to:
- 298 bytes in JSON-CDC (minified).
- 118 bytes in CCF (fully self-describing mode).
- ~20 bytes in CCF (partially self-describing mode) with 12 bytes for data and ~8 bytes for type id (counter, hash, etc.)

Unlike prior formats, CCF defines all requirements for deterministic encoding (sort orders, smallest encoded forms, and Cadence-specific requirements) to allow CCF codecs implemented in different programming languages to deterministically produce identical messages.

For security, CCF was designed to allow efficient detection and rejection of malformed messages without creating Cadence objects.  This allows more costly checks (e.g. validity) to be performed only on well-formed messages.

CCF leverages vendor-neutral Internet Standards such as CBOR (RFC 8949), which is designed to be relevant for decades.

## Internet Standards

CCF uses a subset of CBOR and Core Deterministic Encoding Requirements which are defined in [RFC 8949](https://www.rfc-editor.org/rfc/rfc8949).  CCF specification document uses CDDL (Concise Data Definition Language) notation and EDN (Extended Diagnostic Notation).  CDDL and EDN are defined in [RFC 8610](https://www.rfc-editor.org/rfc/rfc8610).  

RFC 8949 and RFC 8610 are [Internet Standards](https://en.wikipedia.org/wiki/Internet_Standard) designed to be relevant for many years (not just regular RFCs).

## Status

- CCF specification is currently a release candidate (RC1) with cleanup underway.
- CCF codec (written in Go) was merged into [Cadence repository](https://github.com/onflow/cadence) with API compatible with JSON-CDC codec.  
  - CCF codec was added by https://github.com/onflow/cadence/pull/2364

### Next steps

- CCF specification will be cleaned up and RC1 status will be replaced by RC2.
- Fuzz tests will be run by the Cadence team for each PR that has changes affecting CCF codec.

## Timeline
- Sep-Oct 2022 - As requested, paused onboarding in order to work fulltime on reviewing checkpointer v6 (onflow/flow-go repo).
- Oct 18, 2022 - Resume onboarding of Cadence external value encoding and requirements.
- Nov 17, 2022 - Share the abridged first draft of CCF with Cadence team and Ramtin for initial sanity check.
- Nov 22, 2022 - First team meeting about abridged draft of CCF to present and discuss revision [20221122b](https://github.com/fxamacker/ccf_draft/blob/2594c4859e51715bb9e770cc42542eb31278cfc4/README.md).
- Nov 29, 2022 - Second team meeting about draft of CCF to present and discuss revision [20221129b](https://github.com/fxamacker/ccf_draft/blob/2c9541a90de968413ec34d31dcf2444949dbce1e/ccf_specs.md).
- Dec 9, 2022 - Merged [PR 35](https://github.com/fxamacker/ccf_draft/pull/35) to add more Cadence types and reassign CBOR tag values. The only Cadence type that is known to be missing from CCF specs is `cadence.PathLink` (blocked by https://github.com/onflow/cadence/issues/2167).
- Dec 15, 2022 - Updated CCF codec (WIP) to incorporate latest CCF specs.  For example, updates to CCF specs from PRs [30](https://github.com/fxamacker/ccf_draft/pull/30), [31](https://github.com/fxamacker/ccf_draft/pull/31), [32](https://github.com/fxamacker/ccf_draft/pull/32), and [35](https://github.com/fxamacker/ccf_draft/pull/35).  All existing CCF codec tests pass (e.g. JSON-Cadence tests ported and modified to CCF).
- Feb 14, 2023 - ABRIDGED DRAFT -> DRAFT.  Third team meeting about draft of CCF to present and discuss unmerged revision [20230214a](https://github.com/fxamacker/ccf_draft/blob/2d6dcb84fba079ebb995a6e55296ca081332a6a4/ccf_specs.md).
- Feb 17, 2023 - DRAFT -> RC1 with revision [20230217a](https://github.com/fxamacker/ccf_draft/blob/4e8f42db29925bd15481301729eca1b52852dcb4/ccf_specs.md)
- Mar 1, 2023 - Open PR 2364 to add CCF codec (about +15,000 lines and `go test -cover` reporting 77%).
- Mar 2023 - Paused in order to work on Atree v0.6: https://github.com/onflow/atree/pull/295
- Mar-Apr 2023 - Updated PR 2364 to match new changes to Cadence that affect external values, incorporate review feedback, and add more tests.
- Apr 5, 2023 - As requested, reduce hours spent on CCF to begin work on Atree register inlining.  https://github.com/onflow/atree/issues/292
- Apr 13, 2023 - Merged PR 2364 to add CCF codec (+20,857 lines, `go test -cover` reported 83%, fuzz tested many billions of executions).

## Preliminary Size and Benchmark Comparisons

We are not comparing apples to apples.  Prior formats (CBF and JSON-Cadence Data Interchange) didn't specify requirements for validity, sorting, etc.

- CCF encoder sorts events data for deterministic encoding.
- CCF decoder verifies that events data are well-formed and sorted.

At this time, CCF decoder doesn't include the option to check for "Preferred Serialization" (encoding to smallest size).

#### Size Comparisons

| Encoding | Event Count | Encoded size | Comments | 
| --- | --- | --- | --- |
| JSON | 48,309 |13,858,836 | JSON-Cadence Data Interchange Format |
| CCF | 48,309 |  6,159,931 | CCF in fully self-describing and deterministic mode |

CCF's partially self-describing mode would be even smaller (roughly 1/14 the size of JSON) in some use cases.

#### Speed and Memory Comparisons

These informal and preliminary benchmarks used commit f911063 in https://github.com/onflow/cadence/pull/2364.

```
$ benchstat bench_json_events_48k.log bench_ccf_events_48k.log 
goos: linux
goarch: amd64
pkg: github.com/onflow/cadence/encoding/ccf
cpu: 13th Gen Intel(R) Core(TM) i5-13600K
                     │ bench_json_events_48k.log │      bench_ccf_events_48k.log       │
                     │          sec/op           │   sec/op     vs base                │
EncodeBatchEvents-20                 96.61m ± 4%   70.73m ± 3%  -26.79% (p=0.000 n=10)
DecodeBatchEvents-20                 647.7m ± 3%   157.5m ± 3%  -75.68% (p=0.000 n=10)
geomean                              250.1m        105.5m       -57.81%

                     │ bench_json_events_48k.log │       bench_ccf_events_48k.log       │
                     │           B/op            │     B/op      vs base                │
EncodeBatchEvents-20                32.45Mi ± 0%   25.82Mi ± 0%  -20.45% (p=0.000 n=10)
DecodeBatchEvents-20               234.97Mi ± 0%   56.16Mi ± 0%  -76.10% (p=0.000 n=10)
geomean                             87.32Mi        38.08Mi       -56.39%

                     │ bench_json_events_48k.log │      bench_ccf_events_48k.log       │
                     │         allocs/op         │  allocs/op   vs base                │
EncodeBatchEvents-20                 756.6k ± 0%   370.4k ± 0%  -51.05% (p=0.000 n=10)
DecodeBatchEvents-20                 4.746M ± 0%   1.288M ± 0%  -72.86% (p=0.000 n=10)
geomean                              1.895M        690.7k       -63.55%
```

### Event Data Details

The 48,309 events used in comparisons are from a transaction on mainnet with unusually high number of events.

There were 9 event types.  These 3 event types had over 15,000 events each: `FlowToken.TokensDeposited`,  `FlowToken.TokensWithdrawn`,  `FlowIDTableStaking.DelegatorRewardsPaid`.

To simplify benchmark code (it's Sunday night), all event values for each event type are the same (i.e. the values are from the first event of that type).

These benchmark results are preliminary and subject to change.

## Notes

Draft of CCF was originally in README.md and moved to ccf_specs.md on Nov 29, 2022. Given this, the initial commit history of the CCF specification is associated with the README.md file rather than ccf_specs.md.

## License

CCF is licensed under the terms of the Apache License, Version 2.0. See [LICENSE](LICENSE) for more information.
