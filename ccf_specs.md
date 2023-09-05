# Cadence Compact Format (CCF)

Author: Faye Amacker  
Status: RC3  
Date: September 4, 2023  
Revision: 20230904a

## Abstract

Cadence Compact Format (CCF) is a data format designed for compact, efficient, and deterministic encoding of [Cadence](https://github.com/onflow/cadence) external values.  Cadence is a modern resource-oriented programming language used by [Flow](https://github.com/onflow/flow-go) blockchain.

CCF messages can be fully self-describing or partially self-describing.  Both are more compact than JSON-based messages.  CCF-based protocols can send Cadence metadata just once for all messages of that type.  Malformed data can be detected without Cadence metadata and without creating Cadence objects.

CCF defines "Deterministic CCF Encoding Requirements" and makes it optional.  CCF codecs implemented in different programming languages can produce the same deterministic encodings.  CCF-based formats and protocols can balance trade-offs by specifying how they use CCF options.

CCF obsoletes [JSON-Cadence Data Interchange Format](https://developers.flow.com/cadence/json-cadence-spec) (JSON-CDC) for use cases that do not require JSON.

## Status of this Document

This document is a release candidate (RC3).

## Copyright Notice

Copyright (c) 2022-2023 Dapper Labs, Inc. and the persons identified as the document authors.

This document is licensed under the terms of the Apache License, Version 2.0. See [LICENSE](LICENSE) for more information.

## Scope

This document specifies Cadence Compact Format.

It is outside the scope of this document to specify individual CCF-based formats or protocols (e.g. events).

## Introduction

CCF is a data format that allows compact, efficient, and deterministic encoding of Cadence external values.

Cadence external values (e.g. events, transaction arguments, etc.) have been encoded using JSON-CDC, which is inefficient, verbose, and doesn't define deterministic encoding.

The same `FeesDeducted` event on the Flow blockchain can encode to:
- 298 bytes in JSON-CDC (minified).
- 118 bytes in CCF (fully self-describing mode).
- &nbsp;20 bytes in CCF (partially self-describing mode).

CCF defines all requirements for deterministic encoding (sort orders, smallest encoded forms, and Cadence-specific requirements) to allow CCF codecs implemented in different programming languages to produce the same deterministic encodings.

Some requirements (such as "Deterministic CCF Encoding Requirements") are defined as optional.  Each CCF-based format or protocol can have its specification state how CCF options are used.  This allows each protocol to balance trade-offs such as compatibility, determinism, speed, encoded data size, etc.

CCF uses CBOR and is designed to allow efficient detection and rejection of malformed messages without creating Cadence objects. This allows more costly checks for validity, etc. to be performed only on well-formed messages.

CBOR is an [Internet Standard](https://www.ietf.org/rfc/std-index.txt) defined by [IETF&nbsp;STD&nbsp;94](https://www.rfc-editor.org/info/std94).  CBOR is designed to be relevant for decades and is used by data formats and protocols such as [W3C&nbsp;WebAuthn](https://www.w3.org/TR/webauthn-2/), C-DNS&nbsp;([IETF&nbsp;RFC&nbsp;8618](https://www.rfc-editor.org/rfc/rfc8618.html)), COSE&nbsp;([IETF&nbsp;STD&nbsp;96](https://www.rfc-editor.org/info/std96)), CWT&nbsp;([IETF&nbsp;RFC&nbsp;8392](https://www.rfc-editor.org/info/rfc8392)), etc.

### Objectives

The goal of CCF is to provide compact, efficient, and deterministic encoding of Cadence external values.  To achieve this:
- CCF uses CBOR's data model with [Preferred Serialization](https://www.rfc-editor.org/rfc/rfc8949.html#name-preferred-serialization) to deterministically encode values to their smallest form.

- CCF separates Cadence type encoding from Cadence value encoding. This has two distinct advantages:

  - More compact encoding.  Cadence type info is not repeatedly encoded unnecessarily in a message.  E.g. for a homogeneous array of a Cadence composite type, each element will not have its Cadence composite type info encoded repeatedly.

  - Detachable Cadence type info. Although Cadence type info is required for decoding, CCF-based protocols can send a message's type info once instead of repeatedly sending it to the same client for all messages of that type.

CCF is designed to support:

- All current and future Cadence types, including composite types.  CCF supports schemaless encoding and is extensible for new Cadence types.

- Compact encoding.  Smaller encoded size is produced by:
  - CBOR's data model with CBOR Preferred Serialization, which produces more compact encoding than JSON.
  - Separate encoding of Cadence types and values to avoid repeatedly encoding the same Cadence type info unnecessarily.

- Compact communications.  Detachable Cadence type info allows CCF-based protocols to optionally avoid resending the same Cadence type info for all messages matching that type.  CCF-based protocols can cache and uniquely identify a Cadence type so it can be matched to Cadence value (such as an event) during decoding.

- Deterministic encoding.  CCF uses CBOR's Preferred Serialization to achieve deterministic encoding.  Other parts of CBOR's Core Deterministic Encoding Requirements are not needed by this specification.

- Early detection of malformed data.  CCF decoders can detect and reject malformed data without creating Cadence objects.  CCF decoders can detect malformed data without having Cadence type info.  If data is not malformed, then CCF decoders can proceed to detect and reject invalid CCF data as described in this document.

- Extensibility.  CCF encodes composite type information in a header (separate from data). Data refers to composite types by unique type ID, encoded as bytes. In the future, composite type information can be stored on-chain and header isn't necessary to be sent with data. Type ID can be an universal counter or hash value.

- Interoperability and Reuse.  CCF uses the same approach taken by COSE (RFC 9052) leveraging CBOR (RFC 8949).  CCF leverages CBOR, which allows CCF codecs to use CBOR codecs under the hood.

- Translation to JSON.  CCF uses a subset of CBOR data model and RFC 8949 specifies how to convert data between CBOR and JSON.

### Why CBOR

CBOR is an [Internet Standard](https://www.ietf.org/rfc/std-index.txt) defined by [IETF&nbsp;STD&nbsp;94 (RFC 8949)](https://www.rfc-editor.org/info/std94):

> The Concise Binary Object Representation (CBOR) is a data format
   whose design goals include the possibility of extremely small code
   size, fairly small message size, and extensibility without the need
   for version negotiation.  These design goals make it different from
   earlier binary serializations such as ASN.1 and MessagePack.

In addition to the aspects listed in CBOR's design goals, CBOR-based formats can support:
- Deterministic encodings across programming languages. Encoders implemented in different languages can produce identical deterministic encodings.
- Separate detection of malformed data and invalid data. Decoders can efficiently reject malformed inputs without creating Cadence objects.

CBOR is well-suited to replace JSON in data formats and protocols. CBOR's data model extends JSON's data model with:
- compact binary encodings
- extension points (CBOR Tags and Simple Values)
- deterministic encoding (Core Deterministic Encoding Requirements)

Published comparisons between CBOR and other binary data formats such as Protocol Buffers, etc. include:
- Appendix C of RFC 8618 Compacted-DNS: [Comparison of Binary Formats](https://www.rfc-editor.org/rfc/rfc8618#appendix-C)
- Appendix E of RFC 8949 (STD 94) CBOR: [Comparison of Other Binary Formats to CBOR's Design Objectives](https://www.rfc-editor.org/rfc/rfc8949.html#name-comparison-of-other-binary-)

CBOR is used in data formats and protocols such as [W3C&nbsp;WebAuthn](https://www.w3.org/TR/webauthn-2/), Compacted-DNS&nbsp;([IETF&nbsp;RFC&nbsp;8618](https://www.rfc-editor.org/rfc/rfc8618.html)), COSE&nbsp;([IETF&nbsp;STD&nbsp;96](https://www.rfc-editor.org/info/std96)), CWT&nbsp;([IETF&nbsp;RFC&nbsp;8392](https://www.rfc-editor.org/info/rfc8392)), etc.

Although using a 100% custom data format can sometimes produce smaller encodings than CBOR, that alone doesn't outweigh the combination of other qualities and considerations such as security, maintainability, risks, etc.

Lastly, Cadence is [already using CBOR](https://github.com/onflow/cadence/blob/master/runtime/interpreter/encode.go) to encode internal values, so using CBOR to encode external values added to the list of stronger reasons making CBOR a good fit.

Other considerations for using CBOR include availability and quality of CBOR codecs in various programming languages.

### Interoperability and Reuse of CBOR Codecs

CBOR data can be exchanged between standards compliant CBOR codecs implemented in any programming language.

CBOR-based formats and protocols can use existing CBOR codecs.  For example, [CCF codec](https://github.com/onflow/cadence/tree/master/encoding/ccf) in Cadence uses [fxamacker/cbor](https://github.com/fxamacker/cbor):
  - `fxamacker/cbor` was designed with security in mind and passed multiple security assessments in 2022.  A [nonconfidential security assessment](https://github.com/veraison/go-cose/blob/v1.0.0-rc.1/reports/NCC_Microsoft-go-cose-Report_2022-05-26_v1.0.pdf) produced by NCC Group for Microsoft Corporation includes parts of fxamacker/cbor.
  - `fxamacker/cbor` is used in projects by Arm Ltd., Cisco, Dapper Labs, EdgeX&nbsp;Foundry, Fraunhofer&#8209;AISEC, Linux&nbsp;Foundation, Microsoft, Mozilla, Tailscale, Teleport, [and&nbsp;others](https://github.com/fxamacker/cbor#who-uses-fxamackercbor).  Notably, it was already [used by Cadence](https://github.com/onflow/cadence/blob/master/runtime/interpreter/encode.go) for internal value encoding.
  - `fxamacker/cbor` is maintained by the author of this document.

CBOR codecs are available in various programming languages.  Examples include:

- __.NET Languages__: Microsoft maintains [System.Formats.Cbor namespace](https://learn.microsoft.com/en-us/dotnet/api/system.formats.cbor), which is the CBOR codec in .Net Platform Extensions.
- __C/C++__: Intel maintains [TinyCBOR](https://github.com/intel/tinycbor), a CBOR codec optimized for very fast operation with very small footprint.
- __Javascript__:  [hildjj/node-cbor](https://github.com/hildjj/node-cbor) and its successor [hildji/cbor2](https://github.com/hildjj/cbor2) are maintained by Joe Hildebrand (former VP of Engineering at Mozilla).

Projects implementing a CCF codec should evaluate more than one CBOR codec for standards compliance, security, and other factors.

### Terminology

This specification uses requirements terminology, CBOR terminology, and CDDL terminology.

This specification also uses the following notations:
- Concise Data Definition Language (CDDL) defined by [RFC 8610](https://www.rfc-editor.org/rfc/rfc8610.html).  CDDL is a notation for unambiguously expressing CBOR and JSON data structures.
- Extended Diagnostic Notation (EDN) defined by [Appendix G of RFC 8610](https://www.rfc-editor.org/rfc/rfc8610.html#appendix-G).  EDN is a "diagnostic notation" used for conversing about encoded CBOR data items.

#### Requirements Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [[RFC2119](https://www.rfc-editor.org/info/rfc2119)] [[RFC8174](https://www.rfc-editor.org/info/rfc8174)] when, and only when, they appear in all capitals, as shown here.

#### CBOR Terminology

This specification uses this subset of CBOR data items as defined in RFC 8949:
- nil
- bool
- positive integer
- negative integer
- byte string
- text string
- bignum
- array
- tagged data item (tag): comprised of a tag number and tag content

#### CDDL Terminology

This specification uses CDDL notation to express CBOR data items:
- `nil`: CBOR nil
- `bool`: CBOR bool
- `uint`: CBOR unsigned integer
- `nint`: CBOR negative integer
- `bstr`: CBOR byte string
- `tstr`: CBOR text string
- `bigint`: CBOR bignum
- `? Foo`: Foo is optional
- `(Foo)`: Foo is a group
- `Foo / Bar`: either Foo or Bar
- `Foo Bar`: Foo followed by Bar
- `[+ Foo]`: one or more Foo in CBOR array
- `[* Foo]`: zero or more Foo in CBOR array
- `#6.nnn(type)`: CBOR tag with tag number nnn and content type of "type".

## Serialization Considerations

CCF is a data format that uses a subset of CBOR with additional requirements for validity and deterministic encoding.

### Cadence Types and Values Encoding

[Cadence values have types](https://developers.flow.com/cadence/language/values-and-types).  For example:
- `true` has the type `Bool` 
- `"hello"` has the type `String` 
- `let aos: [String] = ["hello", "world"]` declares `aos` of type `[String]`. 
- `let aoa: [AnyStruct] = ["hello", true]` declares `aoa` of type `[AnyStruct]`.

CCF encoding decouples value encoding from type encoding.  For example:  
- Cadence booleans is encoded as a tuple of type and value: the `Bool` type and its value.  
- Cadence strings is encoded as a tuple of type and value: the `String` type and its value.

For compactness, encoders can omit encodings of Cadence type when:
- Optional value is nil, provided that the optional type information is encoded in the outer container.
- Element's type matches the outer container's element type.

For example, when encoding a Cadence container such as `Array`:
- `[String]`: each element's type matches the outer container's element type, so encoders can omit encoding each element's type.
- `[AnyStruct]`: each element's type cannot match the outer container's element type, so encoders must encode the type for each element.

### Valid CCF Encoding Requirements

A CCF encoding is valid if it complies with "Valid CCF Encoding Requirements".

Encoders MUST produce valid CCF encodings from valid input items.  Encoders are not required to check for invalid input items (e.g. invalid UTF-8 strings, duplicate dictionary keys, etc.)  Applications MUST NOT provide invalid items to encoders.

Decoders MUST reject CCF encodings that are not valid unless otherwise specified in this section.

A CCF encoding complies with "Valid CCF Encoding Requirements" if it complies with the following restrictions:

- CBOR data items MUST be well-formed and valid as defined in RFC 8949.  E.g. CBOR text strings MUST contain valid UTF-8.  As an exception, RFC 8949 requirements for CBOR maps are not applicable because CCF does not use CBOR maps.

- CCF encodings must comply with specifications in "CCF Specified in CDDL Notation" section of this document.

- `composite-type.id` MUST be unique in `ccf-typedef-message` or `ccf-typedef-and-value-message`.

- `composite-type.cadence-type-id` MUST be unique in `ccf-typedef-message` or `ccf-typedef-and-value-message`.

- `field-name` MUST be unique in `composite-type`.

- `type-ref.id` MUST refer to `composite-type.id`.

- `composite-type-value.id` MUST be unique in the same `composite-type-value` data item.

- `type-value-ref.id` MUST refer to `composite-type-value.id` in the same `composite-type-value` data item.

- `name` MUST be unique in `composite-type-value.fields`. 
  
- `name` MUST be unique in `function-value.type-parameters`. 

- All parameter lists MUST have unique `identifier`. For example, `indentifier` MUST be unique in
  - `composite-type-value.initializers`
  - `function-value.parameters`

- Elements MUST be unique in `restricted-type` or `restricted-type-value`.

- Keys MUST be unique in `dict-value`.  Decoders are not always required to check for duplicate dictionary keys.  In some cases, checking for duplicate dictionary key is not necessary or it may be delegated to the application.

### Deterministic CCF Encoding Requirements

A CCF encoding is deterministic if it satisfies the "Deterministic CCF Encoding Requirements".

Encoders SHOULD emit CCF encodings that are deterministic.  CCF-based protocols MUST specify when encoders are required to emit deterministic CCF encodings.  For example:
- a CCF-based protocol for encoding transaction arguments might want to specify that encoders MUST produce deterministic encodings of the values.
- a CCF-based protocol for encoding script results might want to specify that encoders are not required to produce deterministic encodings of the values (if results are sent to clients that don't care about the values being deterministically encoded). 

Decoders SHOULD check CCF encodings to determine whether they are deterministic encodings.  CCF-based protocols MUST specify when decoders are required to check for deterministic encodings and how to handle nondeterministic encodings.

A CCF encoding satisfies the "Deterministic CCF Encoding Requirements" if it satisfies the following restrictions:

- CCF encodings MUST satisfy "Valid CCF Encoding Requirements" defined in this document.

- CCF encodings MUST satisfy "Core Deterministic Encoding Requirements" defined in RFC 8948 Section 4.2.1.  As an exception, RFC 8949 requirements for CBOR maps are not applicable because CCF does not use CBOR maps.

- `composite-type.id` in `ccf-typedef-and-value-message` MUST be identical to its zero-based index in `composite-typedef`.

- `composite-type-value.id` MUST be identical to the zero-based encoding order `type-value`.

- `inline-type-and-value` MUST NOT be used when type can be omitted as described in "Cadence Types and Values Encoding".

- The following data items MUST be sorted using bytewise lexicographic order of their deterministic encodings:
  - Type definitions MUST be sorted by `cadence-type-id` in `composite-typedef`.
  - `dict-value` key-value pairs MUST be sorted by key.
  - `composite-type.fields` MUST be sorted by `name`
  - `composite-type-value.fields` MUST be sorted by `name`.
  - `restricted-type.restrictions` MUST be sorted by restriction's `cadence-type-id`.
  - `restricted-type-value.restrictions` MUST be sorted by restriction's `cadence-type-id`.

## Security Considerations

CBOR security considerations in [Section 10 of RFC 8949 (CBOR)](https://www.rfc-editor.org/rfc/rfc8949.html#name-security-considerations) apply to CCF.

There are two types of checks for acceptable data:
- Checks for well-formedness
- Checks for validity

CBOR defines data [well-formedness](https://www.rfc-editor.org/rfc/rfc8949.html#name-well-formedness-errors-and-) and a CBOR decoder MUST detect and reject malformed data before checking for validity.

CCF decoders MUST detect and reject malformed data before checking for validity.

CCF decoders SHOULD detect and reject malformed data before creating Cadence objects and without requiring Cadence type information.

CCF decoders can handle invalid CCF messages as required by each CCF-based protocol.  In some cases, it may be more practical for the application to check if the decoded data is acceptable.

CCF decoders SHOULD allow CBOR limits to be specified and enforced, such as:
- Max number of array elements
- Max number of map pairs
- Max nested levels

For example, max number of array elements would forbid any single array in a CCF message from exceeding that many elements.

The main trade-off for decoder limits:
- limits set too high can allow memory exhaustion and other attacks to succeed.
- limits set too low creates the possibility of being unable to decode non-malicious messages that exceeds limits.

Encoders usually don't enforce limits because it's simpler and more efficient for applications to enforce limits before providing the data to encoders.

## CCF Examples

Examples of JSON are in JSON-Cadence Data Interchange Format:
- Minified JSON encoding size is used for comparisons.
- Non-minified JSON encoding is shown for readability.

Examples of CCF encoded data are shown in hex or Extended Diagnostic Notation (EDN) for readability.

For more info about EDN, see [Appendix G of RFC 8610 (CDDL)](https://www.rfc-editor.org/rfc/rfc8610.html#appendix-G).

### Cadence Simple Type Value

Cadence `Int` type with value `42` encoded to JSON is 27 bytes when minified.

```json
{
  "type": "Int",
  "value": "42"
}
```

CCF encoding is 9 bytes:

```
d88282d88904c2412a
```

It represents `130([137(4), 42])`, where:
- `137(4)` is Cadence `Int`.
- `42` is raw value.

### Cadence Homogeneous Array with Simple Type Elements

Cadence `[Int]` type of value `[1, 2, 3]` encoded to JSON is 107 bytes when minified:

```json
{
  "type": "Array",
  "value": [
    {
      "type": "Int",
      "value": "1"
    },
    {
      "type": "Int",
      "value": "2"
    },
    {
      "type": "Int",
      "value": "3"
    }
  ]
}
```

CCF encoding is 18 bytes:

```
d88282d88bd8890483c24101c24102c24103
```

It represents `130([139(137(4)), [1, 2, 3]])`, where:
- `139(137(4))` is Cadence array of `Int`.
- `[1, 2, 3]` is raw value.

### Cadence Heterogeneous Array with Simple Type Elements

Cadence `[AnyStruct]` type of value `[1, "a", true]` encoded to JSON is 112 bytes when minified:

```json
{
  "type": "Array",
  "value": [
    {
      "type": "Int",
      "value": "1"
    },
    {
      "type": "String",
      "value": "a"
    },
    {
      "type": "Bool",
      "value": true
    }
  ]
}
```

CCF encoding is 34 bytes:

```
d88282d88bd889182783d88282d88904c24101d88282d889016161d88282d88900f5
```

It represents `130([139(137(39)), [130([137(4), 1]), 130([137(1), "a"]), 130([137(0), true])]])`, where:
- `139(137(39))` is Cadence array of `AnyStruct`.
- `130([137(4), 1])` is first element (`137(4)` is Cadence `Int`, `1` is raw value).
- `130([137(1), "a"])` is second element (`137(1)` is Cadence `String`, `"a"` is raw value).
- `130([137(0), true]` is third element (`137(0)` is Cadence `Boolean`, `true` is raw value).

### Cadence Homogeneous Array with Composite Type Elements

This example is for CCF in fully self-describing mode (partially self-describing mode encodes smaller messages).

Cadence composite types, such as struct, resource, and event, are not inlined as type info.  Each encoded composite type info has a unique ID which is used to bind with Cadence value.

Cadence `[Foo]` type encoded to JSON is 353 bytes when minified:

```json
{
  "type": "Array",
  "value": [
    {
      "type": "Resource",
      "value": {
        "id": "S.test.Foo",
        "fields": [
          {
            "name": "bar",
            "value": {
              "type": "Int",
              "value": "1"
            }
          }
        ]
      }
    },
    {
      "type": "Resource",
      "value": {
        "id": "S.test.Foo",
        "fields": [
          {
            "name": "bar",
            "value": {
              "type": "Int",
              "value": "2"
            }
          }
        ]
      }
    },
    {
      "type": "Resource",
      "value": {
        "id": "S.test.Foo",
        "fields": [
          {
            "name": "bar",
            "value": {
              "type": "Int",
              "value": "3"
            }
          }
        ]
      }
    }
  ]
}
```

CCF encoding is 47 bytes (in fully self-describing mode):

```
d8818281d8a183406a532e746573742e466f6f818263626172d8890482d88bd888408381c2410181c2410281c24103
```

It represents `129([[161([h'', "S.test.Foo", [["bar", 137(4)]]])], [139(136(h'')), [[1], [2], [3]]]])`, which contains type and value.

Type data item represents `161([h'', "S.test.Foo", [["bar", 137(4)]]])` , which defines a resource type:
- `h''` is CCF type ID `0`.  This ID is used inside value data item to bind type with raw value.
- `"S.test.Foo"` is Cadence type ID.
- `[["bar", 137(4)]]` is field definition of one field: `"bar"` field of Cadence `Int`.

Value data item represents `[139(136(h'')), [[1], [2], [3]]]]`, where:
- `139(136(h''))` is Cadence array of type identified by ID `0`.
- `[[1], [2], [3]]]` is array of `Foo` resource raw field data.

### Cadence Homogeneous Array with Composite Type Elements (One Field Type Is Abstract)

This example is for CCF in fully self-describing mode (partially self-describing mode encodes smaller messages).

Composite field `baz` is abstract type.

Cadence `[Foo]` type encoded to JSON is 508 bytes when minified:

```json
{
  "type": "Array",
  "value": [
    {
      "type": "Resource",
      "value": {
        "id": "S.test.Foo",
        "fields": [
          {
            "name": "bar",
            "value": {
              "type": "Int",
              "value": "1"
            }
          },
          {
            "name": "baz",
            "value": {
              "type": "Int",
              "value": "1"
            }
          }
        ]
      }
    },
    {
      "type": "Resource",
      "value": {
        "id": "S.test.Foo",
        "fields": [
          {
            "name": "bar",
            "value": {
              "type": "Int",
              "value": "2"
            }
          },
          {
            "name": "baz",
            "value": {
              "type": "String",
              "value": "a"
            }
          }
        ]
      }
    },
    {
      "type": "Resource",
      "value": {
        "id": "S.test.Foo",
        "fields": [
          {
            "name": "bar",
            "value": {
              "type": "Int",
              "value": "3"
            }
          },
          {
            "name": "baz",
            "value": {
              "type": "Bool",
              "value": true
            }
          }
        ]
      }
    }
  ]
}
```

CCF encoding is 80 bytes (in fully self-describing mode):

```
d8818281d8a183406a532e746573742e466f6f828263626172d88904826362617ad889182782d88bd888408382c24101d88282d88904c2410182c24102d88282d88901616182c24103d88282d88900f5
```

It represents `129([[161([h'', "S.test.Foo", [["bar", 137(4)], ["baz", 137(39)]]])], [139(136(h'')), [[1, 130([137(4), 1])], [2, 130([137(1), "a"])], [3, 130([137(0), true])]]]])`, which contains type and value.

Type data item represents `161([h'', "S.test.Foo", [["bar", 137(4)], ["baz", 137(39)]]])` , which defines a resource type:
- `h''` is CCF type ID `0`.  This ID is used inside value data item to bind type with raw value.
- `"S.test.Foo"` is Cadence type ID.
- `["bar", 137(4)]` is first field definition: `"bar"` field of Cadence `Int`.
- `["baz", 137(39)]` is second field definition: `"baz"` field of Cadence `AnyStruct`.

Value data item represents `[139(136(h'')), [[1, 130([137(4), 1])], [2, 130([137(1), "a"])], [3, 130([137(0), true])]]]`, where:
- `139(136(h''))` is Cadence array of type identified by ID `0`.
- `[1, 130([137(4), 1])]` is first resource raw field data.
- `[2, 130([137(1), "a"])]` is second resource raw field data.
- `[3, 130([137(0), true])]` is third resource raw field data.

### `FeesDeducted` Event

This example is for CCF in fully self-describing mode (partially self-describing mode encodes smaller messages).

This example of `FeesDeducted` event encoded to JSON is 298 bytes when minified:

```json
{
  "type": "Event",
  "value": {
    "id": "A.f919ee77447b7497.FlowFees.FeesDeducted",
    "fields": [
      {
        "name": "amount",
        "value": {
          "type": "UFix64",
          "value": "0.00002969"
        }
      },
      {
        "name": "inclusionEffort",
        "value": {
          "type": "UFix64",
          "value": "1.00000000"
        }
      },
      {
        "name": "executionEffort",
        "value": {
          "type": "UFix64",
          "value": "0.00000575"
        }
      }
    ]
  }
}
```

CCF encoding is 118 bytes (in fully self-describing mode):

```
d8818281d8a283407828412e663931396565373734343762373439372e466c6f77466565732e466565734465647563746564838266616d6f756e74d88917826f657865637574696f6e4566666f7274d88917826f696e636c7573696f6e4566666f7274d8891782d8884083190b9919023f1a05f5e100
```

It represents `129([[162([h'', "A.f919ee77447b7497.FlowFees.FeesDeducted", [["amount", 137(23)], ["executionEffort", 137(23)], ["inclusionEffort", 137(23)]]])], [136(h''), [2969, 575, 100000000]]])`, which contains type and value.

Type data item represents `162([h'', "A.f919ee77447b7497.FlowFees.FeesDeducted", [["amount", 137(23)], ["executionEffort", 137(23)], ["inclusionEffort", 137(23)]]])` , which defines an event type:
- `h''` is CCF type ID `0`.  This ID is used inside value data item to bind type with raw value.
- `"A.f919ee77447b7497.FlowFees.FeesDeducted"` is Cadence type ID.
- `[["amount", 137(23)], ["executionEffort", 137(23)], ["inclusionEffort", 137(23)]]` is field definition with 3 fields: `"amount"` field of Cadence `UFix64`, `"inclusionEffort"` field of Cadence `UFix64`, and `"executionEffort"` field of Cadence `UFix64`.  Fields are sorted by field name.

Value data item represents `[136(h''), [2969, 575, 100000000]]`, where:
- `136(h'')` is type identified by ID `0`.
- `[2969, 575, 100000000]` is `FeesDeducted` event raw field data.

## Specifications

There are 3 top level CCF messages: `ccf-typedef-message`, `ccf-typedef-and-value-message`, and `ccf-type-and-value-message`.

- `ccf-typedef-message` is a list of type definitions for Cadence composite types.
- `ccf-typedef-and-value-message` is a Cadence value preceded by a list of type definitions of referenced Cadence composite types.
- `ccf-type-and-value-message` is a Cadence value preceded by builtin or known composite Cadence type(s).

Cadence types are encoded as `inline-type` (inlined) or as `composite-typedef` (not inlined).

Cadence data is encoded depending on its type. For example, Cadence `UInt8` is encoded as CBOR positive integer, Cadence `String` is encoded as CBOR text string, Cadence `Address` is encoded as CBOR byte string, and Cadence struct data is encoded as an array of its raw field data.

### Cadence Types and Type Values

Cadence types and Cadence type values (run-time types) are encoded differently.  They contain different data because they are used differently.

Cadence types are used to decode Cadence data, so they only contain information needed for decoding.  For example, composite type's field info is needed to decode composite value.  However, interface type's field info isn't needed to decode values implementing interface type.

Cadence type value is a Cadence value which provides comprehensive information about a type.  For example, composite type value and interface type value contain info about both fields and initializers.

### CCF Specified in CDDL Notation

```cddl
;CDDL-BEGIN

; CCF uses CBOR tag numbers 128-255, which are unassigned by IANA.
; https://www.iana.org/assignments/cbor-tags/cbor-tags.xhtml

; NOTE: when changing values, also update uses in rules!

; CBOR tag numbers (128-135) for root objects
cbor-tag-typedef = 128
cbor-tag-typedef-and-value = 129
cbor-tag-type-and-value = 130
; 131-135 are reserved

; CBOR tag numbers (136-183) for types

; inline types
cbor-tag-type-ref = 136
cbor-tag-simple-type = 137
cbor-tag-optional-type = 138
cbor-tag-varsized-array-type = 139
cbor-tag-constsized-array-type = 140
cbor-tag-dict-type = 141
cbor-tag-reference-type = 142
cbor-tag-restricted-type = 143
cbor-tag-capability-type = 144
; 145-159 are reserved

; composite types
cbor-tag-struct-type = 160
cbor-tag-resource-type = 161
cbor-tag-event-type = 162
cbor-tag-contract-type = 163
cbor-tag-enum-type = 164
; 165-175 are reserved

; interface types
cbor-tag-struct-interface-type = 176
cbor-tag-resource-interface-type = 177
cbor-tag-contract-interface-type = 178
; 179-183 are reserved

; CBOR tag numbers (184-231) for type value

; non-composite and non-interface type values
cbor-tag-type-value-ref = 184
cbor-tag-simple-type-value = 185
cbor-tag-optional-type-value = 186
cbor-tag-varsized-array-type-value = 187
cbor-tag-constsized-array-type-value = 188
cbor-tag-dict-type-value = 189
cbor-tag-reference-type-value = 190
cbor-tag-restricted-type-value = 191
cbor-tag-capability-type-value = 192
cbor-tag-function-type-value = 193
; 194-207 are reserved

; composite type values
cbor-tag-struct-type-value = 208
cbor-tag-resource-type-value = 209
cbor-tag-event-type-value = 210
cbor-tag-contract-type-value = 211
cbor-tag-enum-type-value = 212
; 213-223 are reserved

; interface type values
cbor-tag-struct-interface-type-value = 224
cbor-tag-resource-interface-type-value = 225
cbor-tag-contract-interface-type-value = 226
; 227-231 are reserved

ccf-message = (
    ccf-typedef-message
    / ccf-typedef-and-value-message
    / ccf-type-and-value-message
)

ccf-typedef-message =
    ; cbor-tag-typedef
    #6.128(composite-typedef)

composite-typedef = [
    ; one-or-more instead of zero-or-more because:
    ; - when encoding a primitive type, such as boolean or string, `ccf-type-and-value-message` is used (no `composite-typedef` at all)
    ; - when encoding a composite type, such as event, `ccf-typedef-and-value-message` is used, which encodes at least one `composite-typedef`
    + (
        struct-type
        / resource-type
        / contract-type
        / event-type
        / enum-type
        / struct-interface-type
        / resource-interface-type
        / contract-interface-type
    )
]

; id is a unique identifier used by CCF to associate composite/interface type information
; with value through type-ref and type-value-ref.
id = bstr

; cadence-type-id is an identifier used by Cadence to identify types.
cadence-type-id = tstr

composite-type = [
    id: id,
    cadence-type-id: cadence-type-id,
    fields: [
        * [
            field-name: tstr,
            field-type: inline-type
        ]
    ]
]

; interface-type doesn't include field info because it's not needed for
; decoding values implementing interface type.
interface-type = [
    id: id,
    cadence-type-id: tstr,
]

struct-type =
    ; cbor-tag-struct-type
    #6.160(composite-type)

resource-type =
    ; cbor-tag-resource-type
    #6.161(composite-type)

event-type =
    ; cbor-tag-event-type
    #6.162(composite-type)

contract-type =
    ; cbor-tag-contract-type
    #6.163(composite-type)

enum-type =
    ; cbor-tag-enum-type
    #6.164(composite-type)

struct-interface-type =
    ; cbor-tag-struct-interface-type
    #6.176(interface-type)

resource-interface-type =
    ; cbor-tag-resource-interface-type
    #6.177(interface-type)

contract-interface-type =
    ; cbor-tag-contract-interface-type
    #6.178(interface-type)

inline-type =
    simple-type
    / optional-type
    / varsized-array-type
    / constsized-array-type
    / dict-type
    / reference-type
    / restricted-type
    / capability-type
    / type-ref

simple-type =
    ; cbor-tag-simple-type
    #6.137(simple-type-id)

optional-type =
    ; cbor-tag-optional-type
    #6.138(inline-type)

varsized-array-type =
    ; cbor-tag-varsized-array-type
    #6.139(inline-type)

constsized-array-type =
    ; cbor-tag-constsized-array-type
    #6.140([
        array-size: uint,
        element-type: inline-type
    ])

dict-type =
    ; cbor-tag-dict-type
    #6.141([
        key-type: inline-type,
        element-type: inline-type
    ])

reference-type =
    ; cbor-tag-reference-type
    #6.142([
      authorized: bool,
      type: inline-type,
    ])

restricted-type =
    ; cbor-tag-restricted-type
    #6.143([
      type: inline-type / nil,
      restrictions: [* inline-type]
    ])

capability-type =
    ; cbor-tag-capability-type
    ; use an array as an extension point
    #6.144([
        ; borrow-type
        inline-type / nil
    ])

type-ref =
    ; cbor-tag-type-ref
    #6.136(id)

; simple-type-id is an enumeration.
simple-type-id = &(
    bool-type-id: 0,
    string-type-id: 1,
    character-type-id: 2,
    address-type-id: 3,
    int-type-id: 4,
    int8-type-id: 5,
    int16-type-id: 6,
    int32-type-id: 7,
    int64-type-id: 8,
    int128-type-id: 9,
    int256-type-id: 10,
    uint-type-id: 11,
    uint8-type-id: 12,
    uint16-type-id: 13,
    uint32-type-id: 14,
    uint64-type-id: 15,
    uint128-type-id: 16,
    uint256-type-id: 17,
    word8-type-id: 18,
    word16-type-id: 19,
    word32-type-id: 20,
    word64-type-id: 21,
    fix64-type-id: 22,
    ufix64-type-id: 23,
    path-type-id: 24,
    capability-path-type-id: 25,
    storage-path-type-id: 26,
    public-path-type-id: 27,
    private-path-type-id: 28,
    auth-account-type-id: 29,
    public-account-type-id: 30,
    auth-account-keys-type-id: 31,
    public-account-keys-type-id: 32,
    auth-account-contracts-type-id: 33,
    public-account-contracts-type-id: 34,
    deployed-contract-type-id: 35,
    account-key-type-id: 36,
    block-type-id: 37,
    any-type-id: 38,
    any-struct-type-id: 39,
    any-resource-type-id: 40,
    meta-type-type-id: 41,
    never-type-id: 42,
    number-type-id: 43,
    signed-number-type-id: 44,
    integer-type-id: 45,
    signed-integer-type-id: 46,
    fixed-point-type-id: 47,
    signed-fixed-point-type-id: 48,
    bytes-type-id: 49,
    void-type-id: 50,
    function-type-id: 51,
    word128-type-id: 52,
    word256-type-id: 53,
    any-struct-attachment-type-id: 54,
    any-resource-attachment-type-id: 55,
)

ccf-typedef-and-value-message =
    ; cbor-tag-typedef-and-value
    #6.129([
      typedef: composite-typedef,
      type-and-value: inline-type-and-value 
    ])
    
ccf-type-and-value-message =
    ; cbor-tag-type-and-value
    #6.130(inline-type-and-value)

inline-type-and-value = [
      type: inline-type,
      value: value,
]
    
value =
    ccf-type-and-value-message
    / simple-value
    / optional-value
    / array-value
    / dict-value
    / composite-value
    / path-value
    / capability-value
    / function-value
    / type-value

optional-value = nil / value

array-value = [* value]

dict-value = [* (key: value, value: value)]

; composite-value is used to encode struct, contract, enum, event, and resource.
composite-value = [* (field: value)]

path-value = [
    domain: uint,
    identifier: tstr,
]

capability-value = 
    path-capability-value
    / id-capability-value

path-capability-value = [
    address: address-value,
    path: path-value
]

id-capability-value = [
    address: address-value,
    id: uint64-value
]

simple-value =
    void-value
    / bool-value
    / character-value
    / string-value
    / address-value
    / uint-value
    / uint8-value
    / uint16-value
    / uint32-value
    / uint64-value
    / uint128-value
    / uint256-value
    / int-value
    / int8-value
    / int16-value
    / int32-value
    / int64-value
    / int128-value
    / int256-value
    / word8-value
    / word16-value
    / word32-value
    / word64-value
    / word128-value
    / word256-value
    / fix64-value
    / ufix64-value

void-value = nil
bool-value = bool
character-value = tstr
string-value = tstr
address-value = bstr .size 8
uint-value = bigint .ge 0
uint8-value = uint .le 255
uint16-value = uint .le 65535
uint32-value = uint .le 4294967295
uint64-value = uint .le 18446744073709551615
uint128-value = bigint .ge 0
uint256-value = bigint .ge 0
int-value = bigint
int8-value = (int .ge -128) .le 127
int16-value = (int .ge -32768) .le 32767
int32-value = (int .ge -2147483648) .le 2147483647
int64-value = (int .ge -9223372036854775808) .le 9223372036854775807
int128-value = bigint
int256-value = bigint
word8-value = uint .le 255
word16-value = uint .le 65535
word32-value = uint .le 4294967295
word64-value = uint .le 18446744073709551615
word128-value = bigint .ge 0
word256-value = bigint .ge 0
fix64-value = (int .ge -9223372036854775808) .le 9223372036854775807
ufix64-value = uint .le 18446744073709551615

function-value = [
    type-parameters: [
        * [
           name: tstr,
           type-bound: type-value / nil
        ]
    ]
    parameters: [
        * [
            label: tstr,
            identifier: tstr,
            type: type-value
        ]
    ]
    return-type: type-value
]

type-value = simple-type-value
    / optional-type-value
    / varsized-array-type-value
    / constsized-array-type-value
    / dict-type-value
    / struct-type-value
    / resource-type-value
    / contract-type-value
    / event-type-value
    / enum-type-value
    / struct-interface-type-value
    / resource-interface-type-value
    / contract-interface-type-value
    / function-type-value
    / reference-type-value
    / restricted-type-value
    / capability-type-value
    / type-value-ref

simple-type-value =
    ; cbor-tag-simple-type-value
    #6.185(simple-type-id)

optional-type-value =
    ; cbor-tag-optional-type-value
    #6.186(type-value)

varsized-array-type-value =
    ; cbor-tag-varsized-array-type-value
    #6.187(type-value)

constsized-array-type-value =
    ; cbor-tag-constsized-array-type-value
    #6.188([
        array-size: uint,
        element-type: type-value
    ])

dict-type-value =
    ; cbor-tag-dict-type-value
    #6.189([
        key-type: type-value,
        element-type: type-value
    ])

struct-type-value =
    ; cbor-tag-struct-type-value
    #6.208(composite-type-value)

resource-type-value =
    ; cbor-tag-resource-type-value
    #6.209(composite-type-value)

event-type-value =
    ; cbor-tag-event-type-value
    #6.210(composite-type-value)

contract-type-value =
    ; cbor-tag-contract-type-value
    #6.211(composite-type-value)

enum-type-value =
    ; cbor-tag-enum-type-value
    #6.212(composite-type-value)

struct-interface-type-value =
    ; cbor-tag-struct-interface-type-value
    #6.224(composite-type-value)

resource-interface-type-value =
    ; cbor-tag-resource-interface-type-value
    #6.225(composite-type-value)

contract-interface-type-value =
    ; cbor-tag-contract-interface-type-value
    #6.226(composite-type-value)

composite-type-value = [
    id: id,
    cadence-type-id: cadence-type-id,
    ; type is only used by enum type value
    type: nil / type-value,
    fields: [
        * [
            name: tstr,
            type: type-value
        ]
    ]
    initializers: [
        ? [
            * [
                label: tstr,
                identifier: tstr,
                type: type-value
            ]
        ]
    ]
]

function-type-value =
    ; cbor-tag-function-type-value
    #6.193(function-value)

reference-type-value =
    ; cbor-tag-reference-type-value
    #6.190([
      authorized: bool,
      type: type-value,
    ])

restricted-type-value =
    ; cbor-tag-restricted-type-value
    #6.191([
      type: type-value / nil,
      restrictions: [* type-value]
    ])

capability-type-value =
    ; cbor-tag-capability-type-value
    ; use an array as an extension point
    #6.192([
      ; borrow-type
      type-value / nil
    ])

type-value-ref =
    ; cbor-tag-type-value-ref
    #6.184(id)

;CDDL-END
```

## Acknowledgments

This document would not exist without Ramtin M. Seraj and Bastian Müller.

Ramtin and Bastian's contributions on this effort is hard to list exhaustively because they inspire individuals and teams to produce impactful results.

Ramtin M. Seraj led the effort to require a deterministic and more compact alternative to JSON-Cadence Data Interchange Format.  This document's "Objectives" section includes and adds to the initial objectives Ramtin listed (in a notion) for a binary format for Cadence external values.  

Bastian Müller helped the author understand details related to Cadence types and values, which made writing this document possible.  Bastian also led the PR reviews of this document, contributed changes, asked great questions during CCF meetings, and identified sections that needed clarification (in both the CDDL notation and English text).
