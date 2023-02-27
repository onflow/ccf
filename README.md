
# Cadence Compact Format (CCF)

Cadence Compact Format (CCF) is a data format designed for compact, efficient, and deterministic encoding of [Cadence](https://github.com/onflow/cadence) external values.  CCF is defined in [ccf_specs.md](ccf_specs.md).

Cadence is a resource-oriented programming language that introduces new features to smart contract programming.  It's used by [Flow](https://github.com/onflow/flow-go) blockchain and has a syntax inspired by Swift, Kotlin, and Rust. Its use of resource types maps well to the Move language.

CCF can be used as a hybrid data format.  CCF-based messages can be fully self-describing or partially self-describing.  Both are more compact than JSON-based messages.  CCF-based protocols can send Cadence metadata just once for all messages of that type.  Malformed data can be detected without Cadence metadata and without creating Cadence objects.

CCF obsoletes [JSON-Cadence Data Interchange Format](https://developers.flow.com/cadence/json-cadence-spec) for use cases that do not require JSON.

## Internet Standards

CCF uses a subset of CBOR and Preferred Serialization which are defined in [RFC 8949](https://www.rfc-editor.org/rfc/rfc8949).  CCF specification document uses CDDL (Concise Data Definition Language) notation and EDN (Extended Diagnostic Notation).  CDDL and EDN are defined in [RFC 8610](https://www.rfc-editor.org/rfc/rfc8610).  

RFC 8949 and RFC 8610 are [Internet Standards](https://en.wikipedia.org/wiki/Internet_Standard) designed to be relevant for many years (not just regular RFCs).

## Status

- CCF specification is currently a release candidate (RC1) with cleanup underway.
- CCF codec currently implements nearly 100% of CCF specs revision [20230216a](https://github.com/fxamacker/ccf_draft/blob/699decd58c82a7566781267a25be4b4019adb464/ccf_specs.md). The CCF codec is being updated locally by @fxamacker and all tests are passing, including JSON-Cadence tests ported and modified to CCF.  Unit tests (including comments) currently total around 7,000 lines.

### Next steps

There are no known codec-impacting changes remaining but some might be discovered during PR review of CCF codec, codec integration, and maybe during fuzz testing.

- CCF specification will be cleaned up and RC1 status will be replaced by RC2 (as a parallel task).
- CCF codec will be updated and opened as a PR in onflow/cadence as soon as it's ready for review.
- More unit tests will be added after the PR for codec is opened or merged in onflow/cadence.
- Fuzz tests will be created and run by the Cadence team before it is used in production.

## Timeline
- Oct 18, 2022 - Resume onboarding of Cadence external value encoding and requirements.
- Nov 17, 2022 - Share the abridged first draft of CCF with Cadence team and Ramtin for initial sanity check.
- Nov 22, 2022 - First team meeting about abridged draft of CCF to present and discuss revision [20221122b](https://github.com/fxamacker/ccf_draft/blob/2594c4859e51715bb9e770cc42542eb31278cfc4/README.md).
- Nov 29, 2022 - Second team meeting about draft of CCF to present and discuss revision [20221129b](https://github.com/fxamacker/ccf_draft/blob/2c9541a90de968413ec34d31dcf2444949dbce1e/ccf_specs.md).
- Dec 9, 2022 - Merged [PR 35](https://github.com/fxamacker/ccf_draft/pull/35) to add more Cadence types and reassign CBOR tag values. The only Cadence type that is known to be missing from CCF specs is `cadence.PathLink` (blocked by https://github.com/onflow/cadence/issues/2167).
- Dec 15, 2022 - Updated CCF codec (WIP) to incorporate latest CCF specs.  For example, updates to CCF specs from PRs [30](https://github.com/fxamacker/ccf_draft/pull/30), [31](https://github.com/fxamacker/ccf_draft/pull/31), [32](https://github.com/fxamacker/ccf_draft/pull/32), and [35](https://github.com/fxamacker/ccf_draft/pull/35).  All existing CCF codec tests pass (e.g. JSON-Cadence tests ported and modified to CCF).
- Feb 14, 2023 - ABRIDGED DRAFT -> DRAFT.  Third team meeting about draft of CCF to present and discuss unmerged revision [20230214a](https://github.com/fxamacker/ccf_draft/blob/2d6dcb84fba079ebb995a6e55296ca081332a6a4/ccf_specs.md).
- Feb 17, 2023 - DRAFT -> RC1 with revision [20230217a](https://github.com/fxamacker/ccf_draft/blob/4e8f42db29925bd15481301729eca1b52852dcb4/ccf_specs.md)


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

CCF's partially self-describing mode would be even smaller (roughly 1/4 the size of JSON) in some use cases.

#### Speed and Memory Comparisons

```
                     │ 48k_events_encode_json.log │     48k_events_encode_ccf.log       │
                     │           sec/op           │   sec/op     vs base                │
EncodeBatchEvents-20                 89.84m ± 17%   69.28m ± 3%  -22.88% (p=0.000 n=10)

                     │ 48k_events_encode_json.log │      48k_events_encode_ccf.log       │
                     │            B/op            │     B/op      vs base                │
EncodeBatchEvents-20                 32.45Mi ± 0%   25.82Mi ± 0%  -20.45% (p=0.000 n=10)

                     │ 48k_events_encode_json.log │     48k_events_encode_ccf.log       │
                     │         allocs/op          │  allocs/op   vs base                │
EncodeBatchEvents-20                  756.6k ± 0%   370.4k ± 0%  -51.05% (p=0.000 n=10)
```

```
                     │ 48k_events_decode_json.log │     48k_events_decode_ccf.log       │
                     │           sec/op           │   sec/op     vs base                │
DecodeBatchEvents-20                  646.2m ± 8%   158.3m ± 5%  -75.50% (p=0.000 n=10)

                     │ 48k_events_decode_json.log │      48k_events_decode_ccf.log       │
                     │            B/op            │     B/op      vs base                │
DecodeBatchEvents-20                234.97Mi ± 0%   56.16Mi ± 0%  -76.10% (p=0.000 n=10)

                     │ 48k_events_decode_json.log │     48k_events_decode_ccf.log       │
                     │         allocs/op          │  allocs/op   vs base                │
DecodeBatchEvents-20                  4.746M ± 0%   1.288M ± 0%  -72.86% (p=0.000 n=10)

```

### Event Data Details

The 48,309 events used in comparisons are from a transaction on mainnet with unusually high number of events.

There were 9 event types.  These 3 event types had over 15,000 events each: `FlowToken.TokensDeposited`,  `FlowToken.TokensWithdrawn`,  `FlowIDTableStaking.DelegatorRewardsPaid`

To simplify benchmark code (it's Sunday night), all event values for each event type are the same (i.e. the values are from the first event of that type).

These benchmark results are preliminary and subject to change.

## Notes

Draft of CCF was originally in README.md and moved to ccf_specs.md on Nov 29, 2022. Given this, the initial commit history of the CCF specification is associated with the README.md file rather than ccf_specs.md.

## License

CCF is licensed under the terms of the Apache License, Version 2.0. See [LICENSE](LICENSE) for more information.
