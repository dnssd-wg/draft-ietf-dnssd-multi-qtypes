---
title: DNS Multiple QTYPEs
docname: draft-ietf-dnssd-multi-qtypes-01
submissiontype: IETF
number:
date:
consensus: true
v: 3
category: std
area: Internet
wg: DNSSD
venue:
  group: DNSSD
  type: Working Group
  mail: dnssd@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/dnssd/
  github: dnssd-wg/draft-ietf-dnssd-multi-qtypes
  latest: https://dnssd-wg.github.io/draft-ietf-dnssd-multi-qtypes/draft-ietf-dnssd-multi-qtypes.html
keyword:
  - DNS
  - resource record
author:
  -
    ins: R. Bellis
    name: Ray Bellis
    org: Internet Systems Consortium, Inc.
    abbrev: ISC
    street: PO Box 360
    city: Newmarket
    code: NH 03857
    country: USA
    phone: +1 650 423 1300
    email: ray@isc.org

--- abstract

This document specifies a method for a DNS client to request additional
DNS record types to be delivered alongside the primary record type
specified in the question section of a DNS query.

--- middle

# Introduction

A commonly requested DNS {{!RFC1035}} feature is the ability to receive
multiple related resource records (RRs) in a single DNS response.

For example, it may be desirable to receive the A, AAAA and HTTPS
records for a domain name together, rather than having to issue
multiple queries.

The DNS wire protocol in theory supported having multiple questions in a
single packet, but in practise this does not work.  In
{{!I-D.draft-ietf-dnsop-qdcount-is-one}}, {{!RFC1035}} is updated to only
permit a single question in a QUERY (OpCode == 0) request.

Sending QTYPE=ANY does not guarantee that all RRsets will be returned.
{{?RFC8482}} specifies that responders may return a single RRset of
their choosing.

To mitigate these issues, this document constrains the problem to those
cases where only the QTYPE varies by specifying a new option for the
Extension Mechanisms for DNS (EDNS {{!RFC6891}}) that contains an
additional list of QTYPE values that the client wishes to receive in
addition to the single QTYPE appearing in the question section.  A
second EDNS option is used in response packets as protection against DNS
middleboxes that echo EDNS options verbatim.

The specification described herein is applicable both for queries from a
stub resolver to recursive servers, and from recursive resolvers to
authoritative servers.

# Terminology used in this document

{::boilerplate bcp14-tagged}

# Description

## Multiple QTYPE EDNS Options Format

The overall format of an EDNS option is shown for reference below,
per {{!RFC6891}}, followed by the option specific data:

       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    0: |                          OPTION-CODE                          |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    2: |                         OPTION-LENGTH                         |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    4: |                                                               |
       /                          OPTION-DATA                          /
       /                                                               /
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

OPTION-CODE: MQTYPE-Query (TBD1) in queries and MQTYPE-Response (TBD2) in responses.

OPTION-LENGTH: Size (in octets) of OPTION-DATA.

OPTION-DATA: Option specific, as below:

                    +0 (MSB)                            +1 (LSB)
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    0: |           QT1 (MSB)           |           QT1 (LSB)           |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    2: /              ...              |              ...              /
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       /           QTn (MSB)           |           QTn (LSB)           |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

QT: a (potentially empty) list of 2 byte fields (QTx) in network order
(MSB first) each specifying a DNS RR type.  The RR types MUST be for
real resource records, and MUST NOT refer to pseudo RR types such as
"OPT", "IXFR", "TSIG", "*", etc.

## Server Response Generation

A conforming server that receives an MQTYPE-Query option in a query MUST
return an MQTYPE-Response option in its response.  The use of a
secondary option code in responses serves as protection against DNS
middleboxes that echo EDNS OPT Records verbatim.

A server that receives an MQTYPE-Response option in a query MUST return
a FORMERR response.

