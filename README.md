# Cadence Compact Format (CCF)

CCF is a binary data format defined in [ccf_specs.md](ccf_specs.md) at https://github.com/onflow/ccf.

CCF-basesd protocols can balance trade-offs in their own specifications. For example:
- CCF specifies deterministic encoding requirements and makes it optional.
- CCF-based protocols MUST specify when to comply with the optional CCF requirements.

It is outside the scope of CCF to specify each CCF-based protocol (e.g., Events).  Protocols are responsible for specifying aspects outside the scope of CCF (e.g., when to comply with each optional CCF requirement and how to encode version numbers).

## CBOR (RFC 8949) and CBOR Sequences (RFC 8742)

A valid CCF data item is also a valid CBOR data item.

Some protocols may want to specify use of CBOR Sequences. For example:
- First CBOR data item contains metadata specified by each protocol (e.g., version and data item count).
- Next CBOR data item(s) are CCF encoded data item(s) representing Cadence data.

## Status

CCF 1.0.0 (March 31, 2025) was released with support for new data types added in Cadence 1.0. 

CCF 1.0.0 specification supports two modes of encoding Cadence data:
- fully self-describing mode (includes type definitions in the same CCF data item)
- partially self-describing mode (can omit type definitions from CCF data item)

A CCF codec implementing the "fully self-describing mode" of CCF is at:

https://github.com/onflow/cadence/tree/master/encoding/ccf

The "partially self-describing mode" has not yet been implemented by any CCF codec.

### Prior Versions

CCF 1.0.0-RC3 did not support new Cadence types added to Cadence after August 4, 2023.

A CCF codec partially supporting CCF 1.0.0-RC3 was deployed in [June 2023 spork](https://developers.flow.com/concepts/nodes/node-operation/upcoming-sporks).

## Introduction

CCF is a binary data format that allows compact, efficient, and deterministic encoding of Cadence external values.

Cadence external values (e.g., events, transaction arguments, etc.) have been encoded using JSON-CDC, which is inefficient, verbose, and doesn't define deterministic encoding.

The same `FeesDeducted` event on the Flow blockchain can be encoded to:
- 298 bytes in JSON-CDC (minified).
- 118 bytes in CCF (fully self-describing mode).
- &nbsp;20 bytes in CCF (partially self-describing mode).

CCF defines all requirements for deterministic encoding (sort orders, shortest forms, and Cadence-specific requirements) so that CCF codecs implemented in different programming languages can produce the same deterministic encodings.

Some requirements (such as "Deterministic CCF Encoding Requirements") are specified as optional. The specification of each CCF-based protocol determines how it uses optional CCF requirements. This allows each protocol to balance trade-offs such as compatibility, determinism, speed, encoded data size, etc.

CCF allows efficient detection of malformed messages without creating Cadence objects. More costly validation is performed only on well-formed messages.

CCF uses a subset of the Concise Binary Object Representation (CBOR) format. CBOR is a binary data format specified by [RFC 8949](https://www.rfc-editor.org/info/std94) and designated by IETF as an [Internet Standard](https://www.ietf.org/rfc/std-index.txt) (STD&nbsp;94). CBOR is designed to be relevant for decades and is used by data formats and protocols such as [W3C&nbsp;WebAuthn](https://www.w3.org/TR/webauthn-2/), Compacted-DNS&nbsp;([RFC&nbsp;8618](https://www.rfc-editor.org/rfc/rfc8618.html)), COSE&nbsp;([IETF&nbsp;STD&nbsp;96](https://www.rfc-editor.org/info/std96)), CWT&nbsp;([RFC&nbsp;8392](https://www.rfc-editor.org/info/rfc8392)), etc.

## History

Commit history of `ccf_specs.md` file prior to Nov 29, 2022 are associated with `README.md` file.

Old branches can be found at https://github.com/fxamacker/ccf_draft.

## License

CCF is licensed under the terms of the Apache License, Version 2.0. See [LICENSE](LICENSE) for more information.

