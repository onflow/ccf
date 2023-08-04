
# Cadence Compact Format (CCF)

Cadence Compact Format (CCF) is a data format designed for compact, efficient, and deterministic encoding of [Cadence](https://github.com/onflow/cadence) external values.  CCF is defined in [ccf_specs.md](ccf_specs.md).

Cadence is a resource-oriented programming language that introduces new features to smart contract programming.  It's used by [Flow](https://github.com/onflow/flow-go) blockchain and has a syntax inspired by Swift, Kotlin, and Rust. Its use of resource types maps well to the Move language.

CCF can be used as a hybrid data format.  CCF-based messages can be fully self-describing or partially self-describing.  Both are more compact than JSON-based messages.  CCF-based protocols can send Cadence metadata just once for all messages of that type.  Malformed data can be detected without Cadence metadata and without creating Cadence objects.

CCF obsoletes [JSON-Cadence Data Interchange Format](https://developers.flow.com/cadence/json-cadence-spec) for use cases that do not require JSON.

### Introduction

CCF is a data format that allows compact, efficient, and deterministic encoding of Cadence external values.

Cadence external values (e.g. events, transaction arguments, etc.) have been encoded using JSON-CDC, which is inefficient, verbose, and doesn't define deterministic encoding.

The same `FeesDeducted` event on the Flow blockchain can encode to:
- 298 bytes in JSON-CDC (minified).
- 118 bytes in CCF (fully self-describing mode).
- &nbsp;20 bytes in CCF (partially self-describing mode).

CCF defines all requirements for deterministic encoding (sort orders, smallest encoded forms, and Cadence-specific requirements) to allow CCF codecs implemented in different programming languages to produce the same deterministic encodings.

Some requirements (such as "Deterministic CCF Encoding Requirements") are defined as optional.  Each CCF-based format or protocol can have its specification state how CCF options are used.  This allows each protocol to balance tradeoffs such as compatibility, determinism, speed, encoded data size, etc.

CCF uses CBOR and is designed to allow efficient detection and rejection of malformed messages without creating Cadence objects. This allows more costly checks for validity, etc. to be performed only on well-formed messages.

CBOR is an [Internet Standard](https://www.ietf.org/rfc/std-index.txt) defined by [IETF&nbsp;STD&nbsp;94](https://www.rfc-editor.org/info/std94).  CBOR is designed to be relevant for decades and is used by data formats and protocols such as [W3C&nbsp;WebAuthn](https://www.w3.org/TR/webauthn-2/), C-DNS&nbsp;([IETF&nbsp;RFC&nbsp;8618](https://www.rfc-editor.org/rfc/rfc8618.html)), COSE&nbsp;([IETF&nbsp;STD&nbsp;96](https://www.rfc-editor.org/info/std96)), CWT&nbsp;([IETF&nbsp;RFC&nbsp;8392](https://www.rfc-editor.org/info/rfc8392)), etc.

## Contributing

If you would like to contribute to CCF, have a look at the [contributing guide](https://github.com/onflow/ccf/blob/main/CONTRIBUTING.md).

## License

CCF is licensed under the terms of the Apache License, Version 2.0. See [LICENSE](LICENSE) for more information.

