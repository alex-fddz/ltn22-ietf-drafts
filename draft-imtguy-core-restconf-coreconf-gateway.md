---
title: "A RESTCONF/CORECONF Gateway"
abbrev: "RESTCONF-CORECONF Gateway"
docname: draft-imtguy-core-restconf-coreconf-gateway-00
category: std
ipr: trust200902
submissiontype: IETF
v: 3
area: "Applications and Real-Time"
workgroup: "core Working Group"
keyword:
 - RESTCONF
 - CORECONF
 - YANG
 - Gateway
date:

author:
 -
    ins: J. A. Fernandez
    name: Javier A. Fernandez
    org: "IMT Atlantique"
    street: "2 rue de la Chataigneraie CS 17607"
    city: "Cesson-Sevigne Cedex"
    code: "35576"
    country: "France"
    email: javier-alejandro.fernandez@imt-atlantique.fr
 -
    ins: L. Toutain
    name: Laurent Toutain
    org: "IMT Atlantique"
    street: "2 rue de la Chataigneraie CS 17607"
    city: "Cesson-Sevigne Cedex"
    code: "35576"
    country: "France"
    email: laurent.toutain@imt-atlantique.fr

normative:
  I-D.ietf-core-comi:
  I-D.toutain-core-sid-extension:
  RFC2119:
  RFC7252:
  RFC8040:
  RFC7641:
  RFC8132:
  RFC8174:
  RFC9254:
  RFC9595:

informative:
  RFC6690:
  RFC9423:
  I-D.ietf-core-yang-library:
  RFC7317:
  I-D.toutain-core-sid-encoding:

--- abstract

This document describes a gateway between RESTCONF and CORECONF. The
gateway is a simple proxy that converts the RESTCONF requests into
CORECONF FETCH and iPATCH requests, so that constrained devices can be
managed with RESTCONF tools.

--- middle

# Introduction {#introduction}

CORECONF {{I-D.ietf-core-comi}} brings the YANG data modeling language
to constrained devices. To fit the limited resources of these devices,
it relies on a compact datastore: YANG identifiers are replaced by
numeric SIDs {{RFC9595}}, the data are encoded in CBOR {{RFC9254}}, and
they are manipulated through a CoAP {{RFC7252}} API.

Network management environments are, for a large part, built around the
RESTCONF paradigm {{RFC8040}}. RESTCONF exposes the same YANG data
models, but through an HTTP API, with data encoded in JSON or XML and
with nodes identified by their names. The tools of these environments
(controllers, orchestrators, scripts, browsers) therefore cannot
directly talk to a CORECONF device: they do not speak CoAP, they do not
manipulate SIDs and CBOR, they do not include SIDs in the URI, and they
do not know the compact datastore organization. Integrating constrained
devices in such an environment would require each tool to be extended.

This document describes a gateway that makes a CORECONF device look like
a RESTCONF server. The gateway receives RESTCONF requests, converts them
into CORECONF requests, and converts the answers back, so that the
constrained devices can be managed with the existing RESTCONF tools,
without any change on the device side.

# Requirements Language

{::boilerplate bcp14-tagged}

# Terminology {#terminology}

# Gateway Architecture {#architecture}

The gateway is a simple proxy: it receives the RESTCONF requests and
converts them into CORECONF FETCH and iPATCH requests {{RFC8132}}, and
it converts the answers back.

The RESTCONF requests designate the data with a path expressed with
YANG identifiers, in the XPath style, and carry the data in JSON or XML.
The CORECONF FETCH and iPATCH requests use the CBOR representation.

To do this conversion, the gateway needs the SID files with the SID
extensions {{I-D.toutain-core-sid-extension}}. These files can be
directly stored inside the gateway, or can be discovered using the DNS
{{I-D.toutain-core-sid-encoding}}.

## Example {#example}

