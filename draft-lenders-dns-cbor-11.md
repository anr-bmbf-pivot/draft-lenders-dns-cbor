---
title: "A Concise Binary Object Representation (CBOR) of DNS Messages"
abbrev: "dns+cbor"
category: std

docname: draft-lenders-dns-cbor-11
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
    org: TUD Dresden University of Technology
    abbrev: TU Dresden
    street: Helmholtzstr. 10
    city: Dresden
    code: D-01069
    country: Germany
    email: martine.lenders@tu-dresden.de
 -  name: Carsten Bormann
    org: Universität Bremen TZI
    street: Postfach 330440
    city: Bremen
    code: D-28359
    country: Germany
    phone: +49-421-218-63921
    email: cabo@tzi.org
 -  fullname: Thomas C. Schmidt
    organization: HAW Hamburg
    email: t.schmidt@haw-hamburg.de
 -  name: Matthias Wählisch
    org: TUD Dresden University of Technology & Barkhausen Institut
    abbrev: TU Dresden & Barkhausen Institut
    street: Helmholtzstr. 10
    city: Dresden
    code: D-01069
    country: Germany
    email: m.waehlisch@tu-dresden.de

normative:
  RFC1035: dns
  RFC3596: aaaa
  RFC5891: idna
  RFC6891: edns
  RFC7252: coap
  RFC8610: cddl
  RFC8949: cbor
  RFC9460: svcb
  I-D.ietf-cbor-packed: cbor-packed
  I-D.ietf-cbor-edn-literals: edn
  IANA.cbor-tags: tags

informative:
  RFC4944: 6lowpan
  RFC6282: iphc
  RFC7228: constr-terms
  RFC8484: doh
  RFC8499: dns-terms
  RFC8618: cdns
  RFC8724: schc
  RFC8824: coap-schc
  RFC9110: http-semantics
  RFC9619: qdcount-1
  I-D.ietf-core-dns-over-coap: doc


--- abstract

This document specifies a compressed data format of DNS messages using
the Concise Binary Object Representation {{-cbor}}.
The primary purpose is to keep DNS messages small in constrained networks.


--- middle

# Introduction

In constrained networks {{-constr-terms}}, the link layer may restrict the payload sizes of frames to
only a few hundreds bytes.  Encrypted DNS resolution, such as DNS over HTTPS (DoH) {{-doh}} or
DNS over CoAP (DoC) {{-doc}}, may lead to DNS message sizes that exceed this limit, even when
implementing header compression such as 6LoWPAN IPHC {{-iphc}} or SCHC {{-schc}},
{{-coap-schc}}.

Although adoption layers such as 6LoWPAN {{-6lowpan}} or SCHC {{-schc}} offer fragmentation to
comply with small MTUs, fragmentation should be avoided in constrained networks.
Fragmentation combined with high packet loss multiplies the likelihood of loss.
Hence, a compression format that reduces fragmentation of DNS messages is beneficial.

This document specifies a compressed data format for DNS messages using Concise Binary Object Representation (CBOR) {{-cbor}} encoding. Additionally,  unnecessary or redundant information are stripped off DNS messages.  To use the outcome of this specification in DoH and DoC,
this document also specifies a Media Type header for DoH and a Content-Format option for DoC.

Note, that there is another format that expresses DNS messages in CBOR, C-DNS {{-cdns}}.
C-DNS is primarily a file format to minimize traces of multiple DNS messages and uses the fact that there are multiple messages to do its compression.
Common values such as names or addresses are collected in separate tables which are referenced from the messages, comparable to CBOR-packed {{-cbor-packed}}.
However, this may add overhead for individual DNS messages.

The format described in this document is a transfer format that aims to provide conciseness and compression for individual DNS messages to be sent over the network.
This is achieved applying the following objectives:

1. Encoding DNS messages in CBOR (conciseness),
2. Omitting (redundant) fields in DNS queries and responses (conciseness),
3. Providing easy to implement name compression that allows for on-the-fly construction of DNS queries and responses (compression), and
4. Providing optional address and value compression in DNS responses using CBOR-packed {{-cbor-packed}} (compression).

# Terminology

CBOR types (unsigned integer, byte string, text string, arrays, etc.) are used as defined in
{{-cbor}}.

The terms "DNS server", "DNS client", and "(DNS) resolver" are used as defined in {{-dns-terms}}.

A DNS query is a message that queries DNS information from an upstream DNS resolver.
The reply to that is a DNS response.

The DNS message format specified in {{-dns}} for DNS over UDP we call "classic DNS format" throughout this document or refer to it by its media type "application/dns-message" as specified in {{-doh}}.

The term "constrained networks" is used as defined in {{-constr-terms}}.

{::boilerplate bcp14-tagged}

# CBOR Representations (application/dns+cbor)

DNS messages are represented as CBOR arrays to minimize overhead.
All CBOR items used in this specification are of definite length.
CBOR arrays that do not follow the length definitions of this or of follow-up specifications, MUST be silently ignored.
CBOR arrays that exceed the message size provided by the transport, MUST be silently ignored.
It is assumed that DNS query and DNS response are distinguished message types and that the query can be mapped to the response by the transfer protocol of choice.
To define the representation of binary objects we use the Concise Data Definition Language (CDDL) {{-cddl}}.
For examples, we use the CBOR Extended Diagnostic Notation {{-edn}}.

