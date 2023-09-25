---
title: "Automating DNS Delegation Information via DDNS"
abbrev: DDNS Updates of Delegation Information
docname: draft-johani-dnsop-automate-zone-cuts-via-ddns-00
date: {DATE}
category: std

ipr: trust200902
area: Internet
workgroup: DNSOP Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: J. Stenstam
    name: Johan Stenstam
    organization: The Swedish Internet Foundation
    email: johan.stenstam@internetstiftelsen.se

normative:

informative:

--- abstract

Delegation information (i.e. the NS RRset, possible glue, possible DS
rrecords) should always be kept in sync between child zone and parent
zone. However, in practice that is not always the case.

When the delegation information is not in sync the child zone is
usually working fine, but without the amount of redundancy that the
zone owner likely expects to have. Hence, should any further problems
ensue it could have catastropic consequences.

The DNS name space has lived with this problem for decades and it
never goes away. Or, rather, it will never go away until a fully
automated mechanism for how to keep the information in sync
automatically is deployed.

This document proposes such a mechanism.

TO BE REMOVED: This document is being collaborated on in Github at:
[https://github.com/johanix/draft-johani-dnsop-automate-zone-cuts-via-ddns](https://github.com/johanix/draft-johani-dnsop-automate-zone-cuts-via-ddns).
The most recent working version of the document, open issues, etc. should all be
available there.  The authors (gratefully) accept pull requests.

--- middle

# Introduction

For DNSSEC signed child zones with TLD registries as parents there is
an emerging trend towards running so-called "CDS scanners" and "CSYNC
scanners" in the parent. 

These scanners detect publication of CDS
records (the child signalling a desire for an update to the DS RRset
in the parent) and/or a CSYNC record (the child signalling a desire
for an update to the NS RRset or, possibly, in-bailiwick glue in the
parent.

Thses scanners have a number of drawbacks, inluding being inefficient
and slow. They are only applicable to DNSSEC-signed child zones and
they add significant complexity to the parent operations. But given
that, they do work.

Generalized DNS Notifications (see
draft-thomassen-dnsop-generalized-dns-notify-NN) propose a method to
alleviate the inefficiency and slowness. But the DNSSEC-only
requirement and the complexity remain.

## Requirements Notation

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**",
"**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**",
"**NOT RECOMMENDED**", "**MAY**", and "**OPTIONAL**" in this document
are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# What is a "NOTIFY"?

A NOTIFY (as of RFC1996) is a message, in DNS packet format, that
allows one party to notify another that some DNS data elsewhere has
changed. It is only a hint and the recipient may ignore it. But if
the recipient does listen to the NOTIFY it should make its own lookups
to veryfy what has changed and whether that should trigger any changes
in the DNS data maintained by the recipient.

I.e. the NOTIFY is essentially a message that says "some data has
change over there; I suggest that you go and check it out yourself".

# What is a "DNS Dynamic Update"?

A DNS Dynamic Update is a message, in DNS packet format, that allows
one party to notify another that some DNS data under the recipients
management should change. The difference to the NOTIFY is that the
dynamic update contains the exact change that should (in the opinion
of the sender) be applied to the recipients DNS data. Furthermore, for
*secure* Dynamic Updates, the message also contains proof why the
update should be trusted (in the form of a digital signature by a key
that the recipient trusts).

In this document the term "Dynamic Update" or "DDNS Update" implies
*secure* dynamic update. Furthermore this document implies that the
signature algorithms are always based on assymetric crypto keys, using
the same algorithms as are being used for DNSSEC. I.e. by using the
right algorithm the resulting signatures will be as strong as
DNSSEC-signatures.

# Terminology

SIG(0)
: An assymmetric signing algorithm that allows the recipient to only
  need to know the public key to verify a signature created by the
  senders private key.

# Updating Delegation Information via DDNS Updates

This is not a new idea. This functionality has been available for
years in a least one DNS implementation (BIND9). However, while used
extensively inside organisations it has not seen much use across
organisational boundaries. And zone cuts, almost by definition,
usually cuts across such boundaries.

When sending a DDNS update it is necessary to know where to send
it. Inside an organisation this information is usually readily
available. But outside it is not. And even if the sender would know
where to send the update, it is not at all clear that the destination
is reachable to the sender (the parent primary is likely to be
protected by firewalls and other measures). 

This constitutes a problem for using DDNS Updates across zone cuts.

Another concern is that traditionally DDNS updates are sent to a
primary nameserver, and if the update signture verifies the update is
automatically applied to the DNS zone. This is often not an acceptable
mechanism. The recipient may, for good reason, require additional
policy checks and likely an audit trail. Finally, the change should
in many cases not be applied to the running zone but rather to some sort of
provisioning system.

This creates another problem for using DDNS Updates for managing
delegation information.

# Interface to the Registry

Interface to the Registry System must be done according to
standardized protocols. This requirement has the following
consequences:

* Zone data must be retrieved from the Registry System using AXFR and
  IXFR {{!RFC5936}}.

* The request for publication of new zone data must be notified with DNS
  NOTIFY {{!RFC1995}}.

* Zone data published by the Registry System must contain a checksum in
  the form of ZONEMD {{!RFC8976}}.

# Local Updates

During normal operation, no changes to the zone data retrieved from
the Registry may take place. However, there may be situations where
the Registry is not reachable (nor is it expected to be reachable
within a reasonable time) and where local updates of zone data must be
able to be carried out. This can for example, be redirection of
socially important infrastructure.

In a crisis situation (emergency operation), zone updates must be able
to take place locally. Updates that take place in this way are
introduced into regular systems as soon as possible. Return to normal
operation may only take place after all changes made during emergency
operation have been introduced into the regular system.

Local updates must be applied using a strict and traceable method. It
must be clear at all times whether local updates have been applied,
what these are and who requested them.

In cases where local updates have taken place, ZONEMD MUST be updated.

N.B. Local updates are an extraordinary measure and must not be used
except in emergency situations. Procedures for who may request these
are decided by the Internet foundation's crisis management.

# Ingress Verification

Before signing, a number of checks must be performed on the zone
contents. The reason why checking must take place before signing is to
ensure that the zone being signed is always correct and can thus
continue to be re-signed in the event of problems upstream. The exact
checks to be carried out are set out in a separate specification and
are subject to local policy.

## Requirements on ingress verification

* The ingress verification must prevent an updated incorrect zone from
  being signed. An already approved previous version of the zone must
  continue to be signed until a new zone is approved.

* New zone controls must be able to be added without the component for
  ingress verification of zone data needing to be redesigned

* The interface from the zone pipeline to the ingress verification
  function must be DNS AXFR. This means that all controls must
  logically sit to the side and not be part of the critical
  path. I.e. the verification code is not part of the zone pipeline.

* All zone controls, self-developed or imported, must have local
  ownership within the organization.

## Examples of ingress verification checks

* Check that the zone data is complete.

* Check that the ZONEMD, which has been generated by the registry system, is correct

* Check that delegation information for the zone itself is correct.

* Check that the delta (i.e. the difference) between the current zone
  and the previous version is within approved limits.

* Check that certain crucial records are present and correct (the
  zone’s SOA record is one example).

# Signing

The task of the signing step is to keep an approved and received zone
signed for an arbitrary length of time until a new zone is received
from upstream (i.e. from Registry via Ingress Verification).

## Key management

The following requirements apply to the management of cryptographic
keys for signing zone data:

* The key material must be stored and used
  in an HSM that meets the security requirements set by the CISO.

* The interface for accessing the key material SHALL be PKCS#11.

* All keys must, to facilitate replication between different signing
  entities, be generated in advance.In order to facilitate replication
  between different signing entities, all keys must be generated in
  advance.

* Exchange of KSK can be initiated automatically or manually, and is
  automatically terminated when the DS record in the parent zone is
  updated (after according to an appropriate safety margin).

* Changing the KSK may only be completed if the DS record in the
  parent zone is updated.

* Replacement of ZSK must be done automatically.

## Zone signing

The following requirements apply to signing zone data:

* Signing must be done using key material via PKCS#11.

* The signing function must support algorithm rollover, e.g,. from RSA/SHA-256 to ECC/P-256/SHA-256.

* Signing must be done with either NSEC or NSEC3 semantics.

* If NSEC3 salt is used, the salt must be periodically changed automatically.

* Change of DNSSEC signing semantics from NSEC to NSEC3 and vice versa
  must be possible automatically.

* Previous signatures that are valid within a configurable time window
  shall be reused when re-signing, in order to reduce the rate of
  change on the zone.

* The signing function shall recreate the ZONEMD RR per {{!RFC8976}} after
  the zone has been signed.

# Egress Verification

After signing, several checks must be performed on the zone contents.
Apart from the obvious validation of generated DNSSEC signatures it is
also important to ensure that the signing step only added DNSSEC-
information without in any way modifying the unsigned data.

## Requirements on egress verification

* All generated DNSSEC signatures (RRSIG records) must be validated.

* The NSEC (or NSEC3) chain must be verified to be complete.

* The non-DNSSEC content of the signed zone must be provably identical
  to the corresponding unsigned zone that entered the signing step.

## Examples of egress verification checks

* Verify the zone with other software than was used for signing.

* Remove all data added by the signing step and check the ZONEMD value over that data with the ZONEMD from before the signing.

# Distribution

The following requirements apply to distribution of the signed zone:

* The signed zone must be retrieved from signing and egress
  verification to the distribution points with AXFR (not IXFR).
  
  It must be AXFR, as otherwise it is not possible to switch from one
  upstream to another in the distribution points (the IXFR from one
  upstream will not apply cleanly to a zone from another upstream).

* The signed zone shall be distributed to the designated authoritative
  name server services using AXFR/IXFR.

* To reduce convergence time towards the public Internet, the signed
  zone must be distributed with IXFR as far as possible.

* At least two complete zone publishing chains must be operational and
  always active.
  
  This is a policy decision, based on the strong intent to avoid all 
  single points of failure, including in the signing operation.

* Choice of zone publishing chain is an active configuration choice
  in each distribution point and must always be the same for all
  distribution points.

* The signed zone from the selected zone publishing chain must be
 retrieved by all distribution points in all operating facilities.


# Resulting Design Consequences

* The requirement on being able to prove that no unsigned data has
  been modified during signing is most efficiently fullfilled by
  computing the ZONEMD checksum on the unsigned data after signing
  (i.e. the signed zone modulo the DNSSEC related records DNSKEY,
  RRSIG. NSEC, NSEC3, NSEC3PARAM, apex CDS and CDNSKEY) and comparing
  that to the ZONEMD checksum for the corresponding unsigned zone.

* The ZONEMD checksums need to be stored outside the zone pipeline,
  indexed by the unsigned zone that each checksum corresponds to.

* The signed zone may (and will) change the SOA Serial independently
  of the unsigned zone. For this reason the ZONEMD checksums can not
  be stored using the SOA Serial as the index. Therefore a separate,
  unique, identifier is attached to each new version of the zone as a
  TXT record. A UUID is used as the identifier.

* Each unsigned zone MUST have a ZONEMD and a UUID index to store it
  under. Therefore changes to the unsigned zone via the local update
  facility must update the UUID in addition to any other change that
  is executed.

* The Registry runs multiple, parallel zone pipelines for the same
  zone with the requirement to be able to switch which pipeline is
  "active" at any time. As the local update facility is responsible
  for updating the UUID if a local change is needed, the same UUID
  will identify the exact same zone in all pipelines.

* The requirement that the zone pipeline only consists of proven and
  widely used software forces all local and custom software (including
  various tests and verification modules) to be located outside the
  zone pipeline. This cause a need for a component inside the zone
  pipeline with the ability to call an external "verifier" for
  verifications. At present only one such component is known (the
  authoritative nameserver NSD with its "verify:" attribute), but we
  hope that there will be more alternatives in the future.

# Security Considerations

The correct generation and DNSSEC signing of a TLD zone is a matter of
significant importance. As such requirements and methods for
verification of correctness is an important matter that has previously
not been much publically discussed.

As a requirements specification this document aims to make this topic
more public and visible with the intent of improving the correctness
of the requirements via public review.

# IANA Considerations

None

-------

# Acknowledgements



--- back

# Change History (to be removed before publication)

* draft-johani-tld-zone-pipeline-00 

> Initial public draft. 

* draft-johani-tld-zone-pipeline-01

> Security and IANA Considerations sections added.
> Minor typos fixed. 
