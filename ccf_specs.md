# DRAFT - Cadence Compact Format (CCF)

Author: Faye Amacker  
Status: ABRIDGED DRAFT  
Date: November 29, 2022  
Revision: 20221129c  

To simplify initial review of the most important aspects, some verbose content is left out (e.g. list of numeric values representing each built-in Cadence type).  The omitted content will be provided in a less abridged version of this document after the first review is completed.

## Abstract

This document defines the Cadence Compact Format (CCF). CCF is a data format designed for compact, efficient, and deterministic encoding of Cadence external values.  This format is intended to be an alternative to and eventually replace [JSON-Cadence Data Interchange Format](https://developers.flow.com/cadence/json-cadence-spec).

[Concise Binary Object Representation (CBOR)](https://www.rfc-editor.org/rfc/rfc8949.html) is an IETF Internet Standard data format with a data model that is a superset of JSON's data model.  CCF uses a subset of CBOR's data model with CBOR Preferred Serialization to deterministically encode values to their smallest form.  CBOR was designed with security considerations in mind, such as malformed data detection.  CCF codecs can inherit detection of malformed data by using an existing CBOR codec.

CCF separates encoding of Cadence type info and values to avoid unnecessarily repeating Cadence type info (e.g. for a homogenous array, the same Cadence type info isn't repeatedly encoded).  This separation allows more compact encoded size.  Another distinct advantage is support for more compact communication size.  Detachable Cadence type info (for all composite types) enables protocols using CCF to optionally transmit the Cadence type info just once rather than repeating it for every message matching the same type.

## Introduction

Currently, Cadence external values (e.g. transaction arguments and events) are being encoded using JSON-Cadence Data Interchange format, which is inefficient, verbose, and doesn't define deterministic encoding (canonical form).

A need exists to provide a more compact, efficient, and deterministic encoding of Cadence external values.

CCF leverages CBOR's data model and Preferred Serialization to deterministically encode values to their smallest form.  This document uses CDDL notation to specify CCF.

- [CDDL (RFC 8610)](https://www.rfc-editor.org/rfc/rfc8610.html) is the Concise Data Definition Language. CDDL is a notation for unambiguously expressing CBOR and JSON data structures.

- [EDN (Appendix G of RFC 8610)](https://www.rfc-editor.org/rfc/rfc8610.html#appendix-G) is the Extended Diagnostic Notation.  EDN is used by this document to describe examples of CCF encoding.

- [CBOR (RFC 8949)](https://www.rfc-editor.org/rfc/rfc8949.html) is an IETF Internet Standard (not just a regular RFC) and is used by other standards, such as W3C WebAuthn, IETF COSE (RFC 9052), and IETF CWT (RFC 8392).  CBOR is extensible without version negotiation and is designed to be relevant for decades.

CBOR is a self-describing binary data format that (among other improvements) extends JSON's data model by allowing for binary data.  CBOR's data model is a superset of JSON's data model.  CBOR supports deterministic encoding with Preferred Serialization and Core Deterministic Encoding Requirements as defined in RFC 8949.

### CCF Design

CCF leverages CBOR's data model and CBOR Preferred Serialization to deterministically encode values to their smallest form.

CCF separates Cadence type encoding from Cadence value encoding. This has two distinct advantages:

- More compact encoding.  Cadence type info is not repeatedly encoded unnecessarily.  E.g. for a homogeneous array of a Cadence composite type, each element will not have its Cadence composite type info encoded repeatedly.

- Detachable Cadence type info. Although Cadence type info is required for decoding, CCF-based protocols have the ability to send a specific type info once instead of repeatedly sending it to the same client for all messages matching that type.

CCF is designed to support:

- All current and future Cadence types, including composite types.  CCF supports schemaless encoding and is extensible for new Cadence types.

- Compact encoding.  Smaller encoded size is provided by:
  - Using CBOR's data model with CBOR Preferred Serialization, which has more compact encoding than JSON.
  - Separate encoding of Cadence types and values avoid repeatedly encoding the same Cadence type info unecessarily (e.g. avoid encoding the same Cadence type info for every element in a homogeneous array).

- Compact communications.  Detachable Cadence type info allows CCF-based protocols to optionally avoid resending the same Cadence type info for all messages matching that type.  CCF-based protocols can cache and uniquely identify a Cadence type so it can be matched to Cadence value (such as an event) during decoding.

- Deterministic encoding.  CCF uses CBOR's [Preferred Serialization](https://www.rfc-editor.org/rfc/rfc8949.html#name-preferred-serialization) to achieve deterministic encoding.  Other parts of CBOR's Core Determinisitic Encoding Requirements are not needed by this specification.

- Early detection of malformed data.  CCF decoders can detect and reject malformed data without creating Cadence objects.  CCF decoders can detect malformed data without having Cadence type info.  If data is not malformed, then CCF decoders can proceed to detect and reject invalid CCF data as described in this document.

- Extensibility.  CCF encodes composite type information in a header (separate from data). Data refers to composite types by unique type ID, encoded as bytes. In the future, composite type information can be stored on-chain and header isn't necessary to be sent with data. Type ID can be an universal counter or hash value.

- Interoperability and Reuse.  CCF uses the same approach taken by COSE (RFC 9052) leveraging CBOR (RFC 8949).  CCF leverages CBOR, which allows CCF codecs to use CBOR codecs under the hood.

- Translation to JSON.  CCF is defined using CDDL notation.  CBOR's data model is a superset of JSON's data model.  CCF uses a subset of CBOR's data model which allows translation from CCF to JSON-based formats.

## Why CBOR

It's good practice to leverage a standard data format (like CBOR) to define more specific data formats and protocols.

Although using a 100% custom data format instead of leveraging an existing data format can sometimes produce smaller encoded data size, the benefits of using CBOR outweigh this tradeoff.

Cadence is [already using CBOR](https://github.com/onflow/cadence/blob/master/runtime/interpreter/encode.go) to encode internal values.  Using CBOR to also encode external values is a good fit for multiple reasons.

Concise Binary Object Representation (CBOR) is a data format defined in IETF [RFC 8949](https://www.rfc-editor.org/rfc/rfc8949.html):

> The Concise Binary Object Representation (CBOR) is a data format
   whose design goals include the possibility of extremely small code
   size, fairly small message size, and extensibility without the need
   for version negotiation.  These design goals make it different from
   earlier binary serializations such as ASN.1 and MessagePack.

As an [Internet Standard](https://en.wikipedia.org/wiki/Internet_Standard) (not just a common RFC), CBOR is used to define other formats and protocols such as:
- W3C [WebAuthn](https://www.w3.org/TR/webauthn-2/) - Web Authentication
- IETF [RFC 9052](https://www.rfc-editor.org/rfc/rfc9052.html) - CBOR Object Signing and Encryption (COSE): Structures and Process
- IETF [RFC 8392](https://www.rfc-editor.org/rfc/rfc8392.html) - CBOR Web Tokens (CWT)

This approach enables a COSE codec to use a CBOR codec under the hood.  COSE codecs can focus on providing COSE-specific features rather than reinventing the wheel, which reduces complexity, cost of development, and risks.

As an Internet Standard, CBOR is designed to avoid requiring changes and remain relevant for decades.

CBOR is designed to allow extremely small code size (e.g. usable on constrained devices), fairly small message size, and extensibility without version negotiation.

CBOR was designed with security considerations in mind.  It allows CBOR codecs to have separate detection of malformed data and invalid data.

CBOR defines deterministic encoding with Preferred Serialization and Core Deterministic Encoding Requirements.

CBOR is well-suited to replace JSON-based data formats and protocols.  CBOR's data model extends JSON's data model with:
- compact binary encodings
- extension points (CBOR Tags and Simple Values)
- deterministic encoding as already mentioned

As one example, CBOR Web Tokens is a modern binary alternative to the text-based JSON Web Tokens (JWT).

Like CBOR Web Tokens, CCF is a modern binary alternative to an existing JSON-based data format.  CCF uses CBOR to provide a compact, efficient, and deterministic data format.

Additional considerations for using CBOR include availability and quality of CBOR codecs in various programming languages.

### Interoperability and Reuse of CBOR Codecs

High quality CBOR codecs are available in various programming languages.

In JavaScript, there are multiple CBOR codecs that are actively maintained. As one example, [hildjj/node-cbor](https://github.com/hildjj/node-cbor) is  maintained by Joe Hildebrand (former VP of Engineering at Mozilla and Distinguished Engineer at Cisco). It's a monorepo with several packages including a "cbor package compiled for use on the web, including all of its non-optional dependencies".

In Go, [fxamacker/cbor](https://github.com/fxamacker/cbor) is a fuzz-tested CBOR codec [already used by Cadence](https://github.com/onflow/cadence/blob/master/runtime/interpreter/encode.go) for internal value encoding.  It's fast, easy to use, and passed multiple security assessments this year without known problems. It is used by Arm Ltd., Chainlink, ConsenSys, Dapper Labs, Duo Labs (cisco), EdgeX Foundry, Microsoft, Mozilla, Oasis Labs, Tailscale, Taurus SA, and many others.  GitHub reports fxamacker/cbor/v2 is used by over 1275 repositories.

## Security Considerations

CBOR security considerations are described in [Section 10 of RFC 8949 (CBOR)](https://www.rfc-editor.org/rfc/rfc8949.html#name-security-considerations).

There are two types of data validation:
- Is the data well-formed?
- Is the data valid?

CBOR defines data [well-formedness](https://www.rfc-editor.org/rfc/rfc8949.html#name-well-formedness-errors-and-) and a CBOR decoder MUST detect and reject malformed data before checking for validity.

CCF message validation should be done after passing checks for well-formedness.  Invalid CCF messages should be rejected.

CCF decoder should detect and reject malformed or malicious data before creating Cadence objects.  Check for well-formedness can be done on other node types without asking EN for the Cadence type info.

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
d88282d88404c2412a
```

It represents `130([132(4), 42])`, where
- `132(4)` is Cadence `Int`.
- `42` is raw value.

### Cadence Homogenous Array with Simple Type Elements

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
d88282d886d8840483c24101c24102c24103
```

It represents `130([134(132(4)), [1, 2, 3]])`, where
- `134(132(4))` is Cadence array of `Int`.
- `[1, 2, 3]` is raw value.

### Cadence Heterogenous Array with Simple Type Elements

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
d88282d886d884182783d88282d88404c24101d88282d884016161d88282d88400f5
```

It represents `130([134(132(39)), [130([132(4), 1]), 130([132(1), "a"]), 130([132(0), true])]])`, where
- `134(132(39))` is Cadence array of `AnyStruct`.
- `130([132(4), 1])` is first element (`132(4)` is Cadence `Int`, `1` is raw value)
- `130([132(1), "a"])` is second element (`132(1)` is Cadence `String`, `"a"` is raw value)
- `130([132(0), true])` is third element (`132(0)` is Cadence `Boolean`, `true` is raw value)

### Cadence Homogenous Array with Composite Type Elements

Cadence composite types, such as struct, resource, and event, are encoded as detachable (not inlined) type info.  Each encoded composite type info has an unique ID which is used to bind with Cadence value.

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

CCF encoding is 48 bytes (first 27 bytes for type and next 21 bytes for raw value):

```
d88081d88a83406a532e746573742e466f6f818263626172d88404d88282d886d883408381c2410181c2410281c24103
```

It contains two data items: type and value.

Type data item represents `128([138([h'', "S.test.Foo", [["bar", 132(4)]]])])` , where
- `h''` is unique ID `0`.  This ID is used inside value data item to bind type with raw value.
- `S.test.Foo` is location and identifier.
- `[["bar", 132(4)]]` is field definition of one field: `"bar"` field of Cadence `Int`.

Value data item represents `130([134(131(h'')), [[1], [2], [3]]])`, where
- `134(131(h''))` is Cadence array of type identified by ID `0`.
- `[[1], [2], [3]]]` is array of `Foo` resouce raw field data.

### Cadence Homogenous Array with Composite Type Elements (One Field Type Is Abstract)

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

CCF encoding is 81 bytes (first 36 bytes for type and next 45 bytes for raw value):

```
d88081d88a83406a532e746573742e466f6f828263626172d88404826362617ad8841827d88282d886d883408382c24101d88282d88404c2410182c24102d88282d88401616182c24103d88282d88400f5
```

It contains two data items: type and value.

Type data item represents `128([138([h'', "S.test.Foo", [["bar", 132(4)], ["baz", 132(39)]]])])`, where
- `h''` is unique ID `0`.  This ID is used inside value data item to bind type with raw value.
- `S.test.Foo` is location and identifier.
- `["bar", 132(4)]` is first field definition: `"bar"` field of Cadence `Int`.
- `["baz", 132(39)]` is second field definition: `"baz"` field of Cadence `AnyStruct`.

Value data item represents `130([134(131(h'')), [[1, 130([132(4), 1])], [2, 130([132(1), "a"])], [3, 130([132(0), true])]]])`, where
- `134(131(h''))` is Cadence array of type identified by ID `0`.
- `[1, 130([132(4), 1])]` is first resource raw field data.
- `[2, 130([132(1), "a"])]` is second resource raw field data.
- `[3, 130([132(0), true])]` is third resource raw field data.

### `FeesDeducted` Event

`FeesDeducted` event is encoded as detachable (not inlined) type info.

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

CCF encoding is 119 bytes (first 101 bytes for type and next 18 bytes for raw value):

```
d88081d88b83407828412e663931396565373734343762373439372e466c6f77466565732e466565734465647563746564838266616d6f756e74d88417826f696e636c7573696f6e4566666f7274d88417826f657865637574696f6e4566666f7274d88417d88282d8834083190b991a05f5e10019023f
```

It contains two data items: type and value.

Type data item represents `128([139([h'', "A.f919ee77447b7497.FlowFees.FeesDeducted", [["amount", 132(23)], ["inclusionEffort", 132(23)], ["executionEffort", 132(23)]]])])` , where
- `h''` is unique ID `0`.  This ID is used inside value data item to bind type with raw value.
- `"A.f919ee77447b7497.FlowFees.FeesDeducted"` is location and identifier.
- `[["amount", 132(23)], ["inclusionEffort", 132(23)], ["executionEffort", 132(23)]]` is field definition with 3 fields: `"amount"` field of Cadence `UFix64`, `"inclusionEffort"` field of Cadence `UFix64`, and `"executionEffort"` field of Cadence `UFix64`.

Value data item represents `130([131(h''), [2969, 100000000, 575]])`, where
- `131(h'')` is type identified by ID `0`.
- `[2969, 100000000, 575]` is `FeesDeducted` event raw field data.

## Terminology

### CBOR

This document uses these CBOR data items as defined in RFC 8949:
- nil
- bool
- positive integer
- negative integer
- byte string
- text string
- bignum
- array
- tagged data item (tag): comprised of a tag number and tag content

## CDDL

This document uses CDDL notation to express CBOR data items:
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

## Extended Diagnostic Notation (EDN)

This document uses diagnostic notation as defined in Appendix G of RFC 8610.

## Specifications

There are 2 top level CCF messages: `ccf-composite-type-message` and `ccf-type-and-value-message`.

Cadence types are encoded either inlined as `inline-type` or not inlined as `ccf-composite-type-message`.

Cadence data is encoded depending on its type.
For example, Cadence `UInt8` is encoded as CBOR positive integer, Cadence `String` is encoded as CBOR text string,
Cadence `Address` is encoded as CBOR byte string, and Cadence struct data is encoded as an array of its raw field data.

```cddl
;CDDL-BEGIN

; CCF uses CBOR tag numbers 128-255, which are unassigned by [IANA](https://www.iana.org/assignments/cbor-tags/cbor-tags.xhtml).

; NOTE: when changing values, also update uses in rules!

; CBOR tag numbers for root objects
cbor-tag-type = 128
; 129 is reserved
cbor-tag-type-and-value = 130

; CBOR tag numbers for types
cbor-tag-type-ref = 131
cbor-tag-simple-type = 132
cbor-tag-optional-type = 133
cbor-tag-varsized-array-type = 134
cbor-tag-constsized-array-type = 135
cbor-tag-dict-type = 136
cbor-tag-struct-type = 137
cbor-tag-resource-type = 138
cbor-tag-event-type = 139
cbor-tag-contract-type = 140
cbor-tag-enum-type = 141
cbor-tag-capability-type = 142

; CBOR tag numbers for type value
cbor-tag-optional-type-value = 143
cbor-tag-varsized-array-type-value = 144
cbor-tag-constsized-array-type-value = 145
cbor-tag-dict-type-value = 146
cbor-tag-struct-type-value = 147
cbor-tag-resource-type-value = 148
cbor-tag-event-type-value = 149
cbor-tag-contract-type-value = 150
cbor-tag-enum-type-value = 151
cbor-tag-struct-interface-type-value = 152
cbor-tag-resource-interface-type-value = 153
cbor-tag-contract-interface-type-value = 154
cbor-tag-function-type-value = 155
cbor-tag-reference-type-value = 156
cbor-tag-restricted-type-value = 157
cbor-tag-capability-type-value = 158

ccf-message = (
    (? ccf-composite-type-message),
    ccf-type-and-value-message
)

ccf-composite-type-message =
    ; cbor-tag-type
    #6.128([
        + (
            struct-type
            / resource-type
            / contract-type
            / event-type
            / enum-type
        )
    ])

composite-type = [
	id: bstr,
	location-identifier: tstr,
	fields: [
        + [
            field-name: tstr,
            field-type: inline-type
        ]
    ]
]

struct-type =
    ; cbor-tag-struct-type
    #6.137(composite-type)

resource-type =
    ; cbor-tag-resource-type
    #6.138(composite-type)

contract-type =
    ; cbor-tag-contract-type
    #6.140(composite-type)

event-type =
    ; cbor-tag-event-type
    #6.139(composite-type)

enum-type =
    ; cbor-tag-enum-type
    #6.141(composite-type)

inline-type =
    simple-type
    / optional-type
    / varsized-array-type
    / constsized-array-type
    / dict-type
    / capability-type
    / type-ref

simple-type =
    ; cbor-tag-simple-type
    #6.132(simple-type-id)

optional-type =
    ; cbor-tag-optional-type
    #6.133(inline-type)

varsized-array-type =
    ; cbor-tag-varsized-array-type
    #6.134(inline-type)

constsized-array-type =
    ; cbor-tag-constsized-array-type
    #6.135([
        array-size: uint,
        element-type: inline-type
    ])

dict-type =
    ; cbor-tag-dict-type
    #6.136([
        key-type: inline-type,
        element-type: inline-type
    ])

capability-type =
    ; cbor-tag-capability-type
    #6.142(
        ; borrow-type
        inline-type
    )

type-ref =
    ; cbor-tag-type-ref
    #6.131(
        ; composite-type-id
        bstr
    )

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
)

ccf-type-and-value-message =
    ; cbor-tag-type-and-value
    #6.130([
      type: inline-type,
      value: value
    ])

value =
    ccf-type-and-value-message
    / simple-value
    / optional-value
    / array-value
    / dict-value
    / composite-value
    / link-value
    / path-value
    / capability-value
    / function-value
    / type-value

optional-value = nil / value

array-value = [* value]

dict-value = [* (key: value, value: value)]

; composite-value is used to encode struct, contract, enum, event, and resource.
composite-value = [+ (field: value)]

link-value = [
    path: path-value,
    borrow-type: tstr,
]

path-value = [
    domain: path-domain,
    identifier: tstr,
]

; path-domain is an enumeration
path-domain = &(
    domain-storage: 1,
    domain-private: 2,
    domain-public: 3,
)

capability-value = [
    address: address-value,
    path: path-value
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
    / fix64-value
    / ufix64-value

void-value = nil
bool-value = bool
character-value = tstr
string-value = tstr
address-value = bstr .size 8
uint-value = bigint
uint8-value = uint
uint16-value = uint
uint32-value = uint
uint64-value = uint
uint128-value = bigint
uint256-value = bigint
int-value = bigint
int8-value = int
int16-value = int
int32-value = int
int64-value = int
int128-value = bigint
int256-value = bigint
word8-value = uint
word16-value = uint
word32-value = uint
word64-value = uint
fix64-value = int
ufix64-value = uint

function-value = function-type-value

type-value =
    simple-type-value
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
    / type-ref

simple-type-value = simple-type

optional-type-value =
    ; cbor-tag-optional-type-value
    #6.143(type-value)

varsized-array-type-value =
    ; cbor-tag-varsized-array-type-value
    #6.144(type-value)

constsized-array-type-value =
    ; cbor-tag-constsized-array-type-value
    #6.145([
        array-size: uint,
        element-type: type-value
    ])

dict-type-value =
    ; cbor-tag-dict-type-value
    #6.146([
        key-type: type-value,
        element-type: type-value
    ])

struct-type-value =
    ; cbor-tag-struct-type-value
    #6.147(composite-type-value)

resource-type-value =
    ; cbor-tag-resource-type-value
    #6.148(composite-type-value)

contract-type-value =
    ; cbor-tag-contract-type-value
    #6.150(composite-type-value)

event-type-value =
    ; cbor-tag-event-type-value
    #6.149(composite-type-value)

enum-type-value =
    ; cbor-tag-enum-type-value
    #6.151(composite-type-value)

struct-interface-type-value =
    ; cbor-tag-struct-interface-type-value
    #6.152(composite-type-value)

resource-interface-type-value =
    ; cbor-tag-resource-interface-type-value
    #6.153(composite-type-value)

contract-interface-type-value =
    ; cbor-tag-contract-interface-type-value
    #6.154(composite-type-value)

composite-type-value = [
    type: nil / type-value,
    id: bstr,
    location-identifier: tstr,
    fields: [
        + [
            name: tstr,
            type: type-value
        ]
    ]
    initializers: [
        * [
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
    #6.155([
      id: tstr,
      parameters: [
        * [
            label: tstr,
            identifier: tstr,
            type: type-value
        ]
      ]
      return-type: type-value
    ])

reference-type-value =
    ; cbor-tag-reference-type-value
    #6.156([
      authorized: bool,
      type: type-value,
    ])

restricted-type-value =
    ; cbor-tag-restricted-type-value
    #6.157([
      id: tstr,
      type: type-value,
      restrictions: [+ type-value]
    ])

capability-type-value =
    ; cbor-tag-capability-type-value
    #6.158(
      ; borrow-type
      type-value
    )

;CDDL-END
```
