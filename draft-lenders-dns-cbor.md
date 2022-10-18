---
title: "A Compressed CBOR Representation Format for DNS Messages"
abbrev: "TODO - Abbreviation"
category: std

docname: draft-lenders-dns-cbor-latest
number:
date:
consensus: true
v: 3
submissiontype: IETF
area: "Applications"
workgroup: TBD
keyword:
 - Internet-Draft
 - CBOR
 - DNS
venue:
  group: TBD
  type: Working Group
  mail: TBD@example.com
  arch: "nicfs.nic.ddn.mil:~/namedroppers/*.Z"
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

informative:


--- abstract

TODO Abstract


--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# CBOR Representation (application/dns+cbor)

To keep overhead minimal, a DNS message is to be represented as CBOR arrays.
All CBOR items used in this specification are of definite length.
CBOR arrays that do not follow the length definitions below or in follow-up specifications, MUST be
silently ignored.
It is assumed, that query and response are distinguished message types for the transport protocol
and that the query can be mapped to the response by the transport protocol of choice.

## Domain Name Representation {#sec:domain-names}

Domain names are represented in their commonly known string format (e.g. "example.org") as a text
string.

## DNS Queries {#sec:queries}

DNS queries are encoded as CBOR arrays containing up to 3 entries.
They contain in the following order the name (as text string, see {{sec:domain-names}}), an optional
record type (as unsigned integer), and an optional record class (as unsigned integer).
If the record type is elided, record type `AAAA` as specified in {{!RFC3596}} is implied.
If record class is elided, record class `IN` as specified in {{!RFC1035}} is implied.
If a record class is required to be provided, the record type MUST also be provided.

### Examples {#sec:query-examples}
A DNS query for the `AAAA`/`IN` record of name "example.org" is represented as the following
in CBOR diagnostic notation (see {{!RFC7049}}, section 6):

    ["example.org"]


Likewise, the `A` record for the same name is represented as

    ["example.org", 1]

A query for ANY record for that name is represented as

    ["example.org", 255, 255]


## Standard DNS Resource Records (RRs) {#sec:rr}

DNS records are, like DNS queries, encoded as CBOR arrays with up to 5 entries, but of length 2 at
minimum.
They contain in the following order an optional name (as text string, see {{sec:domain-names}}), a
TTL (as unsigned integer), an optional record type (as unsigned integer), an optional record class
(as unsigned integer), and lastly a record data entry (as byte string or text string).

The presence of the optional name can be determined by the first element of the resource record
being a string.
If the name is elided, the name from the query, either from transport context or the provided
question section, see {{sec:responses}} below, is implied.
If the record type is elided, record type from the question is implied. If
record class is elided, record class from the question is implied. If a record class
is required to be provided, the record type MUST also be provided.

The byte format of the record data follows the wire format as specified {{!RFC1035}}, section 3.3
(or other specifications of the respective record type). Mind that this specifically does not
include the RDLENGTH field from {{!RFC1035}} as this value is encoded in the length field of the
CBOR byte string.

If and only if the record data represents a domain name (e.g., for CNAME or PTR records), the record
data MAY be represented as a text string as specified in {sec:domain-names}.
This can actually save us 2 bytes of data, as the byte representation of DNS names requires both an
additional byte to define the length of the first name component, as well as a 0 byte at the end of
the name.

## DNS Responses {#sec:responses}

DNS responses are encoded of CBOR arrays of up to 4 CBOR arrays.

If only 1 array is contained then this is the answer section represented as an array of one or
more DNS Resource Record (see {{sec:rr}}).

2 arrays are a question section and an answer section: The question section is encoded like a DNS
query as specified in {{sec:queries}}, the answer section are represented as an array of one or more
DNS Resource Records (see {{sec:rr}})

3 arrays are a question section, an answer section, and an additional section (TBD: back choice to
favor additional section by empirical data). Again, the question section is encoded like a DNS query
as specified in {{sec:queries}} and both answer and additional section are represented each as an
array of one or more DNS Resource Records (see {{sec:rr}}).

Lastly, 4 arrays are a question section, an answer section, an authority section, and an additional
section (TBD: back by empirical data). They follow the specification for 3 arrays in the answer:
the question section is encoded like a DNS query as specified in {{sec:queries}} and answer,
authority, and additional section are represented each as an array of one or more DNS Resource
Records (see {{sec:rr}}).

### Examples
The responses to the examples provided in {{sec:query-examples}} in CBOR diagnostic notation (see
{{!RFC7049}}, section 6) can be seen below.

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

(TBD: I think the encoding for PTR and NS record data is wrong...)

## EDNS(0)

TBD, do we need special formatting here?

## ABNF

TBD? But there is no ABNF definition for CBOR...

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

Encoding considerations: Must be encoded as using {{!RFC7049}}. See \[TBD-this-spec\] for details.

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

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