The examples use the module ietf-system {{RFC7317}}, with the SIDs
assigned in {{I-D.ietf-core-comi}}: system-state/clock/current-datetime
is SID 1723, ntp/server (a list keyed by name) is SID 1756, name is SID
1759, prefer is SID 1760, udp is SID 1761 and address is SID 1762. The
gateway finds these SIDs, the keys of the lists, and the types of the
keys in the SID file with the SID extensions. The YANG identifiers of
the RESTCONF path are replaced by SIDs, and each predicate value is
converted into CBOR according to the type of the key.

### Simple URI

A URI without predicate designates a leaf. It becomes a single SID.

~~~~
GET /restconf/data/ietf-system:system-state/clock/current-datetime

FETCH /c
Content-Format: TBD2 (application/yang-identifiers+cbor-seq)

1723                 / current-datetime (SID 1723) /

CBOR: 19 06 bb
~~~~

### URI with a predicate

The predicate of a list entry is converted into the key value, and the
SID of the list and the key are grouped in a CBOR array, in the order
of the key statement.

~~~~
GET /restconf/data/ietf-system:system/ntp/server=tic.nrc.ca

FETCH /c
Content-Format: TBD2 (application/yang-identifiers+cbor-seq)

[1756, "tic.nrc.ca"] / server (SID 1756), name = "tic.nrc.ca" /

CBOR: 82 19 06 dc 6a 74 69 63 2e 6e 72 63 2e 63 61
~~~~

### Node below a list entry

When the target is below a list entry, the predicate is still carried:
the array starts with the SID of the target node, followed by the keys
of the lists crossed by the path, from the outermost to the innermost.

~~~~
GET /restconf/data/ietf-system:system/ntp/server=tic.nrc.ca/udp/address

FETCH /c
Content-Format: TBD2 (application/yang-identifiers+cbor-seq)

[1762, "tic.nrc.ca"] / address (SID 1762) of server "tic.nrc.ca" /

CBOR: 82 19 06 e2 6a 74 69 63 2e 6e 72 63 2e 63 61
~~~~

### Editing a list entry

For a request that carries data, the JSON body is converted into CBOR.
The target, with its predicate, is the key of the map; the names inside
the body are replaced by SIDs, delta-encoded from the SID of their
parent.

~~~~
PUT /restconf/data/ietf-system:system/ntp/server=tic.nrc.ca
{ "ietf-system:server": [ {
    "name": "tic.nrc.ca",
    "prefer": true,
    "udp": { "address": "132.246.11.231" } } ] }

iPATCH /c
Content-Format: TBD3 (application/yang-instances+cbor-seq)

{
  [1756, "tic.nrc.ca"] : {   / server (SID 1756) /
    3 : "tic.nrc.ca",        / name (SID 1759) /
    4 : true,                / prefer (SID 1760) /
    5 : {                    / udp (SID 1761) /
      1 : "132.246.11.231"   / address (SID 1762) /
    }
  }
}

CBOR: a1 82 19 06 dc 6a 74 69 63 2e 6e 72 63 2e 63 61 a3 03 6a 74 69
      63 2e 6e 72 63 2e 63 61 04 f5 05 a1 01 6e 31 33 32 2e 32 34 36
      2e 31 31 2e 32 33 31
~~~~

A deletion is converted into the same map, with the CBOR value
undefined (simple value 23) for the entry.

~~~~
DELETE /restconf/data/ietf-system:system/ntp/server=tac.nrc.ca

iPATCH /c
Content-Format: TBD3 (application/yang-instances+cbor-seq)

{
  [1756, "tac.nrc.ca"] : undefined   / server (SID 1756) /
}

CBOR: a1 82 19 06 dc 6a 74 61 63 2e 6e 72 63 2e 63 61 f7
~~~~

### Notifications {#notifications}

The notifications use the module example-port of {{I-D.ietf-core-comi}},
whose notification example-port-fault (SID 60010) contains the leaves
port-name (SID 60011) and port-fault (SID 60012).

