# DRAFT - Cadence Compact Format (CCF)

Author: Faye Amacker  
Status: ABRIDGED DRAFT  
Date: November 17, 2022  
Revision: 20221122a

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

Although using a 100% custom data format instead of leveraging an existing data format like CBOR can sometimes produce smaller encoded data size, the benefits of using CBOR outweigh this tradeoff.

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

It's good practice to leverage a standard data format to define more specific data formats and protocols.

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
```
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
``` 
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

### Cadence Homogenous Array with Composite Type Elements

Cadence composite types, such as struct, resource, and event, are encoded as detachable (not inlined) type info.  Each encoded composite type info has an unique ID which is used to bind with Cadence value.  

Cadence `[Foo]` type encoded to JSON is 353 bytes when minified:
```
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

### `FeesDeducted` Event

`FeesDeducted` event is encoded as detachable (not inlined) type info.

This example of `FeesDeducted` event encoded to JSON is 298 bytes when minified:
```
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
- `Foo + Bar`: Foo followed by Bar
- `[+ Foo]`: one or more Foo in CBOR array
- `[* Foo]`: zero or more Foo in CBOR array
- `#6.nnn(type)`: CBOR tag with tag number nnn and content type of "type".

## Extended Diagnostic Notation (EDN)

This document uses diagnostic notation as defined in Appendix G of RFC 8610.

## Specifications 

There are 2 top level CCF messages: CCF_CompositeTypeInfo_Message and CCF_TypeAndValue_Message.

Cadence types are encoded either inlined as CCF_InlineTypeInfo or not inlined as CCF_CompositeTypeInfo_Message.

Cadence data is encoded depending on its type.  For example, Cadence `UInt8` is encoded as CBOR positive integer, Cadence `String` is encoded as CBOR text string, Cadence `Address` is encoded as CBOR byte string, and Cadence struct data is encoded as an array of its raw field data.   

```

CCF_Message = (? CCF_CompositeTypeInfo_Message) + CCF_TypeAndValue_Message

CCF_CompositeTypeInfo_Message = #6.CBORTagType([+ (CCF_StructTypeInfo_Message / CCF_ResourceTypeInfo_Message / CCF_ContractTypeInfo_Message / CCF_EventTypeInfo_Message / CCF_EnumTypeInfo_Message)])

composite_type_record = [
	id: bstr,
	location_identifier: tstr,
	fields: [+ [field_name: tstr, field_type: CCF_InlineTypeInfo]]
]
       
CCF_StructTypeInfo_Message = #6.CBORTagStructType(composite_type_record)

CCF_ResourceTypeInfo_Message = #6.CBORTagResourceType(composite_type_record)

CCF_ContractTypeInfo_Message = #6.CBORTagContractType(composite_type_record) 

CCF_EventTypeInfo_Message = #6.CBORTagEventType(composite_type_record)

CCF_EnumTypeInfo_Message = #6.CBORTagEnumType(composite_type_record)

CCF_InlineTypeInfo = CCF_SimpleTypeInfo / CCF_OptionalTypeInfo / CCF_VarSizedArrayTypeInfo / CCF_ConstSizedArrayTypeInfo / CCF_DictTypeInfo / CCF_CapabilityTypeInfo / CCF_TypeRef  

CCF_SimpleTypeInfo = #6.CBORTagSimpleType(simple_type_id)

simple_type_id = uint

CCF_OptionalTypeInfo = #6.CBORTagOptionalType(CCF_InlineTypeInfo)

CCF_VarSizedArrayTypeInfo = #6.CBORTagVarArrayType(CCF_InlineTypeInfo)

CCF_ConstSizedArrayTypeInfo = #6.CBORTagConstArrayType([array_size: uint, element_type: CCF_InlineTypeInfo])

CCF_DictTypeInfo = #6.CBORTagDictType([key_type: CCF_InlineTypeInfo, element_type: CCF_InlineTypeInfo])

CCF_CapabilityTypeInfo = #6.CBORTagCapabilityType(borrow_type: CCF_InlineTypeInfo)

CCF_TypeRef = #6.CBORTagTypeRef(composite_type_id: bstr)

CCF_TypeAndValue_Message = #6.CBORTagTypeAndValue([
	type: CCF_InlineTypeInfo, 
	value: cadence_value / [cadence_value]
])

cadence_raw_simple_type_value = nil / bstr / tstr / uint / nint / bigint

cadence_value = CCF_TypeAndValue_Message / cadence_raw_simple_type_value

```
