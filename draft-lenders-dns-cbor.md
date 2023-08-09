---
title: "A Concise Binary Object Representation (CBOR) of DNS Messages"
abbrev: "dns+cbor"
category: std

docname: draft-lenders-dns-cbor-latest
number:
date:
consensus: true
v: 3
submissiontype: IETF
area: "Applications"
workgroup: CBOR
keyword:
 - Internet-Draft
 - CBOR
 - DNS
venue:
  group: CBOR
  type: Working Group
  mail: cbor@ietf.org
  arch: "https://mailarchive.ietf.org/arch/browse/cbor/"
  github: "anr-bmbf-pivot/draft-lenders-dns-cbor"
  latest: "https://anr-bmbf-pivot.github.io/draft-lenders-dns-cbor/draft-lenders-dns-cbor.html"

author:
 -  fullname: Martine Sophie Lenders
    organization: Freie Universität Berlin
    abbrev: FU Berlin
    email: m.lenders@fu-berlin.de
 -  fullname: Carsten Bormann
    organization: Universität Bremen TZI
    email: cabo@tzi.org
 -  fullname: Thomas C. Schmidt
    organization: HAW Hamburg
    email: t.schmidt@haw-hamburg.de
 -  name: Matthias Wählisch
    org: TUD Dresden University of Technology
    abbrev: TU Dresden
    street: Helmholtzstr. 10
    city: Dresden
    code: D-01069
    country: Germany
    email: m.waehlisch@tu-dresden.de

normative:
  RFC1035: dns
  RFC3596: aaaa
  RFC7252: coap
  RFC8610: cddl
  RFC8949: cbor
  I-D.ietf-cbor-packed: cbor-packed

informative:
  RFC4944: 6lowpan
  RFC6282: iphc
  RFC7228: constr-terms
  RFC8484: doh
  RFC8724: schc
  RFC8824: coap-schc
  RFC9110: http-semantics
  I-D.ietf-core-dns-over-coap: doc


--- abstract

This document specifies a compressed data format of DNS messages using
the Concise Binary Object Representation {{-cbor}}.
The primary purpose is to keep DNS messages small in constrained networks.


--- middle

# Introduction

In constrained networks {{-constr-terms}}, the link layer may restrict the payload sizes to
only a few hundreds bytes.  Encrypted DNS resolution, such as DNS over HTTPS (DoH) {{-doh}} or
DNS over CoAP (DoC) {{-doc}}, may lead to DNS message sizes that exceed this limit, even when
implementing header compression such as 6LoWPAN IPHC {{-iphc}} or SCHC {{-schc}},
{{-coap-schc}}.

Although adoption layers such as 6LoWPAN {{-6lowpan}} or SCHC {{-schc}} offer fragmentation to
comply with small MTUs, fragmentation should be avoided in constrained networks, because
fragmentation combined with high packet loss multiplies the loss.  As such, a compression
format for DNS messages is needed.

This document specifies a compressed data format for DNS messages.  DNS messages are encoded in
Concise Binary Object Representation (CBOR) {{-cbor}} and, additionally, unnecessary or
redundant information is removed.  To use the outcome of this specification in DoH and DoC,
this document also specifies a Media Type header for DoH and a Content-Format option for DoC.

# Terminology

CBOR types (unsigned integer, byte string, text string, arrays, etc.) are used as defined in
{{-cbor}}.

TBD DNS server and client.

A DNS query is a message that queries DNS information from an upstream DNS resolver.

The term "constrained networks" is used as defined in {{-constr-terms}}.

{::boilerplate bcp14-tagged}

# CBOR Representations (application/dns+cbor)

To keep overhead minimal, a DNS message is represented as CBOR arrays.  All CBOR items used in
this specification are of definite length.  CBOR arrays that do not follow the length
definitions of this or follow-up specifications, MUST be silently ignored.  It is assumed that
DNS query and DNS response are distinguished message types and that the query can be mapped to
the response by the transport protocol of choice.  To define the representation of binary
objects we use the Concise Data Definition Language (CDDL) {{-cddl}}.

If, for any reason, a DNS message is not representable in the CBOR format specified in this
document, a fallback to the another DNS message format, e.g., the classic DNS wire format, MUST
always be possible.

## Domain Name Representation {#sec:domain-names}

