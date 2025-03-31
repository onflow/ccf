# Cadence Compact Format (CCF)

Author: Faye Amacker  
Version: 1.0.0  
Date: March 31, 2025

## Abstract

Cadence Compact Format (CCF) is a binary data format designed for compact, efficient, and deterministic encoding of [Cadence](https://github.com/onflow/cadence) external values. Cadence is a modern resource-oriented programming language used by [Flow](https://github.com/onflow/flow-go) blockchain.

CCF messages can be fully self-describing or partially self-describing. Both are more compact than JSON-based messages. CCF-based protocols can send Cadence metadata once, and CCF messages can reuse it. Malformed data can be detected without Cadence metadata and without creating Cadence objects.

CCF specifies "Deterministic CCF Encoding Requirements" and makes it optional. CCF codecs implemented in different programming languages can produce the same deterministic encodings. CCF-based protocols can balance trade-offs by specifying how they use optional CCF requirements.

CCF obsoletes [JSON-Cadence Data Interchange Format](https://developers.flow.com/cadence/json-cadence-spec) (JSON-CDC) for use cases that do not require JSON.

## Copyright Notice

Copyright © 2022-2025 Flow Foundation and the persons identified as the document authors.

This document is licensed under the terms of the Apache License, Version 2.0. See [LICENSE](LICENSE) for more information.

## Scope

This document specifies Cadence Compact Format.

This document explicitly specifies some requirements as optional. 

It is outside the scope of this document to specify individual CCF-based protocols (e.g., events). For example, CCF-based protocols MUST specify when encoders are required to emit CCF encodings that satisfy "Deterministic CCF Encoding Requirements."

This document does not specify how to encode version numbers of CCF itself or CCF-based protocols. CCF-based protocols can specify an encoding that uses CalVer, SemVer, sequence-based versioning, any other versioning, or no versioning. Some CCF-based protocols may want to use CBOR Sequences ([RFC 8742](https://www.rfc-editor.org/rfc/rfc8742.html)) to provide a version number in the first CBOR data item, followed by CBOR data item(s) representing CCF message(s).

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

### Objectives

The goal of CCF is to provide compact, efficient, and deterministic encoding of Cadence external values with:
- Fully self-describing mode: CCF messages include Cadence types and values.
- Partially self-describing mode: CCF messages include Cadence values referencing omitted Cadence types by unique type ID.

CCF supports:

- Cadence external values: CCF supports all Cadence built-in types and user-defined types (e.g., composite types). For extensibility, CCF reserves multiple ranges of CBOR tag numbers (unassigned by IANA) for future Cadence built-in data types.

- Compact encoding:
  - CCF uses a subset of the CBOR data model with [Preferred Serialization](https://www.rfc-editor.org/rfc/rfc8949.html#name-preferred-serialization) to encode values to their shortest form.
  - CCF separately encodes Cadence types and values to avoid repeatedly encoding the same Cadence types when feasible. For example, the element type of a homogeneous array is only encoded once.

- Compact communications: Detachable Cadence types allow CCF-based protocols to optionally avoid resending the same Cadence types for all messages. CCF-based protocols can cache and uniquely identify a Cadence type so it can be matched to a Cadence value during decoding.

  For example, CCF encodes Cadence composite types separately from Cadence values. Encoded values refer to their composite type by unique type ID, encoded as bytes. If the Cadence composite type (metadata) can be stored on-chain, it doesn't need to be sent with the value. Type ID can be a universal counter, hash digest, or other unique identifier specified by CCF-based protocols.

- Deterministic encoding: CCF uses CBOR's Preferred Serialization to achieve deterministic encoding. Other parts of CBOR's Core Deterministic Encoding Requirements are not needed by this specification.

- Early detection of malformed data: CCF decoders can detect malformed data without having Cadence type information. CCF decoders can detect and reject malformed data without creating Cadence objects. If data is well-formed, CCF decoders can proceed to detect and reject invalid CCF data as described in this document.

- Interoperability and reuse: CCF uses CBOR, so CCF codecs can use generic CBOR codecs that are well-tested and widely used by other projects.

- Converting Data Between CCF and JSON: CCF uses a subset of the CBOR data model. So the guidance in RFC 8949 on [converting data between CBOR and JSON](https://www.rfc-editor.org/rfc/rfc8949.html#name-converting-data-between-cbo) is applicable to CCF.


### Why CBOR

CBOR is a binary data format specified by [RFC 8949](https://www.rfc-editor.org/info/std94) and designated by IETF as an [Internet Standard](https://www.ietf.org/rfc/std-index.txt) (STD&nbsp;94).

Design goals of CBOR balances trade-offs, making it useful as a building block for new data formats and protocols:

> The Concise Binary Object Representation (CBOR) is a data format
   whose design goals include the possibility of extremely small code
   size, fairly small message size, and extensibility without the need
   for version negotiation. These design goals make it different from
   earlier binary serializations such as ASN.1 and MessagePack.

CBOR is used by data formats and protocols such as [W3C&nbsp;WebAuthn](https://www.w3.org/TR/webauthn-2/), Compacted-DNS&nbsp;([RFC&nbsp;8618](https://www.rfc-editor.org/rfc/rfc8618.html)), COSE&nbsp;([IETF&nbsp;STD&nbsp;96](https://www.rfc-editor.org/info/std96)), CWT&nbsp;([RFC&nbsp;8392](https://www.rfc-editor.org/info/rfc8392)), etc.

Notably, CBOR-based data formats like CCF can be specified to support:
- Deterministic encodings across programming languages. Encoders implemented in different languages can produce identical deterministic encodings.
- Separate detection of malformed data and invalid data. Decoders can efficiently reject malformed inputs without creating Cadence objects.

CBOR is well-suited to replace JSON in data formats and protocols. CBOR's data model extends JSON's data model with:
- compact binary encodings
- extension points (CBOR Tags and Simple Values)
- deterministic encoding (Core Deterministic Encoding Requirements)

Published comparisons between CBOR and other binary data formats, such as Protocol Buffers, etc., include:
- Appendix C of RFC 8618 Compacted-DNS: [Comparison of Binary Formats](https://www.rfc-editor.org/rfc/rfc8618#appendix-C)
- Appendix E of RFC 8949 (STD 94) CBOR: [Comparison of Other Binary Formats to CBOR's Design Objectives](https://www.rfc-editor.org/rfc/rfc8949.html#name-comparison-of-other-binary-)

Although using a 100% custom data format can sometimes produce smaller encodings than CBOR, that alone doesn't outweigh the combination of other qualities and considerations such as maturity of specification, security, maintainability, risks, etc.

Other considerations for using CBOR include the availability and quality of CBOR codecs in various programming languages.

### Interoperability and Reuse of CBOR Codecs

CBOR data can be exchanged between standards-compliant CBOR codecs implemented in any programming language.

Projects implementing a CCF codec should evaluate more than one CBOR codec for standards compliance, security, and other factors.

When evaluating or comparing codecs, benchmarks should include decoding malicious data.

### Notations and Terminology

This document uses CDDL and EDN notations:
- Concise Data Definition Language (CDDL) is defined by [RFC 8610](https://www.rfc-editor.org/rfc/rfc8610.html). CDDL is a notation for unambiguously expressing CBOR and JSON data structures.
- Extended Diagnostic Notation (EDN) is defined by [Appendix G of RFC 8610](https://www.rfc-editor.org/rfc/rfc8610.html#appendix-G). EDN is a "diagnostic notation" used to converse about encoded CBOR data items.

This specification uses requirements terminology, CBOR terminology, and CDDL terminology.

#### Requirements Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [BCP 14](https://www.rfc-editor.org/info/bcp14) [[RFC2119](https://www.rfc-editor.org/info/rfc2119)] [[RFC8174](https://www.rfc-editor.org/info/rfc8174)] when, and only when, they appear in all capitals, as shown here.

#### CBOR Terminology

This specification uses this subset of CBOR data items as defined in RFC 8949:
- nil
- bool
- unsigned integer
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
- `[+ Foo]`: one or more Foo in a CBOR array
- `[* Foo]`: zero or more Foo in a CBOR array
- `#6.nnn(type)`: CBOR tag with tag number nnn and content type of "type"

## Serialization Considerations

CCF is a binary data format that uses a subset of CBOR with additional requirements for validity and deterministic encoding.

### Cadence Types and Values Encoding

[Cadence values have types](https://developers.flow.com/cadence/language/values-and-types). For example:
- `true` has the type `Bool` 
- `"hello"` has the type `String` 
- `let aos: [String] = ["hello", "world"]` declares `aos` of type `[String]`. 
- `let aoa: [AnyStruct] = ["hello", true]` declares `aoa` of type `[AnyStruct]`.

CCF encoding decouples value encoding from type encoding. For example:
- Cadence boolean is encoded as a tuple of type and value: the `Bool` type and its value.
- Cadence string is encoded as a tuple of type and value: the `String` type and its value.

For compactness, encoders can omit encodings of Cadence type when:
- Optional value is nil, provided that the optional type information is encoded in the outer container.
- Element's type matches the outer container's element type.

For example, when encoding a Cadence container such as `Array`:
- `[String]`: each element's type matches the outer container's element type, so encoders can omit each element's type.
- `[AnyStruct]`: each element's type cannot match the outer container's element type, so encoders must encode the type for each element.

### Valid CCF Encoding Requirements

A CCF encoding is valid if it complies with "Valid CCF Encoding Requirements."

Encoders MUST produce valid CCF encodings from valid input items. Encoders are not required to check for invalid input items (e.g., invalid UTF-8 strings, duplicate dictionary keys, etc.) Applications MUST NOT provide invalid items to encoders.

Decoders MUST reject CCF encodings that are not valid unless otherwise specified in this section.

A CCF encoding complies with "Valid CCF Encoding Requirements" if it complies with the following restrictions:

- CBOR data items MUST be well-formed and valid as defined in RFC 8949. For example, CBOR text strings MUST contain valid UTF-8. As an exception, RFC 8949 requirements for CBOR maps are not applicable because CCF does not use CBOR maps.

- CCF encodings MUST comply with specifications in the "CCF Specified in CDDL Notation" section of this document.

- `composite-type.id` MUST be unique in `ccf-typedef-message` or `ccf-typedef-and-value-message`.

- `composite-type.cadence-type-id` MUST be unique in `ccf-typedef-message` or `ccf-typedef-and-value-message`.

- `field-name` MUST be unique in `composite-type.fields`.

- `type-ref.id` MUST refer to `composite-type.id`.

- `composite-type-value.id` MUST be unique in the same `composite-type-value` data item.

- `type-value-ref.id` MUST refer to `composite-type-value.id` in the same `composite-type-value` data item.

- `name` MUST be unique in `composite-type-value.fields`. 

- `name` MUST be unique in `function-value.type-parameters`. 

- All parameter lists MUST have a unique `identifier`. For example, `identifier` MUST be unique in
  - `composite-type-value.initializers`
  - `function-value.parameters`

- Elements MUST be unique in `intersection-type` or `intersection-type-value`.

- Elements MUST be unique in `entitlement-set-authorization-type.entitlements` or `entitlement-set-authorization-type-value.entitlements`.
 
- Keys MUST be unique in `dict-value`. Decoders are not always required to check for duplicate dictionary keys. In some cases, checking for duplicate dictionary keys is not necessary, or the checking may be delegated to the application.

### Deterministic CCF Encoding Requirements

A CCF encoding is deterministic if it satisfies the "Deterministic CCF Encoding Requirements."

Encoders SHOULD emit deterministic CCF encodings. However, some CCF-based protocols may not require deterministic CCF encodings.

CCF-based protocols MUST specify when encoders are required to emit deterministic CCF encodings. For example:
- A CCF-based protocol that prioritizes security above performance (or requires explicitly sorted fields) might want to specify that encoders MUST produce deterministic encodings of the values.
- A CCF-based protocol for encoding unsorted fields might want to specify that encoders are not required to produce deterministic encodings of the values (if compatibility with legacy systems is a higher priority than "Deterministic CCF Encoding Requirements"). 

Decoders SHOULD check CCF encodings to determine whether they are deterministic encodings. CCF-based protocols MUST specify when decoders are required to check for deterministic encodings and how to handle nondeterministic encodings.

A CCF encoding satisfies the "Deterministic CCF Encoding Requirements" if it satisfies the following restrictions:

- CCF encodings MUST satisfy "Valid CCF Encoding Requirements" defined in this document.

- CCF encodings MUST satisfy the Core Deterministic Encoding Requirements defined in [Section 4.2.1](https://www.rfc-editor.org/rfc/rfc8949.html#name-core-deterministic-encoding) of RFC 8949. As an exception, RFC 8949 requirements for CBOR maps are not applicable because CCF does not use CBOR maps.

- `composite-type.id` in `ccf-typedef-and-value-message` MUST be identical to its zero-based index in `composite-typedef`.

- `composite-type-value.id` MUST be identical to the zero-based encoding order `type-value`.

- `inline-type-and-value` MUST NOT be used when type can be omitted as described in "Cadence Types and Values Encoding."

- The following data items MUST be sorted using bytewise lexicographic order of their deterministic encodings:
  - Type definitions MUST be sorted by `cadence-type-id` in `composite-typedef`.
  - `dict-value` key-value pairs MUST be sorted by key.
  - `composite-type.fields` MUST be sorted by `name`
  - `composite-type-value.fields` MUST be sorted by `name`.
  - `intersection-type.types` MUST be sorted by restriction's `cadence-type-id`.
  - `intersection-type-value.types` MUST be sorted by restriction's `cadence-type-id`.
  - `entitlement-set-authorization-type.entitlements` MUST be sorted.
  - `entitlement-set-authorization-type-value.entitlements` MUST be sorted.

## Security Considerations

CBOR security considerations in [Section 10](https://www.rfc-editor.org/rfc/rfc8949.html#name-security-considerations) of RFC 8949 apply to CCF.

There are two types of checks for acceptable data: well-formedness and validity.

CCF decoders MUST detect and reject malformed data items before checking for validity. [Section 1.2](https://www.rfc-editor.org/rfc/rfc8949.html#section-1.2) of RFC 8949 defines "well-formed" data items.

CCF decoders SHOULD detect and reject malformed data before creating Cadence objects and without requiring Cadence type information.

Each CCF-based protocol MUST specify how to handle invalid CCF messages. In some cases, it may be more practical for the application to check if the decoded data is acceptable.

CCF decoders SHOULD allow CBOR limits to be specified and enforced, such as:
- maximum number of array elements
- maximum number of map pairs
- maximum nested levels

For example, the maximum number of array elements would forbid any single array in a CCF message from exceeding that many elements.

The main trade-off for decoder limits:
- Limits set too high can allow memory exhaustion and other attacks to succeed.
- Limits set too low create the possibility of being unable to decode non-malicious messages that exceed limits.

Encoders usually don't enforce limits because it's simpler and more efficient for applications to enforce limits before providing the data to encoders.

## CCF Examples

Examples show Cadence data encoded in:
- [JSON-Cadence Data Interchange Format](https://developers.flow.com/cadence/json-cadence-spec) (JSON-CDC)
- CCF in fully self-describing mode

CCF in partially self-describing mode is not shown here. CCF in partially self-describing mode can omit type definitions, so it can be significantly more compact than CCF in self-describing mode.

CCF encodings are shown in hex and described using Extended Diagnostic Notation (EDN). For more information about EDN, see [Appendix G](https://www.rfc-editor.org/rfc/rfc8610.html#appendix-G) of RFC 8610.

### Cadence Simple Type Value

Cadence `Int` type with value `42` encodes to:
- 27 bytes in JSON-CDC (minified)
- 9 bytes in CCF (fully self-describing mode)

JSON-CDC encoding (not minified for readability):

```json
{
  "type": "Int",
  "value": "42"
}
```

CCF encoding (in hex for readability):

`d88282d88904c2412a`

The CCF data item in diagnostic notation is `130([137(4), 42])`:
- `137(4)` is the Cadence `Int`.
- `42` is the raw value.

A type definition is not needed because Cadence `Int` is a Simple Type (e.g., not a user-defined composite).

### Cadence Homogeneous Array with Simple Type Elements

Cadence `[Int]` type of value `[1, 2, 3]` encodes to:
- 107 bytes in JSON-CDC (minified)
- 18 bytes in CCF (fully self-describing mode)

JSON-CDC encoding (not minified for readability):

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

CCF encoding (in hex for readability):

`d88282d88bd8890483c24101c24102c24103`

The CCF data item in diagnostic notation is `130([139(137(4)), [1, 2, 3]])`, where:
- `139(137(4))` is the Cadence array of `Int`.
- `[1, 2, 3]` is the raw value.

### Cadence Heterogeneous Array with Simple Type Elements

Cadence `[AnyStruct]` type of value `[1, "a", true]` encodes to:
- 112 bytes in JSON-CDC (minified)
- 34 bytes in CCF (fully self-describing mode)

JSON-CDC encoding (not minified for readability)

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

CCF encoding (in hex for readability):

`d88282d88bd889182783d88282d88904c24101d88282d889016161d88282d88900f5`

The CCF data item in diagnostic notation is `130([139(137(39)), [130([137(4), 1]), 130([137(1), "a"]), 130([137(0), true])]])`, where:
- `139(137(39))` is the Cadence array of `AnyStruct`.
- `130([137(4), 1])` is the first element. `137(4)` is the Cadence `Int`, `1` is the raw value.
- `130([137(1), "a"])` is the second element. `137(1)` is the Cadence `String`, `"a"` is the raw value.
- `130([137(0), true]` is the third element. `137(0)` is the Cadence `Boolean`, `true` is the raw value.

### Cadence Homogeneous Array with Composite Type Elements

Cadence composite types, such as struct, resource, and event, are not inlined as type information. Each encoded composite type information has a unique ID that is used to bind with a Cadence value.

This example uses a Cadence homogeneous array `[Foo]`. The elements have the same Cadence composite type `Foo`, which has a field named `bar` of type Cadence `Int`.

Cadence `[Foo]` type encodes to:
- 353 bytes in JSON-CDC (minified)
- 47 bytes in CCF (fully self-describing mode)

JSON-CDC encoding (not minified for readability):

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

CCF encoding (in hex for readability):

`d8818281d8a183406a532e746573742e466f6f818263626172d8890482d88bd888408381c2410181c2410281c24103`

In fully self-describing mode, a CCF data item can contain the value and type definition(s) used by the value.

The CCF data item in diagnostic notation is:

`129([[161([h'', "S.test.Foo", [["bar", 137(4)]]])], [139(136(h'')), [[1], [2], [3]]]])`

Within the CCF data item, the type is `161([h'', "S.test.Foo", [["bar", 137(4)]]])` which defines a resource type:
- `h''` is CCF type ID `0`. This ID is used inside the value data item to bind the type with the raw value.
- `"S.test.Foo"` is the Cadence type ID.
- `[["bar", 137(4)]]` is the field definition of the `"bar"` field of Cadence `Int`.

Within the CCF data item, the value is `[139(136(h'')), [[1], [2], [3]]]]`, where:
- `139(136(h''))` is the Cadence array of the type identified by ID `0`.
- `[[1], [2], [3]]]` is the array of raw field data of the `Foo` resource.

### Cadence Homogeneous Array with Composite Type Elements (One Field Type Is Abstract)

This example uses a Cadence homogeneous array `[Foo]`. The elements have the same Cadence composite type. The `baz` field of the composite is an abstract type.

This example encodes to:
- 508 bytes in JSON-CDC (minified)
-  80 bytes in CCF (fully self-describing mode)

JSON-CDC encoding (not minified for readability):

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

CCF encoding (in fully self-describing mode):

`d8818281d8a183406a532e746573742e466f6f828263626172d88904826362617ad889182782d88bd888408382c24101d88282d88904c2410182c24102d88282d88901616182c24103d88282d88900f5`

In fully self-describing mode, a CCF data item can contain the value and type definition(s) used by the value.

The CCF data item in diagnostic notation is:

`129([[161([h'', "S.test.Foo", [["bar", 137(4)], ["baz", 137(39)]]])], [139(136(h'')), [[1, 130([137(4), 1])], [2, 130([137(1), "a"])], [3, 130([137(0), true])]]]])`

Within the CCF data item, the type is `161([h'', "S.test.Foo", [["bar", 137(4)], ["baz", 137(39)]]])` , which defines a resource type:
- `h''` is CCF type ID `0`. This ID is used inside the value data item to bind the type with the raw value.
- `"S.test.Foo"` is the Cadence type ID.
- `["bar", 137(4)]` is the first field: `"bar"` field of Cadence `Int`.
- `["baz", 137(39)]` is the second field: `"baz"` field of Cadence `AnyStruct`.

Within the CCF data item, the value is `[139(136(h'')), [[1, 130([137(4), 1])], [2, 130([137(1), "a"])], [3, 130([137(0), true])]]]`, where:
- `139(136(h''))` is the Cadence array of the type identified by ID `0`.
- `[1, 130([137(4), 1])]` is the raw field data of the first resource.
- `[2, 130([137(1), "a"])]` is the raw field data of the second resource.
- `[3, 130([137(0), true])]` is the raw field data of the third resource.

### `FeesDeducted` Event

Cadence `FeesDeducted` event encodes to:
- 298 bytes in JSON-CDC (minified):
- 118 bytes in CCF (fully self-describing mode)

JSON-CDC encoding (not minified for readability):

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

CCF encoding (in hex for readability):

`d8818281d8a283407828412e663931396565373734343762373439372e466c6f77466565732e466565734465647563746564838266616d6f756e74d88917826f657865637574696f6e4566666f7274d88917826f696e636c7573696f6e4566666f7274d8891782d8884083190b9919023f1a05f5e100`

The CCF data item in diagnostic notation is:

`129([[162([h'', "A.f919ee77447b7497.FlowFees.FeesDeducted", [["amount", 137(23)], ["executionEffort", 137(23)], ["inclusionEffort", 137(23)]]])], [136(h''), [2969, 575, 100000000]]])`.

CCF data items contain type definition and value in fully self-describing mode.

Within the CCF data item, the type is `162([h'', "A.f919ee77447b7497.FlowFees.FeesDeducted", [["amount", 137(23)], ["executionEffort", 137(23)], ["inclusionEffort", 137(23)]]])`:
- `h''` is CCF type ID `0`. This ID is used inside the value data item to bind the type with the raw value.
- `"A.f919ee77447b7497.FlowFees.FeesDeducted"` is the Cadence type ID.
- `[["amount", 137(23)], ["executionEffort", 137(23)], ["inclusionEffort", 137(23)]]` is field definition with 3 fields: `"amount"` field of Cadence `UFix64`, `"inclusionEffort"` field of Cadence `UFix64`, and `"executionEffort"` field of Cadence `UFix64`. Fields are sorted by field name.

Within the CCF data item, the value is  `[136(h''), [2969, 575, 100000000]]`, where:
- `136(h'')` is the type identified by ID `0`.
- `[2969, 575, 100000000]` is `FeesDeducted` event raw field data.

## Specifications

There are 3 top-level CCF messages: `ccf-typedef-message`, `ccf-typedef-and-value-message`, and `ccf-type-and-value-message`.

- `ccf-typedef-message` is a list of type definitions for Cadence composite types.
- `ccf-typedef-and-value-message` is a Cadence value preceded by a list of type definitions of referenced Cadence composite types.
- `ccf-type-and-value-message` is a Cadence value preceded by built-in or known composite Cadence type(s).

Cadence types are encoded as `inline-type` (inlined) or as `composite-typedef` (not inlined).

Cadence data is encoded depending on its type. For example, Cadence `UInt8` is encoded as a CBOR unsigned integer, Cadence `String` is encoded as a CBOR text string, Cadence `Address` is encoded as a CBOR byte string, and Cadence struct data is encoded as an array of its raw field data.

### Cadence Types and Type Values

Cadence has types and values. A "type value" is a value that represents a type.

Cadence types and Cadence type values are encoded differently. They contain different data because they are used differently.

Cadence types are used to decode Cadence data, so they only contain information needed for decoding. For example, field information of a composite type is needed to decode the composite value. However, field information of an interface type isn't needed to decode values implementing the interface type.

Cadence type value is a Cadence value that provides comprehensive information about a type. For example, composite type value and interface type value contain information about both fields and initializers.

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
cbor-tag-intersection-type = 143
cbor-tag-capability-type = 144
cbor-tag-inclusiverange-type = 145
cbor-tag-entitlement-set-authorization-type = 146
cbor-tag-entitlement-map-authorization-type = 147
; 148-159 are reserved

; composite types
cbor-tag-struct-type = 160
cbor-tag-resource-type = 161
cbor-tag-event-type = 162
cbor-tag-contract-type = 163
cbor-tag-enum-type = 164
cbor-tag-attachment-type = 165
; 166-175 are reserved

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
cbor-tag-intersection-type-value = 191
cbor-tag-capability-type-value = 192
cbor-tag-function-type-value = 193
cbor-tag-inclusiverange-type-value = 194
cbor-tag-entitlement-set-authorization-type-value = 195
cbor-tag-entitlement-map-authorization-type-value = 196
; 197-207 are reserved

; composite type values
cbor-tag-struct-type-value = 208
cbor-tag-resource-type-value = 209
cbor-tag-event-type-value = 210
cbor-tag-contract-type-value = 211
cbor-tag-enum-type-value = 212
cbor-tag-attachment-type-value = 213
; 214-223 are reserved

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
        / attachment-type
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
            field-type: inline-type,
        ]
    ]
]

; interface-type doesn't include field information because it's not needed for
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

attachment-type =
    ; cbor-tag-attachment-type
    #6.165(composite-type)

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
    / intersection-type
    / capability-type
    / inclusiverange-type
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
        element-type: inline-type,
    ])

dict-type =
    ; cbor-tag-dict-type
    #6.141([
        key-type: inline-type,
        element-type: inline-type,
    ])

reference-type =
    ; cbor-tag-reference-type
    #6.142([
      authorized: authorization-type,
      type: inline-type,
    ])

intersection-type =
    ; cbor-tag-intersection-type
    #6.143([
      types: [+ inline-type],
    ])

capability-type =
    ; cbor-tag-capability-type
    ; use an array as an extension point
    #6.144([
        ; borrow-type
        inline-type / nil,
    ])

inclusiverange-type =
    ; cbor-tag-inclusiverange-type
    #6.145(inline-type)

authorization-type =
    unauthorized-type
    / entitlement-set-authorization-type
    / entitlement-map-authorization-type

unauthorized-type = nil

entitlement-set-authorization-type =
	; cbor-tag-entitlement-set-authorization-type
	#6.146([
	    kind: uint,
	    entitlements: [+ tstr],
	])

entitlement-map-authorization-type =
	; cbor-tag-entitlement-map-authorization-type
	#6.147(tstr)

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
    capability-type-id: 25,
    storage-path-type-id: 26,
    public-path-type-id: 27,
    private-path-type-id: 28,
    deployed-contract-type-id: 35,
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
    storage-capability-controller-type-id: 56,
    account-capability-controller-type-id: 57,
    account-type-id: 58,
    account-contracts-type-id: 59,
    account-keys-type-id: 60,
    account-inbox-type-id: 61,
    account-storage-capabilities-type-id: 62,
    account-account-capabilities-type-id: 63,
    account-capabilities-type-id: 64,
    account-storage-type-id: 65,
    mutate-type-id: 66,
    insert-type-id: 67,
    remove-type-id: 68,
    identity-type-id: 69,
    storage-type-id: 70,
    save-value-type-id: 71,
    load-value-type-id: 72,
    copy-value-type-id: 73,
    borrow-value-type-id: 74,
    contracts-type-id: 75,
    add-contract-type-id: 76,
    update-contract-type-id: 77,
    remove-contract-type-id: 78,
    keys-type-id: 79,
    add-key-type-id: 80,
    revoke-key-type-id: 81,
    inbox-type-id: 82,
    publish-inbox-capability-type-id: 83,
    unpublish-inbox-capability-type-id: 84,
    claim-inbox-capability-type-id: 85,
    capabilities-type-id: 86,
    storage-capabilities-type-id: 87,
    account-capabilities-type-id: 88,
    publish-capability-type-id: 89,
    unpublish-capability-type-id: 90,
    get-storage-capability-controller-type-id: 91,
    issue-storage-capability-controller-type-id: 92,
    get-account-capability-controller-type-id: 93,
    issue-account-capability-controller-type-id: 94,
    capabilities-mapping-type-id: 95,
    account-mapping-type-id: 96,
    hashable-struct-type-id: 97,
    fixedSize-unsigned-integer-type-id: 98,
)

ccf-typedef-and-value-message =
    ; cbor-tag-typedef-and-value
    #6.129([
      typedef: composite-typedef,
      type-and-value: inline-type-and-value,
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
    / inclusiverange-value
    / function-value
    / type-value

optional-value = nil / value

array-value = [* value]

dict-value = [* (key: value, value: value)]

; composite-value is used to encode struct, contract, enum, event, resource, and attachment.
composite-value = [* (field: value)]

path-value = [
    domain: uint,
    identifier: tstr,
]

capability-value = [
    address: address-value,
    id: uint64-value,
]

inclusiverange-value = [
    start: value,
    end: value,
    step: value,
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
           type-bound: type-value / nil,
        ]
    ],
    parameters: [
        * [
            label: tstr,
            identifier: tstr,
            type: type-value,
        ]
    ],
    return-type: type-value,
    purity: int,
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
    / attachment-type-value
    / struct-interface-type-value
    / resource-interface-type-value
    / contract-interface-type-value
    / function-type-value
    / reference-type-value
    / intersection-type-value
    / capability-type-value
    / inclusiverange-type-value
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
        element-type: type-value,
    ])