A RESTCONF client receives the notifications of an event stream with
Server-Sent Events, by a GET on the location of the stream, as
specified in {{Section 6 of RFC8040}}. The location of a stream is
chosen by the server, and discovered by the client in the
"ietf-restconf-monitoring" module. The gateway therefore reuses the
paths of the device: the location of an event stream of the device is
its path, "/s" in the examples, and the location of the stream of a
single notification is this path followed by the path of the
notification in the XPath style, for instance
"/s/example-port:example-port-fault". When it receives the GET, the
gateway observes the event stream of the device {{RFC7641}}, and
converts each CORECONF notification into a Server-Sent Event, encoded
in JSON. Since a CORECONF notification does not carry the time of the
event, the gateway sets the "eventTime" to the time at which it receives
the notification.

~~~~
GET /s HTTP/1.1
Host: gateway.example.com
Accept: text/event-stream

  GET /s
  Observe: 0

  2.05 Content
  Observe: 12
  Content-Format: TBD3 (application/yang-instances+cbor-seq)

  {
    60010 : {             / example-port-fault (SID 60010) /
      1 : "0/4/21",       / port-name (SID 60011) /
      2 : "Open pin 2"    / port-fault (SID 60012) /
    }
  }

  CBOR: a1 19 ea 6a a2 01 66 30 2f 34 2f 32 31 02 6a 4f 70 65 6e 20
        70 69 6e 20 32

HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {
data:   "ietf-restconf:notification" : {
data:     "eventTime" : "2026-11-26T10:00:00Z",
data:     "example-port:example-port-fault" : {
data:       "port-name" : "0/4/21",
data:       "port-fault" : "Open pin 2"
data:     }
data:   }
data: }
~~~~

Each new CoAP notification of the device gives one or several
Server-Sent Events on the same HTTP connection. When a CoAP
notification carries several notifications, ordered from the newest
{{I-D.ietf-core-comi}}, the gateway sends them from the oldest to the
newest.

On the stream of a single notification, the gateway converts the path
of the notification into its SID, and observes the event stream of the
device with a FETCH, which only returns this notification.

~~~~
GET /s/example-port:example-port-fault HTTP/1.1
Host: gateway.example.com
Accept: text/event-stream

  FETCH /s
  Observe: 0
  Content-Format: TBD2 (application/yang-identifiers+cbor-seq)

  60010                   / example-port-fault (SID 60010) /

  CBOR: 19 ea 6a
~~~~

A RESTCONF client can also restrict the notifications with the "filter"
query parameter ({{Section 4.8.4 of RFC8040}}). When the XPath
expression only designates notifications, the gateway converts it into
their SIDs, as for the stream of a single notification. A more complex
expression, for instance testing the value of a leaf, cannot be
converted into SIDs: the gateway then observes the whole stream and
applies the filter itself.

# Bootstrap and SID File Discovery {#bootstrap}

Before it can convert a request, the gateway has to know which CORECONF
device is addressed, which YANG modules this device implements, and
which SID files describe them.

A path in the XPath style, as used by RESTCONF, is ambiguous: it
identifies the modules by their names. For IETF YANG data models, the
prefix "ietf-" in a module name can point to an official YANG data
model, but this is not always the case: only the modules published by
the IETF are registered in the IANA "YANG Module Names" registry, and
nothing prevents another organization from using a similar name. On
the contrary, a SID is non-ambiguous: it belongs to a range allocated
to a single organization {{RFC9595}}, and it can easily lead to the
YANG data model or to the SID file, for instance through the DNS
{{I-D.toutain-core-sid-encoding}}.

## Device Discovery {#device-discovery}

{{I-D.ietf-core-comi}} discovers the resources of a CORECONF device with
the CoRE Link Format {{RFC6690}}, through "/.well-known/core". This
discovery suits the RESTCONF side of the gateway, but it is difficult
to put on a constrained device: the CoRE Link Format is a text format,
which is not compact, and it identifies the YANG modules by their
names.

This document defines instead the YANG module "ietf-coreconf-well-known"
({{well-known-module}}). Its list "module" gives the YANG modules
implemented by the device, each identified by its entry-point SID
{{RFC9595}}, that is the SID assigned to the module itself. The SID of
this list is well-known (TBD4): the gateway
retrieves it with a FETCH on the datastore resource of the device, and
determines what the device does without any prior knowledge of it. The
datastore resource is configured in the gateway, and is "/c" by default.

