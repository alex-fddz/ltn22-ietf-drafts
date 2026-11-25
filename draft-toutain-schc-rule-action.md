---
title: "Actions for Static Context Header Compression (SCHC) Rules"
abbrev: "SCHC Rule Action"
docname: draft-toutain-schc-rule-action-00
category: std
ipr: trust200902
submissiontype: IETF
v: 3
area: "Internet"
workgroup: "schc Working Group"
keyword:
 - SCHC
 - Rule
 - Action
 - Proxy
date:

author:
 -
    ins: L. Toutain
    name: Laurent Toutain
    org: "IMT Atlantique"
    street: "2 rue de la Chataigneraie CS 17607"
    city: "Cesson-Sevigne Cedex"
    code: "35576"
    country: "France"
    email: laurent.toutain@imt-atlantique.fr
 -
    ins: A. Minaburo
    name: Ana Minaburo
    org: "Consultant"
    country: "France"

normative:
  I-D.toutain-schc-rfc9363bis:
  RFC2119:
  RFC4443:
  RFC8174:
  RFC8724:

informative:
  I-D.ietf-schc-icmpv6-compression:
  I-D.toutain-schc-coreconf-management:

--- abstract

In the SCHC framework, selecting a compression Rule triggers the
compression of the packet and the transmission of the resulting SCHC
Packet to the other end. This document introduces the Action of a Rule,
which extends this behavior: the Rule is still selected through its
Matching Operators, and the packet is also handed over to a local
process named by the Action. Depending on the Action, the compression
is either stopped, or continues and the SCHC Packet is still sent. As a
first Action, it defines a ping proxy, where the SCHC Core answers
ICMPv6 Echo Requests on behalf of a Device.

--- middle

# Introduction {#introduction}

In SCHC {{RFC8724}}, a compression Rule plays two roles. Its Matching
Operators select the Rule for an incoming packet, and its
Compression/Decompression Actions build the SCHC Packet, which is then
sent to the other end.

The motivation of the Action is to add a treatment once the matching
phase has selected the Rule. Depending on the result of the Action,
either the Action issues a residue, which is sent to the other end, or
the treatment of the Action is just performed and nothing is sent.

The Action can be used, for example, to proxy a treatment in the SCHC
Core on behalf of the Device. The ping proxy is such an example: the
SCHC Core replies to the ICMPv6 {{RFC4443}} Echo Requests sent to the
Device when certain conditions are met, for instance when the Device
has sent SCHC Packets during a period of time. The Echo Request and the
Echo Reply then do not cross the constrained link.

The Action can also be used during Rule management procedures
{{I-D.toutain-schc-coreconf-management}}. For instance, a more generic
Rule, ignoring the application address and port, can trigger the
creation of a more specific Rule, which sets these values.

# Requirements Language

{::boilerplate bcp14-tagged}

# Terminology {#terminology}

This document reuses the terminology of {{RFC8724}}.

Action:
: The process applied to a packet that matches a compression Rule. The
  Action applies to the Rule as a whole, and not to a single entry.

# Action of a Rule {#action}

A compression Rule MAY carry an Action. A Rule matches a packet through
the Matching Operators of its entries, as specified in {{RFC8724}}. Once
a Rule matches the packet, the packet is handed over to the process
named by its Action.

The default Action is `action-forward`: the packet is compressed and the
SCHC Packet is sent to the lower layer, as specified in {{RFC8724}}. A
Rule without an Action therefore behaves as in {{RFC8724}}.

The other Actions fall into two categories:

* Some Actions block the compression phase: the packet is processed
  locally and nothing is sent to the other end. The Rule is then only
  used for the selection, the Compression/Decompression Actions of its
  entries do not matter, and they SHOULD be set to `not-sent`.
  `action-proxy-ping` ({{proxy-ping}}) belongs to this category.

* Other Actions issue a residue, which is sent to the other end in
  the SCHC Packet, as specified in {{RFC8724}}.

The definition of an Action MUST state to which category it belongs.

When the Matching Operators of several Rules match the packet, the
Actions of all these Rules are applied. The residues issued by the
Actions allowed to create one, including `action-forward`, take part in
the selection process, so that only one residue, for instance the
shortest one, is sent to the other end.

## Action Arguments {#action-arguments}

An Action may take arguments. They are viewed as a list of values, each
built like a Target Value, as for the arguments of the Matching
Operators and of the Compression/Decompression Actions
{{I-D.toutain-schc-rfc9363bis}}. `action-forward` takes no argument.

