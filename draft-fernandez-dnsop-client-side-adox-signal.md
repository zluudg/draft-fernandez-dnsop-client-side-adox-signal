---
title: "Client-side Request for Encrypted Authoritative DNS Queries"
abbrev: "Client-side ADoX Signal"
category: std

docname: draft-fernandez-dnsop-client-side-adox-signal-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: OPS
workgroup: DNSOP
keyword:
 - DNS Privacy
venue:
  group: DNSOP
  type: Working Group
  mail: dnsop@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/dnsop/
  github: "zluudg/draft-client-side-adox-signal"
  latest: "https://zluudg.github.io/draft-client-side-adox-signal/draft-fernandez-dnsop-client-side-adox-signal.html"

author:
 -
    fullname: "Leon Fernandez"
    organization: The Swedish Internet Foundation
    email: "leon.fernandez@internetstiftelsen.se"
 -
    fullname: "Valentin Gosu"
    organization: Mozilla
    email: "ietf.valentin@gosu.se"

normative:

informative:

...

--- abstract

This document describes an EDNS0 option that a client can attach to a
DNS query to signal to a recursive resolver that it SHOULD prefer
encrypted transports while resolving the query. The option can also
be used on the return path by the resolver to signal whether encryption
was achievable or not.



--- middle

# Introduction {#intro}
A DNS client can choose what protocol to use when it sends a query to
a resolver. There are several privacy-friendly options that the client
can choose between such as HTTPS {{!RFC8484}}, QUIC {{!RFC9250}} and TLS {{!RFC7858}}. However, the choice
only affects the first hop and the query will typically be sent in
cleartext by the resolver during the resolution process. In order to
preserve the privacy of the client beyond the first hop, the client
SHOULD be able to signal a desire that the resolver uses encrypted
transports during the resolution process. Furthermore, the client
SHOULD be informed of how the resolution was carried out, in case it
would like to discard data that was not received over an encrypted
transport.

This document defines an EDNS0 {{!RFC6891}} option that allows a client
to signal a preference for encrypted authoritative DNS (ADoX). By
signalling a preference, the client can urge a recursive resolver to
discover possibilities for encrypted transport. Means of discovery
include, but are not limited to, {{?RFC9539}}, {{?I-D.johani-dnsop-transport-signaling}}
and {{?I-D.wesplaap-deleg-transport}}. The signal from the client can
also prevent the recursive resolver from leaking queries that would
otherwise have been sent in cleartext. The same EDNS0 option can also
be used on the return path so that the recursive resolver can give an
indication to the client about what level of privacy was achieved for
a particular query.

# Terminology
Client: A stub resolver as defined in {{!RFC9499}} or some other
    software, such as a browser, is close to the end-user and sends
    recursice DNS queries.

Resolver: As defined in {{!RFC9499}}.

ADoX: Authoritative DNS-over-X, where X implies some arbitrary encrypted
    transport protocol.

# The EDNS0 PRIVACY Option
The EDNS0 PRIVACY option is structured as follows:

~~~
                                               1   1   1   1   1   1
       0   1   2   3   4   5   6   7   8   9   0   1   2   3   4   5
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 0:  |                            OPTION-CODE                        |
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 2:  |                           OPTION-LENGTH                       |
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 4:  |          PRIVACY REQUEST      |        PRIVACY_RESPONSE       |
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
~~~

Field definition details:

OPTION-CODE: The EDNS0 option code for client-privacy-signal, defined
    as TBD1.

OPTION-LENGTH: The option length MUST have a value of 2 if this option
    is present. Consumers of this signal SHOULD NOT act on
    malformed options.

PRIVACY REQUEST: This value encodes the client's desired level of privacy.
    Currently defined values are listed in {{privacy-req-values}}.
    In a DNS response, the recipient MUST ignore this field and
    the sender SHOULD set it to 0.

PRIVACY RESPONSE: This value encodes the level of privacy that the resolver
    was able to achieve. Currently defined values are listed in
    {{privacy-rsp-values}}. In a DNS query, the recipient MUST ignore
    this field and the sender SHOULD set it to 0.