On receipt of a valid MQTYPE-Query option the server SHOULD attempt to
return any resource records known to it that match the additional
(QNAME, QTx, QCLASS) tuples.  These records MUST be returned in the
Answer Section of the response, but the answer for the primary QTYPE
from the Question Section MUST be included first.

If any invalid QTx is received in the query (e.g. one corresponding to a
meta-RR) the server MUST return a FORMERR response.

For any particular QTx in the query, if the server provides additional
answers, or has knowledge that the RR type does not exist for that
QNAME (a "negative answer"), it must include that QTx value in the
Multiple QTYPE Option of its response.

A negative answer is therefore indicated by the combination of the
presence of a QTx value in the Multiple QTYPE Option and the absence of
a matching record in the Answer Section.  This is necessary (in the
absence of DNSSEC) to differentiate between absence of the record from
the zone and absence of the record from the response.

A server that is authoritative for the specified QNAME on receipt of a
Multiple QTYPE Option MUST attempt to return all specified RR types
except where that would result in truncation or a risk of a significant
DNS amplification attack in which case it MAY omit some (or all) of the
records for the additional RR types.  Those RR types MUST then also be
omitted from the Multiple QTYPE Option in the response.

A caching recursive server receiving a Multiple QTYPE Option query
SHOULD attempt to fill its positive and negative caches with all of the
specified RR types before returning its response to the client.  It MAY
limit itself to a smaller subset of the specified RR types if the
processing overhead to fill its caches is too great or if there is a
risk of a significant DNS amplification attack.

While this document specifies no limit on the number of QTx values that
may be specified, the author anticipates that server implementations
will provide configuration settings to constrain the response sizes.

### DNSSEC

If the DNS client sets the "DNSSEC OK" (DO) bit in the query then the
server MUST also return the related DNSSEC records that would have been
returned in a standalone query for the same QTYPE.

A negative answer from a signed zone MUST contain the appropriate
authenticated denial of existence records, per {{!RFC4034}} and
{{!RFC5155}}.

In a signed zone there is a theoretical risk of valid signatures for one
RR type and invalid signatures for another.  This is the only case known
to the author where the response code for any particular QNAME may be
inconsistent across different RR types.

Should a validating resolver produce NOERROR for some RR types and
SERVFAIL for others it MUST omit the RR types that failed to validate
from its response and from the QTx fields on the Multiple QTYPE option.

## Client Response Processing

Recursive resolvers MAY use this method to obtain multiple records from
an authoritative server.  For the purposes of Section 5.4.1 of
{{!RFC2181}} any authoritative answers received MUST be ranked the same
as the answer for the primary question.

If the response to a query containing an MQTYPE-Query option does not
contain an MQTYPE-Response option, or if it erroneously contains an
MQTYPE-Query option, the client MUST treat the response as if this
option is unsupported by the server and SHOULD process the response as
normal.

The client SHOULD subsequently initiate standalone queries (i.e. without
using the MQTYPE-Query option) for any QTx value that did not generate a
negative answer.

# Security Considerations

The method documented here does not change any of the security
properties of the DNS protocol itself.

It should however be noted that this method does increase the potential
amplification factor when the DNS protocol is used as a vector for a
denial of service attack.

# IANA Considerations

NB: to be rewritten once assignments have been made.

IANA is requested to assign two new values (TBD1 and TBD2) in the DNS
EDNS0 Option Codes registry for MQTYPE-Query and MQTYPE-Response.  They
should be consecutive, with the -Query option being an even number.

--- back

# Acknowledgements
{:numbered="false"}

The author wishes to thank the following for their feedback and reviews
during the initial development of this document: Michael Graff, Olafur
Gudmundsson, Matthijs Mekking, and Paul Vixie.

In addition the author wishes to thank the following for subsequent
review during discussion in the DNSSD Working Group: Chris Box, Stuart
Cheshire, Esko Dijk, Ted Lemon, and David Schinazi.