Each entry may also give the revision of the module, so that the
gateway selects the matching SID file. The revision is optional, since
it makes the list larger: when it is absent, the gateway uses the latest
SID file of the module, which is valid for the previous revisions, a SID
never changing its meaning. To stay compact, the revision is a union of
an integer made of the year, the month, and the day of the revision date
(YYYYMMDD), encoded in 5 bytes in CBOR, and of the revision date string
(YYYY-MM-DD), encoded in 11 bytes. Both are encoded without CBOR tag
{{RFC9254}}, the major type distinguishing them.

In the following example, the device implements the revision 2014-08-06
of the module ietf-system, whose SIDs are assigned from 1700, and the
module example-port, whose SIDs are assigned from 60000, without
revision. The SIDs
of the leaves "entry-point" and "revision" are assumed to be TBD4+1 and
TBD4+2.

~~~~
FETCH /c
Content-Format: TBD2 (application/yang-identifiers+cbor-seq)

TBD4                     / module (well-known SID) /

2.05 Content
Content-Format: TBD3 (application/yang-instances+cbor-seq)

{
  TBD4 : [                 / module (well-known SID) /
    {
      1 : 1700,            / entry-point (TBD4+1): ietf-system /
      2 : 20140806         / revision (TBD4+2): 2014-08-06 /
    },
    {
      1 : 60000            / entry-point (TBD4+1): example-port /
    }
  ]
}
~~~~

When the gateway needs the features or the deviations of the
modules, it can read the YANG library
{{I-D.ietf-core-yang-library}} of the device with CORECONF, once the SID
file of the YANG library is known.

## SID File Retrieval {#sid-file-retrieval}

The gateway needs, for each module of the device, the SID file with the
SID extensions {{I-D.toutain-core-sid-extension}}. The sid-extensions
leaf of the header of the file indicates that the items carry type
information and keys. A SID file without this indication does not give
the keys of the lists and the types of the leaves, and cannot be used
to convert the predicates of a URI.

The gateway obtains the SID files in the following order:

1. The SID files stored locally in the gateway, selected with the
   entry-point SID of the module. The files are put there by the
   operator, for instance when the device is installed.

2. When no local file matches, the gateway discovers the SID file with
   the DNS {{I-D.toutain-core-sid-encoding}}. From the entry-point SID
   of the module, the gateway builds the SID DNS name and queries its
   TXT records. The "repository" key gives the location of the SID
   repository from which the file is retrieved. A SID that is not an
   entry point, received for instance in a response of the device, can
   also be resolved: its records give the entry-point SID of its module,
   which is then queried in turn.

3. If the DNS does not return a result, the discovery fails and the
   gateway cannot serve the module. It answers the RESTCONF requests on
   this module with an error.

A SID, once published, never changes its meaning. The gateway can
therefore keep the SID files it retrieved, and the DNS answers, without
having to check them again for the revision of the module.

# Building the /.well-known/core Resource {#well-known-core}

The gateway can build, on behalf of the device, the "/.well-known/core"
resource defined in {{I-D.ietf-core-comi}}, from the information
provided by "ietf-coreconf-well-known". A CORECONF client can then
discover the resources of the device with the CoRE Link Format
{{RFC6690}}, while the device itself only implements the compact list
of its modules.

The gateway builds the following links. The resource types
"core.c.ds", "core.c.dn", and "core.c.yl", and the "ds" attribute, are
defined in {{I-D.ietf-core-comi}}. The resource type "core.c.ym" and the
"ep" attribute are new, and defined by this document.

* The datastore resource (rt="core.c.ds", defined in
  {{I-D.ietf-core-comi}}): the path is the one configured in the
  gateway, "/c" by default, and the "ds" attribute is 1029, the SID of
  the unified datastore.

* The YANG modules (rt="core.c.ym", new in this document): one link for
  each module of the list, described below.

* The data node resources (rt="core.c.dn", defined in
  {{I-D.ietf-core-comi}}): for each module of the list, the gateway
  takes the data nodes of its SID file, and builds the location of each
  of them from its SID, encoded as specified in {{I-D.ietf-core-comi}}.

