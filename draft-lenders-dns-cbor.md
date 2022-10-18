---
title: "A Concise Binary Object Representation (CBOR) of DNS Messages"
abbrev: "TODO - Abbreviation"
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
 -  fullname: Thomas C. Schmidt
    organization: HAW Hamburg
    email: t.schmidt@haw-hamburg.de
 -  fullname: Matthias Wählisch
    organization: Freie Universität Berlin
    abbrev: FU Berlin
    email: m.waehlisch@fu-berlin.de

normative:
  RFC1035: dns
  RFC3596: aaaa
  RFC7049: cbor
  RFC8610: cddl

informative:
  RFC4944: 6lowpan
  RFC6282: iphc
  RFC7228: constr-terms
  RFC8484: doh
  RFC8724: schc
  RFC8824: coap-schc
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

A DNS query is a message that queries DNS information from an upstream DNS resolver.

The term "constrained networks" is used as defined in {{-constr-terms}}.

{::boilerplate bcp14-tagged}

# CBOR Representation (application/dns+cbor)

To keep overhead minimal, a DNS message is represented as CBOR arrays.  All CBOR items used in
this specification are of definite length.  CBOR arrays that do not follow the length
definitions of this or follow-up specifications, MUST be silently ignored.  It is assumed that
DNS query and DNS response are distinguished message types and that the query can be mapped to
the response by the transport protocol of choice.  To define the representation of binary
objects we use the Concise Data Definition Language (CDDL) {{-cddl}}.

## Domain Name Representation {#sec:domain-names}

Domain names are represented in their commonly known string format (e.g., "example.org", see Section
2.3.1 in {{-dns}}) and in IDNA encoding {{!RFC5890}} as a text string. For the purpose of this
document, domain names remain case-insensitive as specified in {{-dns}}.

The representation of a domain name is defined in {{fig:domain-name}}.

{:cddl: artwork-align="center"}

