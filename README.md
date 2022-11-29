
# Cadence Compact Format (CCF)

CCF is defined in [ccf_specs.md](ccf_specs.md).

CCF is a data format designed for compact, efficient, and deterministic encoding of Cadence external values.  This format is intended to be an alternative to and eventually replace [JSON-Cadence Data Interchange Format](https://developers.flow.com/cadence/json-cadence-spec).

CCF leverages [Concise Binary Object Representation (CBOR)](https://www.rfc-editor.org/rfc/rfc8949.html), which is an IETF Internet Standard data format with a data model that is a superset of JSON's data model.  CCF uses a subset of CBOR's data model with CBOR Preferred Serialization to deterministically encode values to their smallest form.  CBOR was designed with security considerations in mind, such as malformed data detection.  CCF codecs can inherit detection of malformed data by using an existing CBOR codec.

CCF separates encoding of Cadence type info and values to avoid unnecessarily repeating Cadence type info (e.g. for a homogenous array, the same Cadence type info isn't repeatedly encoded).  This separation allows more compact encoded size.  Another distinct advantage is support for more compact communication size.  Detachable Cadence type info (for all composite types) enables protocols using CCF to optionally transmit the Cadence type info just once rather than repeating it for every message matching the same type.

## Status

CCF is currently in DRAFT status as of revision 20221129b.

## Notes

Draft of CCF was originally in README.md and moved to ccf_specs.md on Nov 29, 2022. Given this, the initial commit history of the CCF specification is associated with the README.md file rather than ccf_specs.md.

## License

CCF is licensed under the terms of the Apache license. See [LICENSE](LICENSE) for more information.