* The YANG library (rt="core.c.yl", defined in {{I-D.ietf-core-comi}}):
  only when the list contains the entry-point SID of the YANG library
  {{I-D.ietf-core-yang-library}}.

The locations of the data nodes are built from their SIDs, as required
by {{I-D.ietf-core-comi}}, and a client that receives them does not know
which modules the device implements. The gateway therefore adds one
link for each module. Its target is the datastore path followed by the
name of the module and a colon, as in a path in the XPath style, for
instance "/c/ietf-system:". The new resource type "core.c.ym" indicates
that the target is a YANG module, and the new "ep" attribute gives its
entry-point SID, as listed in "ietf-coreconf-well-known". The name is
easy to read, but ambiguous ({{bootstrap}}); the entry-point SID is
non-ambiguous, and leads the client to the YANG module and to the SID
file, for instance through the DNS {{I-D.toutain-core-sid-encoding}}.

~~~~
link-extension    = ( "ep" "=" sid )
                    ; entry-point SID of the module
sid               = 1*DIGIT
~~~~

In the following example, the device implements the two modules of
the example of {{device-discovery}}: ietf-system (entry point 1700),
whose SID file gives the data nodes boot-datetime (SID 1722) and
current-datetime (SID 1723), and example-port (entry point 60000),
whose SID file gives a data node with the SID 60001.

~~~~
GET /.well-known/core?rt=core.c.ds

2.05 Content
Content-Format: 40 (application/link-format)

</c>;rt="core.c.ds";ds=1029

GET /.well-known/core?rt=core.c.ym

2.05 Content
Content-Format: 40 (application/link-format)

</c/ietf-system:>;rt="core.c.ym";ep=1700,
</c/example-port:>;rt="core.c.ym";ep=60000

GET /.well-known/core?rt=core.c.dn

2.05 Content
Content-Format: 40 (application/link-format)

</c/a6>;rt="core.c.dn",
</c/a7>;rt="core.c.dn",
...
</c/Oph>;rt="core.c.dn",
...
~~~~

# YANG Module for the Well-Known List of Modules {#well-known-module}

The tree diagram of the module is:

~~~~
module: ietf-coreconf-well-known
  +--ro module* [entry-point]
     +--ro entry-point    sid:sid
     +--ro revision?      compact-revision
~~~~

~~~~ yang
<CODE BEGINS> file "ietf-coreconf-well-known@2026-11-26.yang"
module ietf-coreconf-well-known {
  yang-version 1.1;
  namespace "urn:ietf:params:xml:ns:yang:ietf-coreconf-well-known";
  prefix cwk;

  import ietf-sid-file {
    prefix sid;
    reference "RFC 9595";
  }

  organization "Personal contribution";
  contact
    "Editor: Javier A. Fernandez
      <mailto:javier-alejandro.fernandez@imt-atlantique.fr>

             Laurent Toutain
             <mailto:laurent.toutain@imt-atlantique.fr>";
  description
    "List of the YANG modules implemented by a CORECONF device,
     each identified by its entry-point SID. The SID of this list
     is well-known, so that a client can retrieve it without any
     prior knowledge of the device.";

  revision 2026-11-26 {
    description
      "Initial revision.";
    reference "draft-imtguy-core-restconf-coreconf-gateway";
  }

  typedef compact-revision {
    type union {
      type uint32 {
        range "10000101..99991231";
      }
      type sid:revision-identifier;
    }
    description
      "Revision of a module, either as an integer made of the
       year, the month, and the day of the revision date
       (YYYYMMDD), or as the revision date string (YYYY-MM-DD).";
  }