~~~ cddl
dns-message = dns-query / dns-response
~~~
{:cddl #fig:dns-msg title="This document defines both DNS Queries and Responses in CDDL"}

If, for any reason, a DNS message cannot be represented in the CBOR format specified in this document, or if unreasonable overehead is introduced, a fallback to another DNS message format, e.g., the classic DNS format specified in {{-dns}}, MUST always be possible.

## Domain Name Representation {#sec:domain-names}

Domain names are represented by a sequence of one or more (unicode) text strings.
For instance, "example.org" would be represented as `"example","org"` in CBOR diagnostic notation.
The root domain "." is represented as an empty string `""`.
The absence of any label or tag TBDt (see {{sec:name-compression}} below) means the name is elided.
For the purpose of this document, domain names remain case-insensitive as specified in {{-dns}}.

The representation of a domain name is defined in {{fig:domain-name}}.
A label may either be encoded in ASCII-compatible encoding (ACE) {{-idna}} embedded within UTF-8 encoding of the text strings or plain UTF-8.
It is RECOMMENDED to use the encoding with the shorter length in bytes.
A decoder MAY identify the ACE encoding by identifying the label as a valid A-label (see {{-idna}}) and MUST assume the label to be encoded in UTF-8 otherwise.

This sequence of text strings is supposed to be embedded into a surrounding array, usually the query
or resource record.

### Name Compression {#sec:name-compression}

{:cddl: sourcecode-name="dns-cbor.cddl"}

~~~ cddl
domain-name = (
  * label,
  ? ( #6.TBDt(uint) / label ),
)
label = tstr
~~~
{:cddl #fig:domain-name title="Domain Name Definition"}

Names are compressed by pointing to existing labels in the message.
CBOR objects are typically decoded depth-first.
Whenever we encounter a label we take the value of a counter _c_ as the position of that label.
The counter _c_ is then increased.

A tag TBDt may follow any sequence of labels, even an empty sequence.
This tag TBDt encapsulates an unsigned integer _i_ which points to a label at position _i_.
_i_ MUST be smaller than _c_.
A name then is decoded as any label that then preceded tag TBDt(_i_) and all labels including and following at position _i_ are appended.
This includes any further occurrence of tag TBDt after the referenced label sequence, though the decoding stops after this tag was recursively decoded.
Note, that this also may include simple values or tags that reference the packing table with CBOR-packed (see {{sec:cbor-packed}}).

For instance, the name "www.example.org" can be encountered twice in the example in
{{fig:name-compression-example}} (notated in CBOR Extended Diagnostic Notation, see {{-edn}}).

~~~ cbor-diag
[
  # AAAA (28, elided) question for "example.org"
  [ "example" / c == 0 /, "org" / c == 1 / ],
  # Answer section:
  [
    # "example.org" (elided) CNAME (5) is "www.example.org"
    [ 5, "www" / c == 2 /, TBDt(0) / references c == 0 / ],
    # "www.example.org" AAAA (28, elided) is 2001:db8::1
    [
      TBDt(2) / references c == 2 /,
      h'20010db8000000000000000000000001'
    ]
  ]
]
~~~
{: #fig:name-compression-example title="Example for name compression." }

<!--
For name compression, a tag TBDt encapsulating an unsigned integer _i_ can be appended to the sequence of text strings.
To extend the name suffix, the unsigned integer _i_ points to the _i_-th text string (counted depth first) in the overall DNS message.
That string and all text strings or another tag TBDt following the string at the _i_-th position are appended to the name sequence.
If another tag TBDt is encountered, it is resolved in the same way.
Strings following a tag TBDt MUST NOT be appended to the name sequence.
To prevent circular references, this DNS name suffix extension algorithm should error whenever a string position is encountered more than once during the extension of a name.
Likewise, the algorithm should error whenever the _i_ is greater than the position of the previous seen string from this occurrence of tag TBDt.
Only backward referencing is allowed for tag TBDt.
Decompression stops when any other type than a text string or any other tag than tag TBDt are
encountered.
-->

The pseudo-code for this DNS name suffix extension algorithm can be seen in {{fig:decode-name}}.

~~~
function decode_name(obj: cbor_obj, cbor_ptr: cbor_major_type): list
{
  name: list = []
  visited: set = {}
  while (typeof(cbor_ptr) in {tstr, tag}):
    if typeof(cbor_ptr) == tag:
      if cbor_ptr.tag != TBDt:
        break
      i: uint = cbor_ptr.value
      if i-th text string after (depth first) cbor_ptr:
        return ERROR("Forward reference not allowed")
      cbor_ptr =
        jump to i-th text string (depth first) in obj
      if cbor_ptr in visited:
        return ERROR("Circular reference")
    # cbor_ptr should be of type tstr at this point
    name.append(cbor_ptr)
    visited.add(cbor_ptr)
  return name
}
~~~
{: #fig:decode-name title="Name Suffix Extension Algorithm"}

The tag TBDt is included in the definition in {{fig:domain-name}}.

## DNS Resource Records {#sec:rr}

{:mlenders: source="mlenders"}

This document specifies the representation of both standard DNS resource records (RRs, see {{-dns}})
and EDNS option pseudo-RRs (see {{-edns}}.[^1]{:mlenders}
If for any reason, a resource record cannot be represented in the given formats, they can be
represented in their binary wire-format form as a byte string.

Further special records, e.g., TSIG can be defined in follow-up specifications and are out of scope
of this document.

The representation of a DNS resource records is defined in {{fig:dns-rr}}.

[^1]: Also add capability to summarize Resource Record Sets to one array, e.g. `["example","org",3600,1,[b'c0002563', h'c00021ab']]`?

~~~ cddl
$$dns-rr = rr / #6.141(opt-rr) / bstr
~~~
{:cddl #fig:dns-rr title="DNS Resource Record Definition"}

### Standard RRs

Standard DNS resource records are encoded as CBOR arrays containing 2 or more entries in the following order:

1. An optional name (as text string, see {{sec:domain-names}}),
2. A TTL (as unsigned integer),
3. An optional record type (as unsigned integer),
4. An optional record class (as unsigned integer), and lastly
5. A record data entry (as byte string, domain name, or array for dedicated record data representation).

If the first item of the resource record is a text string, it is the first label of a domain name (see {{sec:domain-names}}).
If the name is elided, the name is derived from the question section of the message.
For responses, the question section is either taken from the query (see {{sec:queries}}) or provided with the response see {{sec:responses}}.
The query may be derived from the context of the transfer protocol.

If the record type is elided, the record type from the question is assumed.
If record class is elided, the record class from the question is assumed.
When a record class is required to be expressed, the record type MUST also be provided.

The byte string format of the record data as a byte string follows the classic DNS format as specified in {{Section 3.3 of -dns}} (or other specifications of the respective record type).
Note that the CBOR format does not include the RDLENGTH field from the classic format as this value is encoded in the length field of the CBOR header of the byte string.

If the record data represents a domain name (e.g., for CNAME or PTR records), the record data MAY be represented as domain name as specified in {{sec:domain-names}}.
This can save 1 byte of data, as the zero byte at the end of the name is not necessary with the CBOR format.
Only 1 byte is required to define type and length of each text string representing a label up until a string length of 23 characters, amortizing to the same remaining length as in the name representation in the classic format.
This way of representing the record data also means that name compression (see {{sec:name-compression}}) can also be used on it.

Depending on the record type, the record data may also be expressed as an array.
Some initial array types are specified below.
Future specifications can extend the definition for `$rdata-array` in {{fig:dns-standard-rr}}.
These extensions mainly serve to expose names to name compression (see {{sec:name-compression}}).
There is an argument to be made for CBOR-structured formats of other record data representations (e.g. DNSKEY or RRSIG), but structuring such records as an array usually adds more overhead than just transferring the byte representation.
As such, structured record data that do not contain names are always to be represented as a byte string.

~~~ cddl
max-uint8 = 0..255
max-uint16 = 0..65535
max-uint32 = 0..4294967295
ttl = max-uint32
rr = [
  ? domain-name,
  ttl: ttl,
  type-spec-rdata,
]
type-spec-rdata = (
  ? type-spec,
  rdata: bstr // ( domain-name ),
)
type-spec-rdata //= ( $$structured-ts-rd )
type-spec = (
  record-type: max-uint16,
  ? record-class: max-uint16,
)
~~~
{:cddl #fig:dns-standard-rr title="DNS Standard Resource Record Definition"}

#### SOA Record Data

The record data of RRs with `record-type` = 6 (SOA) MAY be expressed as an array with at least 7 entries representing the 7 parts of the SOA resource record defined in {{-dns}} in the following order:

- MNAME as a domain name (see {{sec:domain-names}}),
- SERIAL as an unsigned integer,
- REFRESH as an unsigned integer,
- RETRY as an unsigned integer,
- EXPIRE as an unsigned integer,
- MINIMUM as an unsigned integer, and
- RNAME as a domain name (see {{sec:domain-names}}).

MNAME and RNAME are put to the beginning and end of the array, respectively, to keep their labels apart.

The definition for MX record data can be seen in {{fig:dns-rdata-soa}}.

~~~ cddl
$$structured-ts-rd //= (
  6,    ; record-type = SOA
  ? 1,  ; record-class = IN
  soa,
)

soa = [
  domain-name,  ; mname
  serial: max-uint32,
  refresh: max-uint32,
  retry: max-uint32,
  expire: max-uint32,
  minimum: max-uint32,
  domain-name,  ; rname
]
~~~
{:cddl #fig:dns-rdata-soa title="SOA Resource Record Data Definition"}

#### MX Record Data

The record data of RRs with `record-type` = 15 (MX) MAY be expressed as an array with at least 2 entries representing the 2 parts of the MX resource record defined in {{-dns}} in the following order:

- PREFERENCE as an unsigned integer and
- EXCHANGE as a domain name (see {{sec:domain-names}}).

The definition for MX record data can be seen in {{fig:dns-rdata-mx}}.

~~~ cddl
$$structured-ts-rd //= (
  15,   ; record-type = MX
  ? 1,  ; record-class = IN
  mx,
)

mx = [
  preference: max-uint16,
  domain-name,  ; exchange
]
~~~
{:cddl #fig:dns-rdata-mx title="MX Resource Record Data Definition"}

#### SRV Record Data

The record data of RRs with `record-type` = 33 (SRV) MAY be expressed as an array with at least 3 entries representing the parts of the SRV resource record defined in {{!RFC2782}} in the following order:

- Priority as an unsigned integer,
- an optional Weight as an unsigned integer,
- Port as an unsigned integer,
- Target as a domain name (see {{sec:domain-names}}).

If the weight is present or not can be determined by the number of unsigned integers before Target.
2 unsigned integers before the Target mean the weight was elided and defaults to 0.
3 unsigned integers before the Target mean the weight is in the second position of the record data array.
The default of 0 was picked, as this is the value domain administrators should pick when there is no server selection to do {{!RFC2782}}.

The definition for SRV record data can be seen in {{fig:dns-rdata-srv}}.

~~~ cddl
$$structured-ts-rd //= (
  33,   ; record-type = SRV
  ? 1,  ; record-class = IN
  srv,
)

srv = [
  priority: max-uint16,
  ? weight: max-uint16 .default 0,
  port: max-uint16,
  domain-name,  ; target
]
~~~
{:cddl #fig:dns-rdata-srv title="SRV Resource Record Data Definition"}

#### SVCB and HTTPS Record Data

The record data of RRs with `record-type` = 64 (SVCB) and `record-type` = 65 (HTTPS) MAY be expressed as an array with at least 3 entries representing the 3 parts of the SVCB/HTTPS resource record defined in {{-svcb}} in the following order:

- An optional SvcPriority as an unsigned integer,
- An optional TargetName as a domain name (see {{sec:domain-names}}), and
- SvcParams as an array of alternating pairs of SvcParamKey (as unsigned integer) and SvcParamValue
  (as byte string).
  The type of SvcParamValue may be extended in future specifications.

If the SvcPriority is present can be determined by checking if the record data array starts with an unsigned integer or not.
If the array does not start with an unsigned integer, the SvcPriority is elided and defaults to 0, i.e., the record is in AliasMode (see {{Section 2.4.2 of -svcb}}).
If the array starts with a unsigned integer, it is the SvcPriority.

If the TargetName is present can be determined by checking if the record data array has a text string or tag TBDt after the SvcPriority, i.e., if the SvcPriority is elided the array would start with a text string or tag TBDt.
If there is no text string or tag TBDt after the SvcPriority, the TargetName is elided and defaults to the sequence of text strings `""` (i.e. the root domain "." in the common name representation defined in {{Section 2.3.1 of -dns}}, see {{sec:domain-names}}) and {{Section 2.5 of -svcb}}.
If there is a text string or tag TBDt after the SvcPriority, the TargetName is not elided and in the domain name form specified in {{sec:domain-names}}.

The definition for SVCB and HTTPS record data can be seen in {{fig:dns-rdata-svcb}}.

~~~ cddl
$$structured-ts-rd //= (
  64 / 65,  ; record-type = SVCB or HTTPS
  ? 1,      ; record-class = IN
  svcb,
)

svcb = [
  ? svc-priority: max-uint16 .default 0,
  ? domain-name,  ; target name
  svc-params: [ *svc-param-pair ],
]

svc-param-pair = (
  svc-param-key: max-uint16,
  svc-param-value: $$svc-param-value,
)
$$svc-param-value = bstr
~~~
{:cddl #fig:dns-rdata-svcb title="SVCB and HTTPS Resource Record Data Definition"}

The SvcParams are provided as an array rather than a map, as their order needs to be preserved {{-svcb}} which can not be guaranteed for maps.

### EDNS OPT Pseudo-RRs {#sec:edns}

EDNS OPT Pseudo-RRs are represented as a CBOR array.
To distinguish them from normal standard RRs, they are marked with tag TBD141.

Name and record type can be elided as they are always "." and OPT (41), respectively {{-edns}}.

The UDP payload size may be the first element as an unsigned integer in the array.
It MUST be elided if its value is the default value of 512, the maximum allowable size for unextended DNS over UDP (see {{Sections 2.3.4 and 4.2.1 of -dns}}).

The next element is a map of the options, with the option code (unsigned integer) as key and the option data (byte string) as value.
The type of option data may be extended in future specifications.

After that, up to three unsigned integers are following.
The first being the extended flags as unsigned integer (implied to be 0 if elided),
the second the extended RCODE as an unsigned integer (implied to be 0 if elided), and
the third the EDNS version (implied to be 0 if elided).
They are dependent on each of their previous elements.
If the EDNS version is not elided, both extended flags and extended RCODE MUST not be elided.
If the RCODE is not elided the extended flags MUST not be elided.

Note that future EDNS versions may require a different format than the one described above.

~~~ cddl
opt-rr = [
  ? udp-payload-size: max-uint16 .default 512,
  options: {* ocode => $$odata },
  ? opt-rcode-v-flags,
]
ocode = max-uint16
opt-rcode-v-flags = (
  flags: max-uint16 .default 0,
  ? opt-rcode-v,
)
rcode = 0..4095
opt-rcode-v = (
  rcode: rcode .default 0,
  ? version: max-uint8 .default 0,
)
$$odata = bstr
~~~
{:cddl #fig:dns-opt-rr title="DNS OPT Resource Record Definition"}

## DNS Queries {#sec:queries}

DNS queries are encoded as CBOR arrays containing up to 6 entries in the following order:

1. An optional boolean field,
2. An optional flag field (as unsigned integer),
3. The question section (as array),
4. An optional answer section (as array),
5. An optional authority section (as array), and
6. An optional additional section (as array)

If the first item is a boolean and when true, it tells the responding resolver that it MUST include the question section in its response. If that boolean is not present, it is assumed to be false.

If the first item of the query is an array, it is the question section, if it is an unsigned integer, it is as flag field and maps to the header flags in {{-dns}} and the "DNS Header Flags" IANA registry including the QR flag and the Opcode.

If the flags are elided, the value 0 is assumed.

This specification assumes that the DNS messages are sent over a transfer protocol that can map the queries to their responses, e.g., DNS over HTTPS {{-doh}} or DNS over CoAP {{-doc}}.
As a consequence, the DNS transaction ID is always elided and the value 0 is assumed.

A question record within the question section is encoded as a CBOR array containing the following entries:

1. The queried name (as domain name, see {{sec:domain-names}}) which MUST not be elided,
2. An optional record type (as unsigned integer), and
3. An optional record class (as unsigned integer)

If the record type is elided, record type `AAAA` as specified in {{-aaaa}} is assumed.
If the record class is elided, record class `IN` as specified in {{-dns}} is assumed.
When a record class is required, the record type MUST also be provided.

There usually is only one question record {{-qdcount-1}}, which is why the question section is a flat array and not nested like the other sections.
This serves to safe overhead from the additional CBOR array header.
In the rare cases when there is more than one question record in the question section, the next question just follows.
In this case, for every question but the last, the record type MUST be included, i.e., it is not optional.
This way it is ensured that the parser can distinguish each question by looking up the name first.

The remainder of the query is either empty or MUST consist of up to three extra arrays.

If one extra array is in the query, it encodes the additional section of the query as an array of DNS resource records (see {{sec:rr}}).
If two extra arrays are in the query, they encode, in that order, the authority and additional sections of the query each as an array of DNS resource records (see {{sec:rr}}).
If three extra arrays are in the query, they encode, in that order, the answer section, the authority, and additional sections of the query each as an array of DNS resource records (see {{sec:rr}}).

As such, the highest precedence in elision is given to the answer section, as it only occurs with mDNS to signify Known Answers {{?RFC6762}}.
The lowest precedence is given to the additional section, as it may contain EDNS OPT Pseudo-RRs, which are common in queries (see {{sec:edns}}).

The representation of a DNS query is defined in {{fig:dns-query}}.

~~~ cddl
dns-query = [
  ? incl-question: bool .default false,
  ? flags: max-uint16 .default 0x0000,
  question-section,
  ? query-extra-sections,
]
question-section = [
  * full-question,
  ? last-question,
]
full-question = (
  domain-name,
  type-spec,
)
last-question = (
  domain-name,
  ? type-spec,
)
query-extra-sections = (
  ? answer-section,
  extra-sections,
)
answer-section = [+ $$dns-rr]
extra-sections = (
  ? authority: [+ $$dns-rr],
  additional: [+ $$dns-rr],
)
~~~
{:cddl #fig:dns-query title="DNS Query Definition"}

## DNS Responses {#sec:responses}

A DNS response is encoded as a CBOR array containing up to 5 entries.

1. An optional flag field (as unsigned integer),
2. An optional question section (as array, encoded as described in {{sec:queries}})
3. The answer section (as array),
4. An optional authority section (as array), and
5. An optional additional section (as array)

As for queries, the DNS transaction ID is elided and implied to be 0.

If the CBOR array is a response to a query for which the flags indicate that flags are set in the
response, they MUST be set accordingly and thus included in the response.
If the flags are not included, the flags are implied to be 0x8000 (everything unset except for the
QR flag).

If the response includes only one array, then the DNS answer section represents an
array of one or more DNS Resource Records (see {{sec:rr}}).

If the response includes more than 2 arrays, the first entry may be the question section, identified
by not being an array of arrays. If it is present, it is followed by the answer section. The
question section is encoded as specified in {{sec:queries}}.

If the answer section is followed by one extra array, this array is the additional section.
Like the answer section, the additional section is represented as an array of one or more DNS Resource Records (see {{sec:rr}}).

If the answer section is followed by two extra arrays, the first is the authority section, and the second is the additional section.
The authority section is also represented as an array of one or more DNS Resource Records (see
{{sec:rr}}).

The authority section is given precedence in elision over the additional section, as due to EDNS options or, e.g., CNAME answers that also provide the A/AAAA records. The additional section tends to show up more often than the authority section.

~~~ cddl
dns-response = [
  ? flags: max-uint16 .default 0x8000,
  ? question-section,
  answer-section,
  ? extra-sections,
]
~~~
{:cddl #fig:dns-response title="DNS Response Definition"}

# Further Compression with CBOR-packed {#sec:cbor-packed}

If both DNS server and client support CBOR-packed {{-cbor-packed}}, it MAY be used for further
compression in DNS responses.
Especially IPv6 addresses, e.g., in AAAA resource records can benefit from straight referencing to
compress common address prefixes.

## Media Type Negotiation

A DNS client uses the media type "application/dns+cbor;packed=1" to negotiate (see, e.g.,
{{-http-semantics}} or {{-coap}}, Section 5.5.4) with the DNS server whether the server supports packed
CBOR.
If it does, it MAY request the response to be in CBOR-packed (media type
"application/dns+cbor;packed=1").
The server then SHOULD reply with the response in CBOR-packed, which it also signals with media type
"application/dns+cbor;packed=1".

## DNS Representation in CBOR-packed

The representation of DNS responses in CBOR-packed has the same semantics as for tag TBD113
({{-cbor-packed}}, Section 3.1) with the rump being the compressed response.
The difference to {{-cbor-packed}} is that tag TBD113 is OPTIONAL.

Packed compression of queries is not specified, as apart from EDNS(0) (see {{sec:edns}}), they only
consist of one question most of the time, i.e., there is close to no redundancy.

## Compression {#sec:pack-compression}

The method of the compressor to construct the packing table, i.e., how the compression is applied, is out of scope of this document. Several potential compression algorithms were evaluated in \[TBD\].

<!--
Discussion TBD:

- For queries, as they are only one question, i.e. at most one value of each at most,
  compression is not necessary.
- Address and name compression are mostly about affix compression
  (i.e. straight/inverse referencing)<br>
  ==> For occasions where value is the affix (e.g., "example.org" in ANY example in
  {{sec:response-examples}}) use shared item referencing to argument table to safe bytes (no extra
  shared item table, no, e.g., 216(""), just simple(0))
  - **Example:** Using Basic CBOR-packed ({{-cbor-packed}}, section 3.1):
    - 130 bytes (Basic CBOR-packed)
    - 200 bytes (plain CBOR, see {{sec:response-examples}})
    - 194 bytes (classic DNS format)

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

# Implementation Status

{::boilerplate RFC7942}

## Python decoder/encoder

The authors of this document provide a [decoder/encoder
implementation](https://github.com/netd-tud/cbor4dns) of both the unpacked and packed format
specified in this document in Python.

Level of maturity:
: prototype

Version compatibility:
: draft-lenders-dns-cbor-10

License:
: MIT

Contact information:
: `Martine Lenders <martine.lenders@tu-dresden.de>`

Last update of this information:
: July 2024

## Embedded decoder/encoder

The authors of this document provide a [decoder/encoder
implementation](https://github.com/RIOT-OS/RIOT/pull/19989) of the unpacked format specified in this
document for the RIOT operating system. It can only encode queries and decode responses.

Level of maturity:
: prototype

Version compatibility:
: draft-lenders-dns-cbor-08

License:
: MIT

Contact information:
: `Martine Lenders <martine.lenders@tu-dresden.de>`

Last update of this information:
: October 2023

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
   IETF CBOR Working Group (cbor@ietf.org) or IETF Applications and Real-Time Area (art@ietf.org)

Intended usage: COMMON

Restrictions on Usage: None?

Author: Martine S. Lenders <m.lenders@fu-berlin.de>

Change controller: IETF

Provisional registrations? No

## CoAP Content-Format Registration

IANA is requested to assign CoAP Content-Format ID for the new DNS message media
types in the "CoAP Content-Formats"
sub-registry, within the "CoRE Parameters" registry {{-coap}}, corresponding the
"application/dns+cbor" media type specified in {{media-type}}:

### "application/dns+cbor" {#cf-app-d-c}

Media-Type: application/dns+cbor

Encoding: -

Id: TBD53

Reference: \[TBD-this-spec\]

### "application/dns+cbor;packed=1"

Media-Type: application/dns+cbor;packed=1

Encoding: -

Id: TBD54

Reference: \[TBD-this-spec\]

## CBOR Tags Registry

In the registry "{{cbor-tags (CBOR Tags)<IANA.cbor-tags}}" {{IANA.cbor-tags}},
IANA is requested to allocate the tags defined in {{tab-tag-values}}.

|    Tag | Data Item        | Semantics                     | Reference              |
|   TBDt | unsigned integer | DNS name suffix extension     | draft-lenders-dns-cbor |
| TBD141 | array            | CBOR EDNS option record       | draft-lenders-dns-cbor |
{: #tab-tag-values cols='r l l' title="Values for Tag Numbers"}

--- back

# Examples

## DNS Queries {#sec:query-examples}

A DNS query of the record `AAAA` in class `IN` for name "example.org" is
represented in CBOR extended diagnostic notation (EDN) (see {{Section 8 of
-cbor}} and {{Appendix G of -cddl}}) as follows:

~~~ cbor-diag
[["example", "org"]]
~~~

A query of an `A` record for the same name is represented as

~~~ cbor-diag
[["example", "org", 1]]
~~~

A query of `ANY` record for that name is represented as

~~~ cbor-diag
[["example", "org", 255, 255]]
~~~

## DNS Responses {#sec:response-examples}

The responses to the examples provided in {{sec:query-examples}} are shown
below. We use the CBOR extended diagnostic notation (EDN) (see {{-edn}} and {{Appendix G of -cddl}}).

To represent an `AAAA` record with TTL 300 seconds for the IPv6 address 2001:db8::1, a minimal
response to `[["example", "org"]]` could be

~~~ cbor-diag
[[[300, h'20010db8000000000000000000000001']]]
~~~

In this case, the name is derived from the query.

If the name or the context is required, the following response would also
be valid:

~~~ cbor-diag
[[["example", "org", 300, h'20010db8000000000000000000000001']]]
~~~

If the query can not be mapped to the response for some reason, a response
would look like:

~~~ cbor-diag
[["example", "org"], [[300, h'20010db8000000000000000000000001']]]
~~~

To represent a minimal response of an `A` record with TTL 3600 seconds for the IPv4 address
192.0.2.1, a minimal response to `[["example", "org", 1]]` could be

~~~ cbor-diag
[[[300, h'c0000201']]]
~~~

Note that here also the 1 of record type `A` can be elided, as this record
type is specified in the question section.

Lastly, a response to `[["example", "org", 255, 255]]` could be

~~~
[
  ["example", "org", 12, 1],
  [[3600, "_coap", "_udp", "local"]],
  [
    [3600, 2, "ns1", TBDt(0)],
    [3600, 2, "ns2", TBDt(0)]
  ],
  [
    [
      TBDt(2), 3600, 28,
      h'20010db8000000000000000000000001'
    ],
    [
      TBDt(2), 3600, 28,
      h'20010db8000000000000000000000002'
    ],
    [
      TBDt(5), 3600, 28,
      h'20010db8000000000000000000000035'
    ],
    [
      TBDt(6), 3600, 28,
      h'20010db8000000000000000000003535'
    ]
  ]
]
~~~

This one advertises two local CoAP servers (identified by service name `_coap._udp.local`) at
2001:db8::1 and 2001:db8::2 and two nameservers for the example.org domain, ns1.example.org at
2001:db8::35 and ns2.example.org at 2001.db8::3535. Each of the transmitted records has a TTL of
3600 seconds.
Note the use of name compression (see {{sec:name-compression}}) in this example.

# Comparison to Classic DNS Wire Format {#sec:comparison-to-classic-dns}

{{tab-cbor-comparison}} shows a comparison between the classic DNS wire format and the
application/dns+cbor format. Note that the worst case results typically appear only rarely in DNS.
The classic DNS format is preferred in those cases. A key for which configuration was used in which
case can be seen in {{tab-cbor-comparison-key}}. Any name label that is longer than 23 bytes adds
a name overhead of 1 byte to its CBOR type header.[^10]{: mlenders}

[^10]: TBD: Also add structured RRs?.

<table anchor="tab-cbor-comparison">
  <name>Comparison of application/dns+cbor to classic DNS format.</name>
  <thead>
    <tr>
      <th align="left" rowspan="2">Item</th>
      <th align="right" rowspan="2">Classic DNS format [bytes]</th>
      <th align="center" colspan="3">application/dns+cbor [bytes]</th>
    </tr>
    <tr>
      <th align="right">best case</th>
      <th align="right">realistic worst case</th>
      <th align="right">theoretical worst case</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td align="left">Header (ID & Flags)</td>
      <td align="right">4</td>
      <td align="right">1</td>
      <td align="right">4</td>
      <td align="right">4</td>
    </tr>
    <tr>
      <td align="left">Count fields</td>
      <td align="right">2</td>
      <td align="right">1</td>
      <td align="right">3</td>
      <td align="right">3</td>
    </tr>
    <tr>
      <td align="left">Question section</td>
      <td align="right">6&nbsp;+&nbsp;name&nbsp;len.</td>
      <td align="right">2&nbsp;+&nbsp;name&nbsp;len.</td>
      <td align="right">6&nbsp;+&nbsp;name&nbsp;len. + name&nbsp;overhead</td>
      <td align="right">9&nbsp;+&nbsp;name&nbsp;len. + name&nbsp;overhead</td>
    </tr>
    <tr>
      <td align="left">Standard RR</td>
      <td align="right">12&nbsp;+&nbsp;name&nbsp;len. + rdata&nbsp;len.</td>
      <td align="right">3<br/> +&nbsp;rdata&nbsp;len.</td>
      <td align="right">14&nbsp;+&nbsp;name&nbsp;len. + rdata&nbsp;len. + name&nbsp;overhead</td>
      <td align="right">17&nbsp;+&nbsp;name&nbsp;len. + rdata&nbsp;len. + name&nbsp;overhead</td>
    </tr>
    <tr>
      <td align="left">Standard RR with name rdata</td>
      <td align="right">12 + name&nbsp;len. + rdata&nbsp;len.</td>
      <td align="right">4 + TBDt&nbsp;len.</td>
      <td align="right">14&nbsp;+&nbsp;name&nbsp;len. + rdata&nbsp;len. + name&nbsp;overheads</td>
      <td align="right">16&nbsp;+&nbsp;name&nbsp;len. + rdata&nbsp;len. + name&nbsp;overheads</td>
    </tr>
    <tr>
      <td align="left">EDNS Opt Pseudo-RR</td>
      <td align="right">11 + options</td>
      <td align="right">2 + options</td>
      <td align="right">6 + options</td>
      <td align="right">14 + options</td>
    </tr>
    <tr>
      <td align="left">EDNS Option</td>
      <td align="right">4 + value&nbsp;len.</td>
      <td align="right">2 + value&nbsp;len.</td>
      <td align="right">4 + value&nbsp;len.</td>
      <td align="right">6 + value&nbsp;len.</td>
    </tr>
  </tbody>
</table>

<table anchor="tab-cbor-comparison-key">
  <name>Configuration key for <xref target="tab-cbor-comparison" />.</name>
  <thead>
    <tr>
      <th align="left" rowspan="2">Item</th>
      <th align="center" colspan="3">application/dns+cbor configuration</th>
    </tr>
    <tr>
      <th align="right">best case</th>
      <th align="right">realistic worst case</th>
      <th align="right">theoretical worst case</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td align="left">Header (ID &amp; Flags)</td>
      <td align="right">Flags elided</td>
      <td align="right">QR, Opcode, AA, TC, or RD are set</td>
      <td align="right">QR, Opcode, AA, TC, or RD are set</td>
    </tr>
    <tr>
      <td align="left">Count fields</td>
      <td align="right">Encoded in CBOR array header</td>
      <td align="right">Encoded in CBOR array header,<br/>&gt;255 records in section</td>
      <td align="right">Encoded in CBOR array header,<br/>&gt;255 records in section</td>
    </tr>
    <tr>
      <td align="left">Question section</td>
      <td align="right">Class, type, and name elided</td>
      <td align="right">Type &gt; 255,<br/>label len. &gt; 23</td>
      <td align="right">Type &gt; 255,<br/>Class &gt; 255,<br/>label len. &gt; 23</td>
    </tr>
    <tr>
      <td align="left">Standard RR</td>
      <td align="right">Class, type, and name elided,<br/>rdata len. &lt; 24</td>
      <td align="right">Type &gt; 255,<br/>label len. &gt; 23<br/>rdata len. &gt; 255</td>
      <td align="right">Type &gt; 255,<br/>Class &gt; 255,<br/>label len. &gt; 23<br/>rdata len. &gt; 255</td>
    </tr>
    <tr>
      <td align="left">Standard RR with name rdata</td>
      <td align="right">Class, type, and name elided,<br/>TBDt(i) with i&nbsp;&lt; 24</td>
      <td align="right">Type &gt; 255,<br/>label len. &gt; 23<br/>name uncompressed</td>
      <td align="right">Type &gt; 255,<br/>Class &gt; 255,<br/>label len. &gt; 23<br/>name uncompressed</td>
    </tr>
    <tr>
      <td align="left">EDNS Opt Pseudo-RR</td>
      <td align="right">All EDNS(0) fields elided</td>
      <td align="right">Rcode &lt; 24,<br/>DO flag set,<br/></td>
      <td align="right">UDP payload<br/>len. &gt; 255<br/>Rcode &gt; 255<br/>Version &gt; 255<br/>DO flag set</td>
    </tr>
    <tr>
      <td align="left">EDNS Option</td>
      <td align="right">Code &lt; 24<br/>Length &lt; 24</td>
      <td align="right">Code &lt; 24<br/>Length &gt; 255</td>
      <td align="right">Code &gt; 255<br/>Length &gt; 255</td>
    </tr>
  </tbody>
</table>

# Change Log

Since [draft-lenders-dns-cbor-10]
---------------------------------
- Address IANA #1392416 early review
- Fix external section references
- Update implementation status

Since [draft-lenders-dns-cbor-09]
---------------------------------
- Add recommendation on label encoding
- Provide extension points
  - Mark dns-rr specifically as extension point
  - Provide extension points for parameter values (options and svc-params)
- Point out CBOR-packed needs to be unpacked when identifying names
- Distinguish from C-DNS {{-cdns}}
- State objectives in introduction
- Fix nits and typos

Since [draft-lenders-dns-cbor-08]
---------------------------------
- Clarify why question section was designed the way it is
- Add answer section to queries for Known Answers in mDNS
- Express names as sequence of labels
- Provide dedicated types for more structured RDATA
- Add RFC1035-like name compression
- Add switching boolean to query message to explicitly have question present in response
- Make EDNS options a map
- Update examples and comparison table in appendices
- Update implementation section

Since [draft-lenders-dns-cbor-07]
---------------------------------
- Add {{sec:comparison-to-classic-dns}} with comparison to classic DNS wire format
- "wire format" -> "classic DNS wire format"

Since [draft-lenders-dns-cbor-06]
---------------------------------
- Fixes wording and spelling mistakes

Since [draft-lenders-dns-cbor-05]
---------------------------------
- Fix {{cf-app-d-c}} title
- Amend for capability to carry more than one question
- Hint at future of name compression in later draft versions
- Use canonical name for CBOR-packed

Since [draft-lenders-dns-cbor-04]
---------------------------------
- Add Implementation Status section
- Remove int as representation for rdata
- Add note on representation of more structured rdata

Since [draft-lenders-dns-cbor-03]
---------------------------------
- Provide format description for EDNS OPT Pseudo-RRs
- Simplify CDDL to more idiomatic style
- Remove DNS transaction IDs

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

[draft-lenders-dns-cbor-10]: https://datatracker.ietf.org/doc/html/draft-lenders-dns-cbor-10
[draft-lenders-dns-cbor-09]: https://datatracker.ietf.org/doc/html/draft-lenders-dns-cbor-09
[draft-lenders-dns-cbor-08]: https://datatracker.ietf.org/doc/html/draft-lenders-dns-cbor-08
[draft-lenders-dns-cbor-07]: https://datatracker.ietf.org/doc/html/draft-lenders-dns-cbor-07
[draft-lenders-dns-cbor-06]: https://datatracker.ietf.org/doc/html/draft-lenders-dns-cbor-06
[draft-lenders-dns-cbor-05]: https://datatracker.ietf.org/doc/html/draft-lenders-dns-cbor-05
[draft-lenders-dns-cbor-04]: https://datatracker.ietf.org/doc/html/draft-lenders-dns-cbor-04
[draft-lenders-dns-cbor-03]: https://datatracker.ietf.org/doc/html/draft-lenders-dns-cbor-03
[draft-lenders-dns-cbor-02]: https://datatracker.ietf.org/doc/html/draft-lenders-dns-cbor-02
[draft-lenders-dns-cbor-01]: https://datatracker.ietf.org/doc/html/draft-lenders-dns-cbor-01
[draft-lenders-dns-cbor-00]: https://datatracker.ietf.org/doc/html/draft-lenders-dns-cbor-00

# Acknowledgments
{:unnumbered}

TODO acknowledge.