# PRIVACY REQUEST Values {#privacy-req-values}

| Value | Mnemonic                  |
|-------|---------------------------|
| 0     | NONE                      |
| 1     | OPPORTUNISTIC             |
| 2     | OPPORTUNISTIC-NO-FALLBACK |
| 3     | STRICT                    |

# PRIVACY RESPONSE Values {#privacy-rsp-values}

| Value | Mnemonic      |
|-------|---------------|
| 0     | CLEARTEXT     |
| 1     | ENCRYPTED     |
| 2     | CACHED        |
| 3     | AUTHENTICATED |

## Interpretations of Privacy Request Values

### NONE
Same effect as having no client signal at all. No privacy requested by
the client that is sending the query.

### OPPORTUNISTIC
The client is requesting a preference for resolving the query using
encrypted transports. However, fallback to unencrypted transport is
acceptable.

### OPPORTUNISTIC-NO-FALLBACK
The client is requesting a preference for resolving the query using
encrypted transports. Fallback to unencrypted transport is not
acceptable.

### STRICT
The client is requesting a preference for resolving the query using
authenticated encrypted transport.

## Interpretations of Privacy Response Values

### CLEARTEXT
The resolver had to send at least one unencrypted query during
resolution.

### ENCRYPTED
The resolver only sent encrypted queries or used cached data during
resolution.

### CACHED
The final answer was fetched directly from the resolver's cache.

### AUTHENTICATED
The resolver only sent encrypted queries or used cached data during
resolution. Moreover, encrypted connections were authenticated, for
example via a TLSA record, and cached data was only used if it was
received via an authenticated transport.


# Protocol Description
A client adds the PRIVACY EDNS0 option to a recursive query before
sending it. A client SHOULD NOT add the PRIVACY option to recursive
queries being sent in cleartext as this would be counter to what the
client is trying to achieve.

Given that the client sent its query using an encrypted transport, the
recursive resolver will try to honor the PRIVACY option. If it is NONE,
it means the client does not care about what transports are used to
resolve the query and the resolver may resolve it in any way deemed
suitable. If the option is OPPORTUNISTIC, the resolver MUST attempt
encrypted transport to all nameserver it knows support encryption while
traversing the delegation chain. However, the
resolver is allowed to fall back to an unencrypted transport after a
failed attempt. If the option is OPPORTUNISTIC-NO-FALLBACK the resolver MUST NOT perform
such a fallback at any point during the resolution process. Using
contents from the cache is always okay as this does not leak any
information about the client's query. If the STRICT option is set,
the resolver MUST authenticate all encrypted connections it makes during
resolution. If cached data is to be used during resolution, it MUST have
been received over an authenticated encrypted connection.

## Setting a PRIVACY RESPONSE Value
A recursive resolver SHOULD only set an EDNS0 PRIVACY option if the
option was present in the query. If the data was readily available in
the resolver's cache, it sets a value of CACHED. If at any point
during the resolution process, a query was sent unencrypted (either
due to falling back or to the client query having a value of NONE), the
resolver MUST set the value to CLEARTEXT. If the resolver solely used
encrypted transports and data from its cache to retrieve the answer, it
responds with a value of ENCRYPTED. What transport cached data arrived
over does not matter in this case. If the resolver was able to resolve
the query using only data that arrived over authenticated transports
(either from the cache or from a response), it response with a value
of AUTHENTICATED

## Failure to Honor a OPPORTUNISTIC-NO-FALLBACK or STRICT Request
If the recursive resolver cannot obtain the answer without sending a
query over an unencrypted transport, and the client requested OPPORTUNISTIC-NO-FALLBACK,
the resolver MUST NOT send the query that could not be sent encrypted. It MUST instead respond with
RCODE SERVFAIL and SHOULD include an Extended DNS Error {{!RFC8914}}
with INFO-CODE TBD2 ("Encrypted Transport Unavailable"). The response
MUST include the PRIVACY option with the value ENCRYPTED, since no
information about the query was sent in cleartext. Failure to honor
a STRICT request works similarly, the resolver sets an INFO-CODE of
TBD2 and an AUTHENTICATED value for the PRIVACY RESPONSE field.