  list module {
    key "entry-point";
    config false;
    description
      "YANG module implemented by the device.";
    leaf entry-point {
      type sid:sid;
      description
        "Entry-point SID of the module, that is the SID assigned
         to the module itself.";
    }
    leaf revision {
      type compact-revision;
      description
        "Revision of the module implemented by the device. When
         absent, the client uses the latest SID file of the
         module.";
    }
  }
}
<CODE ENDS>
~~~~
{: #fig-well-known-yang title="YANG module listing the modules of a device"}

# Security Considerations {#security-considerations}

# IANA Considerations {#iana-considerations}

This document requests IANA to:

* register the resource type "core.c.ym" (YANG module) in the
  "Resource Type (rt=) Link Target Attribute Values" registry
  {{RFC6690}};
* register the "ep" attribute in the "Target Attributes" registry
  {{RFC9423}};
* register the namespace
  "urn:ietf:params:xml:ns:yang:ietf-coreconf-well-known" in the "IETF
  XML Registry";
* register the module "ietf-coreconf-well-known" in the "YANG Module
  Names" registry;
* allocate a SID range for this module in the "YANG SID" registry
  {{RFC9595}}, the SID of the list "module" being the well-known
  SID TBD4.

The Content-Formats TBD2 and TBD3 are those of
{{I-D.ietf-core-comi}}.

--- back

# Changes {#changes}

* Initial version.
* {{device-discovery}}: the device no longer needs "/.well-known/core",
  which is not compact and identifies the modules by their names. The
  new module "ietf-coreconf-well-known" ({{well-known-module}}) lists
  the entry-point SIDs of the modules implemented by the device, at a
  well-known SID (TBD4) that the gateway retrieves with a FETCH. The
  YANG library is only read when the features or deviations are
  needed.
* {{notifications}}: the location of a RESTCONF stream reuses the path
  of the event stream of the device ("/s"), followed, for the stream of
  a single notification, by the path of the notification in the XPath
  style ("/s/example-port:example-port-fault"), converted into a FETCH
  with its SID. The "filter" query parameter is kept for the other
  cases.
* {{notifications}}: new example of notifications: a RESTCONF GET on a
  stream with Server-Sent Events is converted into a CoAP Observe of
  the event stream of the device, each CORECONF notification into an
  event with an "eventTime" set by the gateway, and a "filter" designating
  notifications into a FETCH with their SIDs.
* {{device-discovery}}, {{well-known-core}}: the second module of the
  examples is renamed example-port, the module of the notification
  example of {{I-D.ietf-core-comi}}, whose SIDs are in its range.
* {{well-known-core}}: the text states explicitly which resource types
  and attributes are defined in {{I-D.ietf-core-comi}} ("core.c.ds",
  "core.c.dn", "core.c.yl", "ds") and which are new in this document
  ("core.c.ym", "ep").
* {{well-known-core}}: the "ep" attribute is no longer added to each
  data node link; the gateway adds instead one link for each module,
  whose target is the datastore path followed by the module name and a
  colon (e.g., "/c/ietf-system:"), with the new resource type
  "core.c.ym" and the "ep" attribute giving the entry-point SID.
* {{device-discovery}}, {{well-known-core}}: the second module of the
  examples is named other-module, and the "/.well-known/core" example
  shows the data nodes of both modules, with their "ep" attribute.
* {{well-known-core}}: the location of a data node stays built from its
  SID, as required by {{I-D.ietf-core-comi}}, and the optional "ep"
  attribute gives the entry-point SID of its module, which is
  non-ambiguous, unlike a path in the XPath style.
* {{well-known-core}}: new section, the gateway builds the
  "/.well-known/core" resource of {{I-D.ietf-core-comi}} (datastore,
  data nodes, and YANG library links) from the list of modules given by
  "ietf-coreconf-well-known".
* {{device-discovery}}: each entry of the list "module" carries the
  entry-point SID and an optional revision, a union of an integer
  (YYYYMMDD) and of the revision date string, to stay compact. Without
  revision, the gateway uses the latest SID file of the module.
* {{sid-file-retrieval}}: the local SID files are selected with the
  entry-point SID of the module, which is also the starting point of the
  DNS discovery.
* {{iana-considerations}}: namespace, module name, and SID range of
  "ietf-coreconf-well-known".
* {{bootstrap}}: added a paragraph explaining that a path in the XPath
  style is ambiguous, since the "ietf-" prefix of a module name does not
  always point to an official YANG data model, while a SID is
  non-ambiguous and leads to the YANG data model or to the SID file.