## Representation {#representation}

Since the Action applies to the Rule as a whole, it is shown above the
entries of the Rule, with its arguments between parentheses, as in
{{fig-action-rule}}.

# Ping Proxy {#proxy-ping}

The Action `action-proxy-ping` processes the packet locally: the SCHC
Core answers the ICMPv6 Echo Request on behalf of the Device, and
nothing is sent on the constrained link.

The Action takes one argument, in seconds, giving the interval during
which the Device is considered active. During this interval, the proxy
answers the Echo Requests; after it, they are discarded.

{{fig-action-rule}} shows a compression Rule for pinging a Device, with
the identity short names used in the Field and CDA columns.

~~~ aasvg
+--------------------------+
| Action: proxy-ping(300)  |
+----+----------------+----+----+----+----+---------+----------+
| #  | Field          | FL | FP | DI | TV | MO      | CDA      |
+----+----------------+----+----+----+----+---------+----------+
| IPv6 Headers description (omitted)                           |
+----+----------------+----+----+----+----+---------+----------+
| 10 | icmpv6-type    | 8  | 1  | Dw | 128| equal   | not-sent |
| 11 | icmpv6-code    | 8  | 1  | Bi | 0  | equal   | not-sent |
| 12 | icmpv6-        | 16 | 1  | Bi | -  | ignore  | not-sent |
|    | identifier     |    |    |    |    |         |          |
| 13 | icmpv6-        | 16 | 1  | Bi | 0  | MSB(24) | not-sent |
|    | sequence       |    |    |    |    |         |          |
+----+----------------+----+----+----+----+---------+----------+
~~~
{: #fig-action-rule title="Example Compression Rule for a Ping to a Device"}

The ICMPv6 Field IDs are defined in {{I-D.ietf-schc-icmpv6-compression}}.

# YANG Data Model Extensions {#yang}

The complete YANG module, including the leaves and identities listed
below, is found in {{I-D.toutain-schc-rfc9363bis}}. This section only
lists the elements related to this document.

The Action is added to the compression Rule itself, as a sibling of its
entries, and not to a single entry:

* `action`: an identityref to `action-type`, defaulting to
  `action-forward`. Being optional and defaulted, a Rule that does not
  carry it behaves exactly as in {{RFC8724}}.
* `action-value`: the arguments of the Action, a list based on the
  Target Value structure.

The corresponding identities are:

* `action-base-type`: base identity for the Action applied to a packet
  matching a compression Rule.
* `action-forward`: the default, where the packet is handed over to the
  lower layer, as specified in {{RFC8724}}.
* `action-proxy-ping`: the packet is processed locally by a ping proxy,
  which answers the Echo Request on behalf of the Device.

# Security Considerations {#security-considerations}

A ping proxy answers on behalf of a Device that may be unreachable or
switched off. The interval given as argument limits the time during
which the SCHC Core claims that the Device is active.

TODO

# IANA Considerations {#iana-considerations}

This document has no IANA actions.

--- back

# Changes {#changes}

* Initial version. The Action of a Rule and the ping proxy example come
  from the "Device is pinged" section of
  {{I-D.ietf-schc-icmpv6-compression}}, since the Action is not
  specific to ICMPv6. The Action is described independently of ICMPv6
  ({{action}}), with its arguments ({{action-arguments}}) and its
  representation in a Rule ({{representation}}).
* Authors: added Ana Minaburo as co-author.
* Abstract, {{introduction}}, {{action}}: an Action no longer always
  replaces the compression. Some Actions block the compression phase,
  others issue a residue, which is sent to the other end.
* {{action}}: the first paragraph states that the packet is handed over
  to the Action once a Rule matches it, instead of once the Rule is
  selected.
* {{action}}: when several Rules match the packet, the Actions of all
  these Rules are applied, and the residues issued by the Actions take
  part in the selection process, so that only one residue is sent to the
  other end.
* {{introduction}}: added a paragraph on the use of the Action during
  Rule management procedures, where a generic Rule ignoring the
  application address and port triggers the creation of a more specific
  Rule setting these values.
* {{introduction}}: the last paragraph presents the Action as a way to
  proxy a treatment in the SCHC Core on behalf of the Device, the ping
  proxy replying to the Echo Requests when certain conditions are met,
  for instance when the Device has sent SCHC Packets during a period of
  time.
* {{introduction}}: the second paragraph now states the motivation of
  the Action, a treatment added once the matching phase has selected
  the Rule, which either issues a residue sent to the other end, or is
  just performed.