It is expected that during early deployment, most OPPORTUNISTIC-NO-FALLBACK and STRICT requests will
fail. However, for debugging purposes, such a signal is valuable for
testing that recursive resolvers actually honor the encryption request.
Also, as the technology matures, the OPPORTUNISTIC-NO-FALLBACK and STRICT options might eventually
become viable.

# Relation to QNAME minimisation
QNAME minimisation {{!RFC9156}} is another privacy-enhancement for
authoritative DNS that recursive resolvers can implement. It SHOULD be
considered orthogonal to what is proposed in this draft. That is, the
value that the resolver sets for the PRIVACY option SHOULD NOT be
affected in any way by whether the resolver employs QNAME minimisation
or not. QNAME minimization does not give as good privacy as encryption
if there are more than two or three delegations. On the other hand,
QNAME minimization avoids leaking information to authoritative servers,
which means there might still be good reasons to use it.

# Conventions and Definitions
{::boilerplate bcp14-tagged}

# Implementation Status {#impl-status}
{: removeInRFC="true"}

This section records the status of known implementations of the
protocol defined by this specification at the time of posting of
this Internet-Draft, and is based on a proposal described in
{{?RFC7942}}. The description of implementations in this section is
intended to assist the IETF in its decision processes in progressing
drafts to RFCs. Please note that the listing of any individual
implementation here does not imply endorsement by the IETF.
Furthermore, no effort has been spent to verify the information
presented here that was supplied by IETF contributors. This is not
intended as, and must not be construed to be, a catalog of available
implementations or their features. Readers are advised to note that
other implementations may exist.

According to {{?RFC7942}}, "this will allow reviewers and working
groups to assign due consideration to documents that have the benefit
of running code, which may serve as evidence of valuable
experimentation and feedback that have made the implemented protocols
more mature. It is up to the individual working groups to use this
information as they see fit".

## TDNS

Organization:
: The Swedish Internet Foundation

Description:
: TDNS, a framework of experimental DNS servers.
  <https://github.com/johanix/tdns>

Maturity:
  : Experimental

  Coverage:
  : Recursive resolver: parsing the client signal, honoring
    OPPORTUNISTIC requests, setting the response PRIVACY value.
    Stub/client: Setting the PRIVACY value and interpreting the PRIVACY
    value in the response.

  Licensing:
  : BSD-2-Clause

  Contact:
  : johan.stenstam@internetstiftelsen.se

  Last updated:
  : October 2026

## Firefox

Organization:
: Mozilla

Description:
: Firefox, a web-browser.
  <https://phabricator.services.mozilla.com/D321467>

Maturity:
  : Experimental

  Coverage:
  : Setting the PRIVACY value in outgoing DoH queries and interpreting
  the PRIVACY value in the response.

  Licensing:
  : MPL-2.0

  Contact:
  : ietf.valentin@gosu.se

  Last updated:
  : October 2026

# Security Considerations {#security}
Enables/simplifies cache probing through the CACHED option. However,
this is already possible through measuring response times or inspecting
TTLs. The PRIVACY option does not introduce any novel attack vectors.

The client can only really trust the PRIVACY option when it is received
over a secure channel, such as DoH, from a trusted recursive resolver.
This is somewhat analogous to how the AD-bit is used between a client
and a resolver in DNSSEC {{!RFC4035}}.

# IANA Considerations {#iana}
This document defines a new EDNS(0) option, entitled "client-privacy-signal",
assigned a value of TBD1 "DNS EDNS0 Option Codes (OPT)" registry

| Value | Name                     | Status   | Reference            |
|-------|--------------------------|----------|----------------------|
| TBD1  | client-privacy-signal  | Standard | ( This document )    |

This document also defines a new  INFO-CODE,
entitled "Encrypted Transport Unavailable", assigned a value of TBD2
in the "Extended DNS Error Codes" registry.

| Value | Name                     | Status   | Reference            |
|-------|--------------------------|----------|----------------------|
| TBD2  | encrypted-transport-unavailable  | Standard | ( This document )    |

--- back


# Acknowledgments
{:numbered="false"}

TODO acknowledge.