Domain names are represented in their commonly known string format (e.g., "example.org", see Section
2.3.1 in {{-dns}}) and in IDNA encoding {{!RFC5890}} as a text string. For the purpose of this
document, domain names remain case-insensitive as specified in {{-dns}}.

The representation of a domain name is defined in {{fig:domain-name}}.

{:cddl: artwork-align="center"}

~~~ CDDL
domain-name = tstr .regexp "([^.]+\.)*[^.]+"
~~~
{:cddl #fig:domain-name title="Domain Name Definition"}

## DNS Resource Records (RRs) {#sec:rr}
### Standard RRs

DNS resource records are encoded either in their binary form as a byte string or as CBOR arrays
containing 2 to 5 entries in the following order:

1. An optional name (as text string, see {{sec:domain-names}}),
2. A TTL (as unsigned integer),
3. An optional record type (as unsigned integer),
4. An optional record class (as unsigned integer), and lastly
5. A record data entry (as unsigned integer, negative integer, byte string, or text string).

If the first item of the resource record is a text string, it is its name.
If the name is elided, the name is derived from the question section of the message.
For responses, the question section is either taken from the query (see {{sec:queries}}) or provided
with the response see {{sec:responses}}.
The query may be derived from the transport context.

If the record type is elided, the record type from the question is assumed.
If record class is elided, the record class from the question is assumed.
When a record class is required, the record type MUST also be provided.

The byte format of the record data as a byte string follows the wire format as specified in Section
3.3 {{-dns}} (or other specifications of the respective record type).  Note that this format does
not include the RDLENGTH field from {{-dns}} as this value is encoded in the length field of the
CBOR byte string.

If the record data represents a domain name (e.g., for CNAME or PTR records), the record data MAY be
represented as a text string as specified in {{sec:domain-names}}.
This can save 1 byte of data, because the byte representation of DNS names requires both an
additional byte to define the length of the first name component and well as a zero byte at the end
of the name.
With CBOR on the other hand only 1 byte is required to define type and length of the text string up
until a string length of 23 characters.
Likewise, if the record data is purely a numerical value, it can be expressed as either an unsigned
or negative integer.

~~~ CDDL
type-spec = (
  record-type: uint,
  ? record-class: uint,
)
rr = (
  ? name: domain-name,
  ttl: uint,
  ? type-spec,
  rdata: int / bstr / domain-name,
)
~~~
{:cddl #fig:dns-standard-rr title="DNS Standard Resource Record Definition"}

### EDNS OPT Pseudo-RRs {#sec:edns}

TBD; reverse extended flags to get MSB-defined DO into LSB?

~~~ CDDL
opt-rcode-v = (
  rcode: uint .default 0,
  ? version: uint .default 0,
)
opt-rcode-v-flags = (
  flags: uint .default 0,
  ? opt-rcode-v,
)
opt-attr-val = (
  ocode: uint,
  odata: bstr,
)
opt = [opt-attr-val]
opt-rr = (
  ? udp-payload-size: uint .default 512,
  options: [* opt],
  ? opt-rcode-v-flags,
)
~~~
{:cddl #fig:dns-opt-rr title="DNS OPT Resource Record Definition"}

### Other special RRs
Further special records, e.g., TSIG can be defined in other specifications and are out of scope of
this document.

### RR Definition

The representation of a DNS resource records is defined in {{fig:dns-rr}}.

~~~ CDDL
dns-rr = [rr] / #6.20([opt-rr]) / bstr
~~~
{:cddl #fig:dns-rr title="DNS Resource Record Definition"}

## DNS Queries {#sec:queries}

DNS queries are encoded as CBOR arrays containing up to 5 entries in the following order:

1. An optional transaction ID (as unsigned integer),
2. An optional flag field (as unsigned integer),
3. The question section (as array),
4. An optional authority section (as array), and
5. An optional additional section (as array)

If the first item of the query is an array, it is the question section, if it is an unsigned
integer, it is the transaction ID.
If the transaction ID is present and followed by another unsigned integer, that item is a flag
field and maps to the header flags in {{-dns}} and the "DNS Header Flags" IANA registry including
the QR flag and the Opcode.
It MUST be lesser than 2^16.

If the transaction ID is elided, the value 0 is assumed, same for the flags.
The transaction ID MUST be included and set to an unpredictable value lesser than 2^32, if the DNS
transport can not ensure the prevention of DNS response spoofing.
An example for such a transport is unencrypted DoC (see {{-doc}}, Section 6).

The question section is encoded as a CBOR array containing up to 3 entries:

1. The queried name (as text string, see {{sec:domain-names}}),
2. An optional record type (as unsigned integer), and
3. An optional record class (as unsigned integer)

If the record type is elided, record type `AAAA` as specified in {{-aaaa}} is assumed.
If the record class is elided, record class `IN` as specified in {{-dns}} is assumed.
When a record class is required, the record type MUST also be provided.

The remainder of the query is either empty or MUST consist of up to two arrays.
The first array, if present, encodes the authority section of the query as an array of DNS
resource records (see {{sec:rr}})
The second array, if present, encodes the additional section of the query as an array of DNS
resource records (see {{sec:rr}})

The representation of a DNS query is defined in {{fig:dns-query}}.

~~~ CDDL
query-id-flags = (
  id: uint .default 0,
  ? flags: uint .default 0,
)
question-section = (
  name: domain-name,
  ? type-spec,
)
extra-sections = (
  ? authority: [+ dns-rr],
  additional: [+ dns-rr],
)
query-sections = (
  ? query-id-flags,
  [question-section],
  ? extra-sections,
)
dns-query = [query-sections]
~~~
{:cddl #fig:dns-query title="DNS Query Definition"}

## DNS Responses {#sec:responses}

DNS responses are encoded as a CBOR array containing up to 7 entries.

1. An optional transaction ID (as unsigned integer),
2. An optional flag field (as unsigned integer),
3. An optional question section (as array, encoded as described in {{sec:queries}})
4. The answer section (as array),
4. An optional authority section (as array), and
5. An optional additional section (as array)

If the CBOR array is a response to a query that contains a transaction ID, it MUST be included and
set to the corresponding value present in the query.
If it is not included, the transaction ID is implied to be 0.

If the CBOR array is a response to a query for which the flags indicate that flags are set in the
response, they MUST be set accordingly and thus included in the response.
If the flags are not included, the flags are implied to be 0x8000 (everything unset except for the
QR flag).

If the response includes only 1 array, this is the DNS answer section represented as an
array of one or more DNS Resource Records (see {{sec:rr}}).

If the response includes 2 arrays, the first entry is a question section and the second
entry is an answer section. The question section is encoded like as specified in {{sec:queries}},
the answer section is represented as an array of one or more DNS Resource Records (see {{sec:rr}}).

If the response includes 3 arrays, the first section is a question section, the second an answer
section, and the third an additional section (TBD: back choice to favor additional section by
empirical data). Again, the question section is encoded like a DNS query as specified in
{{sec:queries}} and both answer and additional sections are represented each as an array of one
or more DNS Resource Records (see {{sec:rr}}).

If the response includes 4 arrays, the first section is a question section, the second an answer
section, the third an authority section, and the fourth an additional section (TBD: back by
empirical data). They follow the specification of 3 arrays in the answer. The authority section is
also represented as an array of one or more DNS Resource Records (see {{sec:rr}}).

~~~ CDDL
response-id-flags = (
  id: uint .default 0,
  ? flags: uint .default 0x8000,
)
response-sections = ((
  ? response-id-flags,
  answer: [+ dns-rr],
) // (
  ? response-id-flags,
  question: [question-section],
  answer: [+ dns-rr],
  ? extra-sections,
))
dns-response = [response-sections]
~~~
{:cddl #fig:dns-response title="DNS Response Definition"}

# Name and Address Compression with Packed CBOR

If both DNS server and client support packed CBOR {{-cbor-packed}}, it MAY be used for name and
address compression in DNS responses.

## Media Type Negotiation

A DNS client uses media type "application/dns+cbor;packed=1" to negotiate (see, e.g.,
{{-http-semantics}} or {{-coap}}, Section 5.5.4) with the DNS server if the server supports packed
CBOR.
If it does, it MAY request the response to be in packed CBOR (media type
"applicaton/dns+cbor;packed=1").
The server then SHOULD reply with the response in packed CBOR.

## DNS Representation in Packed CBOR

The representation of DNS responses in packed CBOR has the same semantics as for tag 113
({{-cbor-packed}}, Section 3.1) with the rump being the compressed response.
The difference to {{-cbor-packed}} is that tag 113 to the array is OPTIONAL.

Compression of queries is not specified, as apart from EDNS options (see Section {{sec:edns}}), they
only consist of one question most of the time.

## Compression {#sec:pack-compression}

How the compressor constructs the packing table, i.e., how the compression is applied, is out of
scope of this document. Several potential compression algorithms were evaluated in \[TBD\].

<!--
Discussion TBD:

- For queries, as they are only one question, i.e. at most one value of each at most,
  compression is not necessary.
- Address and name compression are mostly about affix compression
  (i.e. straight/inverse referencing)<br>
  ==> For occasions were value is the affix (e.g., "example.org" in ANY example in
  {{sec:response-examples}}) use shared item referencing to argument table to safe bytes (no extra
  shared item table, no, e.g., 216(""), just simple(0))
  - **Example:** Using Basic Packed CBOR ({{-cbor-packed}}, section 3.1):
    - 130 bytes (Basic Packed CBOR)
    - 200 bytes (plain CBOR, see {{sec:response-examples}})
    - 194 bytes (wire-format)

    >     113(
    >       [
    >         ["_coap._udp.local", "example.org", 3600, 28],
    >         [h'20010db800000000000000000000', simple(1)],
    >         [
    >           [simple(1), 12, 1],
    >           [[simple(1), simple(0)]],
    >           [
    >             [simple(1), 2, 217("ns1.")],
    >             [simple(1), 2, 217("ns2.")]
    >           ],
    >           [
    >             [simple(0), simple(1), simple(3), 6(h'0001')],
    >             [simple(0), simple(1), simple(3), 6(h'0002')],
    >             [217("ns1."), simple(1), simple(3), 6(h'0035')],
    >             [217("ns2."), simple(1), simple(3), 6(h'3535')]
    >           ]
    >         ]
    >       ]
    >     )

    vs. application/dns+cbor;packed=1 (shared and argument table as one) 126&nbsp;bytes:

    >     [
    >       [
    >         h'20010db800000000000000000000',
    >         "_coap._udp.local", "example.org", 3600, 28
    >       ],
    >       [
    >         [simple(2), 12, 1],
    >         [[simple(3), simple(1)]],
    >         [
    >           [simple(2), 2, 218("ns1.")],
    >           [simple(2), 2, 218("ns2.")]
    >         ],
    >         [
    >           [simple(1), simple(3), simple(4), 6(h'0001')],
    >           [simple(1), simple(3), simple(4), 6(h'0002')],
    >           [218("ns1."), simple(3), simple(4), 6(h'0035')],
    >           [218("ns2."), simple(3), simple(4), 6(h'3535')]
    >         ]
    >       ]
    >     ] -->

# Comparison to wire format

TBD: Table comparing DNS wire-format, DNS+CBOR, and DNS+CBOR-packed

# Security Considerations

TODO Security


# IANA Considerations

## Media Type Registration {#media-type}

This document registers a media type for the serialization format of DNS messages in CBOR. It
follows the procedures specified in {{!RFC6838}}.

### "application/dns+cbor"

Type name: application

Subtype name: dns+cbor

Required parameters: None

Optional parameters: packed

Encoding considerations: Must be encoded as using {{-cbor}}. See \[TBD-this-spec\] for details.

Security considerations: See {{security-considerations}} of this draft

Interoperability considerations: TBD

Published specification: \[TBD-this-spec\]

Applications that use this media type: TBD DNS over X systems

Fragment Identifier Considerations: TBD

Additional information:

&nbsp;&nbsp;&nbsp;Deprecated alias names for this type: N/A

&nbsp;&nbsp;&nbsp;Magic number(s): N/A

&nbsp;&nbsp;&nbsp;File extension(s): dnsc

&nbsp;&nbsp;&nbsp;Macintosh file type code(s): none

Person & email address to contact for further information:
   Martine S. Lenders <m.lenders@fu-berlin.de>

Intended usage: COMMON

Restrictions on Usage: None?

Author: Martine S. Lenders <m.lenders@fu-berlin.de>

Change controller: Martine S. Lenders <m.lenders@fu-berlin.de>

Provisional registrations? No

## CoAP Content-Format Registration

IANA is requested to assign CoAP Content-Format ID for the new DNS message media
types in the "CoAP Content-Formats"
sub-registry, within the "CoRE Parameters" registry {{-coap}}, corresponding the
"application/dns+cbor" media type specified in {{media-type}}:

### "application/dns-cbor"

Media-Type: application/dns+cbor

Encoding: -

Id: TBD

Reference: \[TBD-this-spec\]

### "application/dns+cbor;packed=1"

Media-Type: application/dns+cbor;packed=1

Encoding: -

Id: TBD

Reference: \[TBD-this-spec\]


--- back

# Examples

## DNS Queries {#sec:query-examples}

A DNS query of the record `AAAA` in class `IN` for name "example.org" is
represented in CBOR extended diagnostic notation (EDN) (see Section 8 in
{{-cbor}} and Appendix G in {{-cddl}}) as follows:

    [["example.org"]]


A query of an `A` record for the same name is represented as

    [["example.org", 1]]

A query of `ANY` record for that name is represented as

    [["example.org", 255, 255]]

## DNS Responses {#sec:response-examples}

The responses to the examples provided in {{sec:query-examples}} are shown
below. We use the CBOR extended diagnostic notation (EDN) (see Section 8 in
{{-cbor}} and Appendix G in {{-cddl}}).

To represent an `AAAA` record with TTL 300 seconds for the IPv6 address 2001:db8::1, a minimal
response to `["example.org"]` could be

    [[[300, h'20010db8000000000000000000000001']]]

In this case, the name is derived from the query.

If the name or the context is required, the following response would also
be valid:

    [[["example.org", 300, h'20010db8000000000000000000000001']]]

If the query can not be mapped to the response for some reason, a response
would look like:

    [["example.org"], [[300, h'20010db8000000000000000000000001']]]

To represent a minimal response of an `A` record with TTL 3600 seconds for the IPv4 address
192.0.2.1, a minimal response to `["example.org", 1]` could be

    [[300, h'c0000201']]

Note that here also the 1 of record type `A` can be elided, as this record
type is specified in the question section.

Lastly, a response to `["example.org", 255, 255]` could be

    [
      ["example.org", 12, 1],
      [[3600, "_coap._udp.local"]],
      [
        [3600, 2, "ns1.example.org"],
        [3600, 2, "ns2.example.org"]
      ],
      [
        [
          "_coap._udp.local", 3600, 28,
          h'20010db8000000000000000000000001'
        ],
        [
          "_coap._udp.local", 3600, 28,
          h'20010db8000000000000000000000002'
        ],
        [
          "ns1.example.org", 3600, 28,
          h'20010db8000000000000000000000035'
        ],
        [
          "ns2.example.org", 3600, 28,
          h'20010db8000000000000000000003535'
        ]
      ]
    ]

This one advertises two local CoAP servers (identified by service name `_coap._udp.local`) at
2001:db8::1 and 2001:db8::2 and two nameservers for the example.org domain, ns1.example.org at
2001:db8::35 and ns2.example.org at 2001.db8::3535. Each of the transmitted records has a TTL of
3600 seconds.

# Change Log

Since [draft-lenders-dns-cbor-02]
---------------------------------
- Add Discussion section and note on compression

Since [draft-lenders-dns-cbor-01]
---------------------------------
- Use MIME type parameter for packed instead of own MIME type
- Update definitions to accommodate for TID and flags, as well as more sections in query
- Clarify fallback to wire-format

Since [draft-lenders-dns-cbor-00]
---------------------------------
- Add support for DNS transaction IDs
- Name and Address compression utilizing CBOR-packed
- Minor fixes to CBOR EDN and CDDL

[draft-lenders-dns-cbor-02]: https://datatracker.ietf.org/doc/html/draft-lenders-dns-cbor-02
[draft-lenders-dns-cbor-01]: https://datatracker.ietf.org/doc/html/draft-lenders-dns-cbor-01
[draft-lenders-dns-cbor-00]: https://datatracker.ietf.org/doc/html/draft-lenders-dns-cbor-00

# Acknowledgments
{:unnumbered}

TODO acknowledge.

- Carsten Bormann
