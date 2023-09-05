
# Cadence Compact Format (CCF)

CCF is defined in [ccf_specs.md](ccf_specs.md).

## Status

CCF 1.0.0-RC3 is partially supported by the [CCF codec](https://github.com/onflow/cadence/tree/master/encoding/ccf) deployed in [June 2023 spork](https://developers.flow.com/concepts/nodes/node-operation/upcoming-sporks).

The CCF codec in Cadence is currently used for encoding events in CCF.  Some CCF options (partially self-describing mode) are not yet implemented by the codec.

New Cadence types added to Cadence after August 4, 2023 are not yet added to CCF specs.

## Introduction

CCF is a data format that allows compact, efficient, and deterministic encoding of Cadence external values.

Cadence external values (e.g. events, transaction arguments, etc.) have been encoded using JSON-CDC, which is inefficient, verbose, and doesn't define deterministic encoding.

The same `FeesDeducted` event on the Flow blockchain can encode to:
- 298 bytes in JSON-CDC (minified).
- 118 bytes in CCF (fully self-describing mode).
- &nbsp;20 bytes in CCF (partially self-describing mode).

CCF defines all requirements for deterministic encoding (sort orders, smallest encoded forms, and Cadence-specific requirements) to allow CCF codecs implemented in different programming languages to produce the same deterministic encodings.

Some requirements (such as "Deterministic CCF Encoding Requirements") are defined as optional. Each CCF-based format or protocol can have its specification state how CCF options are used. This allows each protocol to balance trade-offs such as compatibility, determinism, speed, encoded data size, etc.

CCF uses CBOR and is designed to allow efficient detection and rejection of malformed messages without creating Cadence objects. This allows more costly checks for validity, etc. to be performed only on well-formed messages.

CBOR is an [Internet Standard](https://www.ietf.org/rfc/std-index.txt) defined by [IETF&nbsp;STD&nbsp;94](https://www.rfc-editor.org/info/std94). CBOR is designed to be relevant for decades and is used by data formats and protocols such as [W3C&nbsp;WebAuthn](https://www.w3.org/TR/webauthn-2/), C-DNS&nbsp;([IETF&nbsp;RFC&nbsp;8618](https://www.rfc-editor.org/rfc/rfc8618.html)), COSE&nbsp;([IETF&nbsp;STD&nbsp;96](https://www.rfc-editor.org/info/std96)), CWT&nbsp;([IETF&nbsp;RFC&nbsp;8392](https://www.rfc-editor.org/info/rfc8392)), etc.

## History

Commit history of `ccf_specs.md` file prior to Nov 29, 2022 are associated with `README.md` file.

Old branches can be found at https://github.com/fxamacker/ccf_draft.

## License

CCF is licensed under the terms of the Apache License, Version 2.0. See [LICENSE](LICENSE) for more information.

