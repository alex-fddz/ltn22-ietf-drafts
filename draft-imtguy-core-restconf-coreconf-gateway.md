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
  RFC6690:
  RFC8132:
  RFC8174:
  RFC9254:
  RFC9595:

informative:
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

# Bootstrap and SID File Discovery {#bootstrap}

Before it can convert a request, the gateway has to know which CORECONF
device is addressed, which YANG modules this device implements, and
which SID files describe them.

## Device Discovery {#device-discovery}

The gateway discovers the resources of a CORECONF device with the
CoRE Link Format {{RFC6690}}, as described in {{I-D.ietf-core-comi}}. A
request on "/.well-known/core" with the resource type "core.c.ds"
returns the datastore resource of the device, and a request with the
resource type "core.c.yl" returns the root resource of its YANG library
{{I-D.ietf-core-yang-library}}.

The YANG library lists the modules, with their revisions, the features
and the deviations implemented by the device. The gateway uses it to
select, for each module, the SID file that matches the revision of the
device, and to build the part of the RESTCONF data tree that it exposes
for this device.

## SID File Retrieval {#sid-file-retrieval}

The gateway needs, for each module of the device, the SID file with the
SID extensions {{I-D.toutain-core-sid-extension}}. The sid-extensions
leaf of the header of the file indicates that the items carry type
information and keys. A SID file without this indication does not give
the keys of the lists and the types of the leaves, and cannot be used
to convert the predicates of a URI.

The gateway obtains the SID files in the following order:

1. The SID files stored locally in the gateway, selected with the module
   name and revision. The files are put there by the operator, for
   instance when the device is installed.

2. When no local file matches, the gateway discovers the SID file with
   the DNS {{I-D.toutain-core-sid-encoding}}. From a SID of the module,
   the gateway builds the SID DNS name and queries its TXT records. If
   the SID is not the entry point of the module, the records give the
   entry-point SID, which is then queried in turn. The "repository" key
   gives the location of the SID repository from which the file is
   retrieved.

3. If the DNS does not return a result, the discovery fails and the
   gateway cannot serve the module. It answers the RESTCONF requests on
   this module with an error.

A SID, once published, never changes its meaning. The gateway can
therefore keep the SID files it retrieved, and the DNS answers, without
having to check them again for the revision of the module.

# Security Considerations {#security-considerations}

# IANA Considerations {#iana-considerations}

This document has no IANA actions.

--- back

# Changes {#changes}

* Initial version.
