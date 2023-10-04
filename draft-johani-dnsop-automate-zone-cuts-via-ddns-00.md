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
records) should always be kept in sync between child zone and parent
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
scanners" by the parent. 

These scanners detect publication of CDS records (the child signalling
a desire for an update to the DS RRset in the parent) and/or a CSYNC
record (the child signalling a desire for an update to the NS RRset
or, possibly, in-bailiwick glue in the parent.

The scanners have a number of drawbacks, including being inefficient
and slow. They are only applicable to DNSSEC-signed child zones and
they add significant complexity to the parent operations. But given
that, they do work.

Generalized DNS Notifications (see
draft-ietf-dnsop-generalized-notify-00) propose a method to
alleviate the inefficiency and slowness of scanners. But the DNSSEC
requirement and the complexity remain.

This document proposes an alternative mechanism to automate the
updating of delegation information in the parent zone for a child
zone based on DNS Dynamic Updates secured with SIG(0) signatures. 

This alternative mechanism shares the property of being efficient
and provide rapid convergence (similar to generalized notifications in
conjuction with scanners). Furthermore, it has the advantages of not
requiring any scanners in the parent at all and also not being
dependent on the child (and parent) being DNSSEC-signed.

Knowledge of DNS NOTIFY {{!RFC1996}} and DNS Dynamic Updates
{{!RFC2136}} and {{!RFC3007}} is assumed. DNS SIG(0) transaction
signatures are documented in {{!RFC2931}}. In addition this
Internet-Draft borrows heavily from the thoughts and problem statement
from the Internet-Draft on Generalized DNS Notifications [which cannot
be referenced?]

## Requirements Notation

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**",
"**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**",
"**NOT RECOMMENDED**", "**MAY**", and "**OPTIONAL**" in this document
are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# What is a "NOTIFY"?

A NOTIFY (as of {{!RFC1996}}) is a message, in DNS packet format, that
allows one party to notify another that some DNS data elsewhere has
changed. It is only a hint and the recipient may ignore it. But if
the recipient does listen to the NOTIFY it should make its own lookups
to verify what has changed and whether that should trigger any changes
in the DNS data provided by the recipient.

I.e. the NOTIFY is essentially a message that says "some data has
change over there; I suggest that you go and check it out yourself".

## What is a "Generalized NOTIFY"?

This is a proposed extension to the use of {{!RFC1996}} NOTIFY. The
extension covers using NOTIFY(CSYNC) to signal the publication of a CSYNC
record in the child zone (prompting the parent zone to look it up and
make a decison on whether to update the delegation information for the
child zone based upon what it found. Another type is NOTIFY(CDS) which
does the same, except it is used to prompt the parent to decide
whether to update the child DS RRset (or not).

A generalized NOTIFY is typically sent across a zone cut and the
recipient is likely a CSYNC or CDS scanner.

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

DDNS Updates can be used to update any information in a zone (subject
to the policy of the recipient). But in the special case where the
data that is updated is the delegation information for a child zone
and it is sent across a zone cut (i.e. the child sends it to the
parent), it acts as a glorified generalized NOTIFY.

The DDNS update in this case is essentially a message that says "some
data has changed; this is the exact change; here is the proof that the
change is authentic, please verify this signature".

# Terminology

SIG(0)
: An assymmetric signing algorithm that allows the recipient to only
  need to know the public key to verify a signature created by the
  senders private key.

# Updating Delegation Information via DDNS Updates.

This is not a new idea. The functionality to update delegation
information in the parent zone via DDNS has been available for
years in a least one DNS implementation (BIND9). However, while DDNS
is used extensively inside organisations it has not seen much use
across organisational boundaries. And zone cuts, almost by definition,
usually cuts across such boundaries.

When sending a DDNS update it is necessary to know where to send
it. Inside an organisation this information is usually readily
available. But outside the organisation it is not. And even if the
sender would know where to send the update, it is not at all clear
that the destination is reachable to the sender (the parent primary is
likely to be protected by firewalls and other measures). 

This constitutes a problem for using DDNS Updates across zone cuts.

Another concern is that traditionally DDNS updates are sent to a
primary nameserver, and if the update signture verifies the update is
automatically applied to the DNS zone. This is often not an acceptable
mechanism. The recipient may, for good reason, require additional
policy checks and likely an audit trail. Finally, the change should
in many cases not be applied to the running zone but rather to some sort of
provisioning system or a database.

This creates another problem for using DDNS Updates for managing
delegation information.

Both problems are addressed by the proposed mechanism for locating the
recipient of a generalized NOTIFY.

# How to Locate The Target for a generalized NOTIFY.

The generalized notifications I-D proposes a new RR type, tentatively
with the mnemonic NOTIFY that has the following format:

	{parent zone}   IN  NOTIFY  {RRtype} {scheme} {port} {target}

where {parent zone} is the domain name of the parent zone and
{target} is the domain name of the recipient of the NOTIFY. {RRtype}
is typically "CDS" or "CSYNC" in the case where delegation
information should be updated (there are also other uses of
generalized notifications). Finally, {scheme} is a number to indicate
the type of notification mechanism to use. Scheme=1 is defined as
"send a generalized NOTIFY".

Example for where children of `parent.` should send NOTIFY(CSYNC):

	parent. IN NOTIFY CSYNC 1 5301 csync-scanner.parent.

This record is looked up by the child primary nameserver at the time
that the child primary is about to publish a new CSYNC record in the
child zone (or the equivalent for a CDS). The interpretation is:

`Send a NOTIFY(CSYNC) to csync-scanner.parent. on port 5301`

# How to Locate the Target for a DDNS Update.

This document proposes the definition of a new {scheme} for the same
record that is used for generalized NOTIFY. Scheme=2 is here defined
as "send a DDNS Update".

Example:

	parent.  IN NOTIFY ANY 2 5302 ddns-receiver.parent.

This record is looked up by the child primary nameserver at the time
that the delegation information for the child zone changes (typically
causing the child to publish a new CSYNC record and/or a new CDS
record). The interpretation is:

`Send a DDNS Update to ddns-receiver.parent. on port 5302`

# Interpretation of the response to the DDNS Update.

All DNS transactions are designed as a pair of messages and this is
true also for DDNS updates. The interpretation of the different
responses to DDNS updates are already fully documented in
{{!RFC2136}}, section 2.2. In particular a response with rcode=5
("REFUSED") should be interpreted as a permanent signal that DDNS
updates are not supported by the receiver.

The case of no response is more complex, as it is not possible to know
whether the DDNS update actually reached the reciever (or was lost in
transit) or the response was not sent (or lost in transit).

For this reason it is suggested that a lack of response is left as
implementation dependent. That way the implementation has sufficient
freedom do chose a sensible approach. Eg. if the sender of the DDNS
Update (like the primary nameserver of the child zone) only serves a
single child, then resending the DDNS Update once or twice may be ok
(to ensure that the lack of response is not due to packets being lost
in transit). On the other hand, if the sender serves a large number of
child zones below the same parent zone, then it may already know that
the receiver for the DDNS Updates is not responding for any of the
child zones, and then resending the update immediately is likely pointless.

# What is the Use Case? 

Because of the drawbacks of CDS and CSYNC scanners they are unlikely
to be able to fully automate the maintenance of delegation information
in all parent zones. The primary reasons are the hard requirement on
DNSSEC in the child zones and the complexity cost of operating the
scanner infrastructure. In practice, scanners are likely only
realistic for parent zones that are operated by well-resourced
registries.

All the parts of the DNS name space where the parent is smaller and 
more resource constrained would be able to automate the delegation 
management via this mechanism without the requirement of operating 
scanners. Also all parts of the name space where there are child zones 
that are not DNSSEC-signed would be able to use this. 

Obviously, also well-resourced parent zones with DNSSEC-signed child
zones would be able to use this DDNS-based mechanism, but in those
cases scanners plus generalized notifications would also work. 

# Security Considerations.

Any fully automatic mechanism to update the contents of a DNS zone
opens up a potential vulnerability should the mechanism not be
implemented correctly. 

In this case the definition of "correct" is a question for the
receiver of the DDNS Update. The receiver should validate the
authenticity of the update and then do the same checks and
verifications as a CDS or CSYNC scanner does. The difference from the
scanner is only in the validation: single SIG(0) signature by a key
that the receiver trusts vs DNSSEC signature that chains back to a
DNSSEC trust anchor that a scanner trusts.

# IANA Considerations.

None.

-------

# Acknowledgements.

* Peter Thomassen and I together came up with the location mechanism
  for the generalized notifications, which this draft relies upon.

* Mark Andrews provided the initial inspiration for writing some code
  to experiment with the combination of the location mechanism from
  the generalised notifications with DDNS Updates across zone cuts.

--- back

# Change History (to be removed before publication)

* draft-johani-dnsop-automate-zone-cuts-via-ddns-00 

> Initial public draft. 