~~~ CDDL
domain-name = tstr .regexp "([^\.]+\.)*[^\.]+"
~~~
{:cddl #fig:domain-name title="Domain Name Definition"}

## DNS Queries {#sec:queries}

DNS queries are encoded as CBOR arrays containing up to 3 entries in the following order:  The
name (as text string, see {{sec:domain-names}}), an optional record type (as unsigned integer),
and an optional record class (as unsigned integer).

If the record type is elided, record type `AAAA` as specified in {{-aaaa}} is assumed.
If the record class is elided, record class `IN` as specified in {{-dns}} is assumed.
If a record class is required, the record type MUST also be provided.

The representation of a DNS query is defined in {{fig:dns-query}}.

~~~ CDDLx
type-spec = (
  record-type: uint,
  ? record-class: uint,
)
dns-question = (
  name: domain-name,
  ? type: type-spec,
)
dns-query = [dns-question]
~~~
{:cddl #fig:dns-query title="DNS Query Definition"}

## Standard DNS Resource Records (RRs) {#sec:rr}

DNS resource records are encoded as CBOR arrays containing 2 to 5 entries in the following
order: An optional name (as text string, see {{sec:domain-names}}), a TTL (as unsigned
integer), an optional record type (as unsigned integer), an optional record class (as unsigned
integer), and lastly a record data entry (as byte string or text string).

If the first element of the resource record is a string, the first element is a name.  If the
name is elided, the name from the query, either derived from transport context or the provided
question section, see {{sec:responses}}, is assumed.  If the record type is elided, the record
type from the question is assumed. If record class is elided, the record class from the
question is assumed. If a record class is required, the record type MUST also be provided.

The byte format of the record data follows the wire format as specified in Section 3.3 {{-dns}}
(or other specifications of the respective record type).  Note that this format does not
include the RDLENGTH field from {{-dns}} as this value is encoded in the length field of the
CBOR byte string.

If and only if the record data represents a domain name (e.g., for CNAME or PTR records), the
record data MAY be represented as a text string as specified in {{sec:domain-names}}.  This can
save 2 bytes of data, because the byte representation of DNS names requires both an additional
byte to define the length of the first name component as well as a 0 byte at the end of the
name.

The representation of a DNS resource records is defined in {{fig:dns-rr}}.

~~~ CDDL
rr = (
  ? name: domain-name,
  ttl: uint,
  ? type: record-type,
  rdata: bstr / domain-name,
)
dns-rr = [rr]
~~~
{:cddl #fig:dns-rr title="DNS Resource Record Definition"}

## DNS Responses {#sec:responses}

DNS responses are encoded as array of CBOR arrays containing up to 4 entries.

If only 1 array is included, then this is the DNS answer section represented as an array of one
or more DNS Resource Records (see {{sec:rr}}).

If 2 arrays are included, then the first entry is a question section and the second entry is
an answer section. The question section is encoded like a DNS query as specified in
{{sec:queries}}, the answer section is represented as an array of one or more DNS Resource
Records (see {{sec:rr}}).

If 3 arrays are included, then the first section is a question section, the second an answer
section, and the third an additional section (TBD: back choice to favor additional section by
empirical data). Again, the question section is encoded like a DNS query as specified in
{{sec:queries}} and both answer and additional sections are represented each as an array of one
or more DNS Resource Records (see {{sec:rr}}).

If 4 arrays are included, then the first section is a question section, the second an answer
section, the third an authority section, and the fourth an additional section (TBD: back by
empirical data). They follow the specification of 3 arrays in the answer. The additional
authority section is also represented as an array of one or more DNS Resource Records (see
{{sec:rr}}).

~~~ CDDL
extra-sections = (
  ? authority: [1* dns-rr],
  additional: [1* dns-rr],
)
sections = (
  ? question: dns-query,
  answer: [1* dns-rr],
  ? extra: extra-sections,
)
dns-response = [sections]
~~~
{:cddl #fig:dns-response title="DNS Response Definition"}

## EDNS(0)

TBD, do we need special formatting here?

# Security Considerations

TODO Security


# IANA Considerations

## Media Type Registration {#media-type}

This document registers the media type for the serialization format of DNS messages in CBOR. It
follows the procedures specified in {{!RFC6838}}.

Type name: application

Subtype name: dns+cbor

Required parameters: None

Optional parameters: None

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

IANA is requested to assign CoAP Content-Format ID for the DNS message media
type in the "CoAP Content-Formats" sub-registry, within the "CoRE Parameters"
registry {{!RFC7252}}, corresponding the "application/dns+cbor" Media Type specified in
{{media-type}}:

Media-Type: application/dns+cbor

Encoding: -

Id: TBD

Reference: \[TBD-this-spec\]

--- back

# Examples

## DNS Queries {#sec:query-examples}

A DNS query for the `AAAA`/`IN` record of name "example.org" is represented as the following in CBOR
extended diagnostic notation (EDN) as defined in Section 8 of {{-cbor}} and Appendix G of {{-cddl}}:

    ["example.org"]


Likewise, the `A` record for the same name is represented as

    ["example.org", 1]

A query for ANY record for that name is represented as

    ["example.org", 255, 255]

## DNS Responses {#sec:response-examples}

The responses to the examples provided in {{sec:query-examples}} in EDN as defined in Section 8 of
{{-cbor}} and Appendix G of {{-cddl}} can be seen below.

To represent an `AAAA` record with TTL 300 seconds for the IPv6 address 2001:db8::1, a minimal
response to `["example.org"]` could be

    [[[300, h'20010db8000000000000000000000001']]]

The name is implied from the query in that case.

However, the following responses would also be a legal response, if e.g. the name or the context is
required

    [[["example.org", 300, h'20010db8000000000000000000000001']]]

or the query can not be mapped to the response for some reason

    [["example.org"], [[300, h'20010db8000000000000000000000001']]]

To represent a minimal response for an `A` record with TTL 3600 seconds for the IPv4 address
192.0.2.1, a minimal response to `["example.org", 1]` could be

    [[300, h'c0000201']]

Mind that here also the 1 for record type `IN` can be elided, as it is specified in the question.

Lastly, a response to `["example.org", 255, 255]` could be

    [
      ["example.org", 255, 255],
      [[3600, 12, "coap._udp.local"]],
      [
        [3600, 2, "ns1.example.org"],
        [3600, 2, "ns2.example.org"],
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

(TBD: The encoding of PTR and NS record data might be wrong...)

# Acknowledgments
{:numbered="false"}

TODO acknowledge.