dict-type-value =
    ; cbor-tag-dict-type-value
    #6.189([
        key-type: type-value,
        element-type: type-value,
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

attachment-type-value =
    ; cbor-tag-attachment-type-value
    #6.213(composite-type-value)

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
            type: type-value,
        ]
    ],
    initializers: [
        ? [
            * [
                label: tstr,
                identifier: tstr,
                type: type-value,
            ]
        ]
    ],
]

function-type-value =
    ; cbor-tag-function-type-value
    #6.193(function-value)

reference-type-value =
    ; cbor-tag-reference-type-value
    #6.190([
      authorized: authorization-type-value,
      type: type-value,
    ])

intersection-type-value =
    ; cbor-tag-intersection-type-value
    #6.191([
      types: [+ type-value]
    ])

capability-type-value =
    ; cbor-tag-capability-type-value
    ; use an array as an extension point
    #6.192([
      ; borrow-type
      type-value / nil
    ])

inclusiverange-type-value =
    ; cbor-tag-inclusiverange-type-value
    #6.194(type-value)

authorization-type-value =
    unauthorized-type-value
    / entitlement-set-authorization-type-value
    / entitlement-map-authorization-type-value

unauthorized-type-value = nil

entitlement-set-authorization-type-value =
	; cbor-tag-entitlement-set-authorization-type-value
	#6.195([
	    kind: uint,
	    entitlements: [+ tstr],
	])

entitlement-map-authorization-type-value =
	; cbor-tag-entitlement-map-authorization-type-value
	#6.196(tstr)

type-value-ref =
    ; cbor-tag-type-value-ref
    #6.184(id)

;CDDL-END
```

## Acknowledgments

This document would not exist without Ramtin M. Seraj and Bastian Müller.

Ramtin and Bastian's contributions to this effort are hard to list exhaustively because they inspire individuals and teams to produce impactful results.

Ramtin M. Seraj led the effort to require a deterministic and more compact alternative to JSON-Cadence Data Interchange Format. This document's "Objectives" section includes and adds to the initial objectives Ramtin listed (in a notion) for a binary format for Cadence external values.

Bastian Müller helped the author understand details related to Cadence types and values, which made writing this document possible. Bastian also led the PR reviews of this document, contributed changes, asked great questions during CCF meetings, and identified sections that needed clarification (in both the CDDL notation and English text).
