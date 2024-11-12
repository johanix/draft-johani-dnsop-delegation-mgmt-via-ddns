---
title: "Automating DNS Delegation Management via DDNS"
abbrev: DDNS Updates of Delegation Information
docname: draft-johani-dnsop-delegation-mgmt-via-ddns-04
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
    country: Sweden
    email: johan.stenstam@internetstiftelsen.se
 -
    ins: E. Bergström
    name: Erik Bergström
    organization: The Swedish Internet Foundation
    country: Sweden
    email: erik.bergstrom@internetstiftelsen.se
 -
    ins: L. Fernandez
    name: Leon Fernandez
    organization: The Swedish Internet Foundation
    country: Sweden
    email: leon.fernandez@internetstiftelsen.se

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
[https://github.com/johanix/draft-johani-dnsop-delegation-mgmt-via-ddns](https://github.com/johanix/draft-johani-dnsop-delegation-mgmt-via-ddns).
The most recent working version of the document, open issues, etc, should all be
available there.  The author (gratefully) accept pull requests.

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

{{?I-D.ietf-dnsop-generalized-notify}} proposes a method to alleviate  
the inefficiency and slowness of scanners. But the DNSSEC requirement
and the complexity remain.

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
from the Internet-Draft on Generalized DNS Notifications (work in progress).

## Requirements Notation

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**",
"**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**",
"**NOT RECOMMENDED**", "**MAY**", and "**OPTIONAL**" in this document
are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Is there a Use Case?

Because of the drawbacks of CDS and CSYNC scanners they are unlikely
to be able to fully automate the maintenance of delegation information
in all parent zones. The primary reasons are the hard requirement on
DNSSEC in the child zones and the complexity cost of operating the
scanner infrastructure. In practice, scanners are likely mostly
realistic for parent zones that are operated by well-resourced
registries.

All the parts of the DNS name space where the parent is smaller and
more resource constrained would be able to automate the delegation
management via this mechanism without the requirement of operating
scanners. Also all parts of the name space where there are child zones
that are not DNSSEC-signed would be able to use this.

Obviously, also well-resourced parent zones with DNSSEC-signed child
zones would be able to use this DNS UPDATE-based mechanism, but in those
cases scanners plus generalized notifications would also work.

# DNS NOTIFY versus DNS UPDATE

DNS NOTIFY and DNS UPDATE messages share several properties
and are used to address similar issues. 

## Similarities between NOTIFY and UPDATE

Both NOTIFY and UPDATE are "push" rather than "pull" messages and
therefore very efficient. 

Both NOTIFY and UPDATE are messages, in DNS packet format. They are
used by one party (the sender) to inform another party (the recipient)
that some piece of DNS information has changed and that as a
consequence of this change the recipient of the message may want to
also make a change to its DNS data.

A NOTIFY (as per {{!RFC1996}}) is only a hint and the recipient may
ignore it. But if the recipient does listen to the NOTIFY it should
make its own lookups to verify what has changed and whether that
should trigger any changes in the DNS data provided by the recipient.

## Differences between NOTIFY and UPDATE

The difference between the UPDATE and the NOTIFY is that the UPDATE
contains the exact change that should (in the opinion of the sender)
be applied to the recipients DNS data. Furthermore, for secure Dynamic
Updates, the message also contains proof why the update should be
trusted (in the form of a digital signature by a key that the
recipient trusts).

In this document the term "Dynamic Update" or "DNS UPDATE" implies
secure dynamic update. Furthermore this document implies that the
signature algorithms are always based on asymmetric crypto keys, using
the same algorithms as are being used for DNSSEC. I.e. by using the
right algorithm the resulting signatures will be as strong as
DNSSEC-signatures.

DNS UPDATEs can be used to update any information in a zone (subject
to the policy of the recipient). But in the special case where the
data that is updated is the delegation information for a child zone
and it is sent across a zone cut (i.e. the child sends it to the
parent), it acts as a glorified generalized NOTIFY.

The DNS UPDATE in this case is essentially a message that says:

    "the delegation information for this child zone has changed; here
    is the exact change; here is the proof that the change is
    authentic, please verify this signature"

# Terminology

SIG(0)
: An assymmetric signing algorithm that allows the recipient to only
  need to know the public key to verify a signature created by the
  senders private key.

# Updating Delegation Information via DNS UPDATEs.

This is not a new idea. There is lots of prior art and prior
documents, including the expired I-D.andrews-dnsop-update-parent-zones-04.

The functionality to update delegation information in the parent zone
via DNS UPDATE has been available for years in a least one DNS
implementation (BIND9). However, while DNS UPDATE is used extensively
inside organisations it has not seen much use across organisational
boundaries. And zone cuts, almost by definition, usually cuts across
such boundaries.

When sending a DNS UPDATE it is necessary to know where to send
it. Inside an organisation this information is usually readily
available. But outside the organisation it is not. And even if the
sender would know where to send the update, it is not at all clear
that the destination is reachable to the sender (the parent primary is
likely to be protected by firewalls and other measures).

This constitutes a problem for using DNS UPDATES across zone cuts.

Another concern is that traditionally DNS UPDATEs are sent to a
primary nameserver, and if the update signture verifies the update is
automatically applied to the DNS zone. This is often not an acceptable
mechanism. The recipient may, for good reason, require additional
policy checks and likely an audit trail. Finally, the change should in
many cases not be applied to the running zone but rather to some sort
of provisioning system or a database.

This creates another problem for using DNS UPDATEs for managing
delegation information.

Both problems are addressed by the proposed mechanism for locating the
recipient of a generalized NOTIFY.

# Locating the Target for a generalized NOTIFY and/or DNS UPDATE.

Section 3 of {{?I-D.ietf-dnsop-generalized-notify}} proposes a new RR
type, tentatively with the mnemonic DSYNC, that has the following
format:

    {qname}   IN  DSYNC  {RRtype} {scheme} {port} {target}

where {target} is the domain name of the recipient of the NOTIFY
message. {RRtype} is typically "CDS" or "CSYNC" in the case where
delegation information should be updated (there are also other uses of
generalized notifications). 

Finally, {scheme} is an 8 bit unsigned integer to indicate the type of
notification mechanism to use. Scheme=1 is defined "NOTIFY", as in
"send a generalized NOTIFY to {target} on port {port}".

This document proposes the definition of a new {scheme} for the same
record that is used for generalized NOTIFY. Scheme=2 is here defined
as "UPDATE", as in "send a DNS UPDATE to {target} on port {port}".
When parsing or presenting DNS zone data the 8 bit unsigned integer
"2" should be replaced by the string "UPDATE", as in the examples
below.

Apart from defining a new scheme to specify the mechanism "UPDATE"
(rather than the mechanism "NOTIFY") this document does not say
anything about what Qname to look up or what RR type. The UPDATE
mechanism should use exactly the same method of locating the target of
the UPDATE as is used for generalized NOTIFY.
 
This lookup addresses the first issue with using DNS UPDATE across
organizational boundaries.

Example 1: a parent zone announces support for DNS UPDATE as a
mechanism for delegation synchronization for all child zones:

    _dsync.parent.  IN DSYNC ANY 2 5302 ddns-receiver.parent.
 
Example 2: a parent zone announces support different DNS UPDATE
targets on a per-child basis 

    childA._dsync.parent.  IN DSYNC ANY 2 5302 ddns-receiver.registrar1.
    childB._dsync.parent.  IN DSYNC ANY 2 5302 ddns-receiver.registrar3.
    childC._dsync.parent.  IN DSYNC ANY 2 5302 ddns-receiver.registrar2.
 
The DSYNC RRset is looked up, typically by the child primary name
server or by a separate agent for the child, at the time that the
delegation information for the child zone changes in some way that
would prompt an update in the parent zone. When the {scheme} is
"UPDATE" (i.e. the number 2 in the wire protocol) the interpretation
is:

`Send a DNS UPDATE to the IP address for the name {target} on port
5302, where {target} is the domain name in the right-hand side of the
DSYNC record that matches the qname in the DNS query.`

# Limitation of Scope for the Proposed Mechanism 

DNS UPDATE is in wide use all over the world, for all sorts of
purposes. It is not in wide use across organizational boundaries. This
document only address the specific case of a child zone that makes a
change in its DNS delegation information that will require an update
of the corresponding information in the parent zone. This includes:

* changes to the NS RRset 
* changes to glue (if any) 
* changes to the DS RRset (if any) 

Only for those specific cases is the described mechanism proposed. 

# The DNS UPDATE Receiver

While the simplest design is to send the DNS UPDATEs to the primary
name server of the parent it will in most cases be more interesting to
send them to a separate UPDATE Receiver. To separate the primary name
server from the UPDATE Receiver, use a {target} with addresses separate
from the addresses of the primary name server.

## Processing the UPDATE in the DNS UPDATE Receiver 

The receiver of the DNS UPDATE messages should implement a suitably 
strict policy for what updates are accepted (typically only allowing 
updates to the NS RRset, glue and DS RRset). 

Furthermore, it is strongly recommended that the policy is further
tightened by only allowing updates to the delegation information of a
child zone with the exact same name as the name of the SIG(0) key the
signed the UPDATE request. I.e. an UPDATE request for the delegation
information for the zone `child.parent.` should only be processed if
it is signed by a SIG(0) key with the name `child.parent.` and the
signature verifies correctly.

Once the DNS UPDATE message has been verified to be correctly signed
by a known and trusted key with the correct name and also adhere to
the update policy it should be subjected to the same set of
correctness tests as CDS/CSYNC scanner would have performed. If these
requirements are also fulfilled the change may be applied to the
parent zone in whatever manner the parent zone is maintained (as a
text file, data in a database, via and API, etc).

# Interpretation of the response to the DNS UPDATE.

All DNS transactions are designed as a pair of messages and this is
true also for DNS UPDATE. The interpretation of the different
responses to DNS UPDATE are fully documented in {{!RFC2136}}, section
2.2.

## RCODE NOERROR

A response with rcode=0 ("NOERROR") should be interpreted as a
confirmation that the DNS UPDATE has been received and
accepted. I.e. the change to the parent DNS data should be expected to
be published in the parent zone at some future time.

## RCODE REFUSED

A response with rcode=5 ("REFUSED") should be interpreted as a
permanent signal that DNS UPDATEs are not supported by the
receiver. This would indicate a parent misconfiguration, as the UPDATE
should not be sent unless the parent has announced support for DNS
UPDATE via publication of an appropriate target location record.

## RCODE BADKEY

A response with rcode=17 ("BADKEY") should be interpreted as a
definitive statement that the DNS UPDATE Receiver does not have access
to the public SIG(0) key needed for signature verification. In this
case the child should fall back to bootstrap of the SIG(0) public key
into the DNS UPDATE Receiver, see below.

## No response to a DNS UPDATE

The case of no response is more complex, as it is not possible to know
whether the DNS UPDATE actually reached the reciever (or was lost in
transit) or the response was not sent (or lost in transit).

For this reason it is suggested that a lack of response is left as
implementation dependent. That way the implementation has sufficient
freedom do chose a sensible approach. Eg. if the sender of the DNS
UPDATE message (like the primary name server of the child zone) only
serves a single child, then resending the DNS UPDATE once or twice may
be ok (to ensure that the lack of response is not due to packets being
lost in transit). On the other hand, if the sender serves a large
number of child zones below the same parent zone, then it may already
know that the receiver for the DNS UPDATEs is not responding for any
of the child zones, and then resending the update immediately is
likely pointless.

# Management of SIG(0) Public Keys

Only the child should have access to the SIG(0) private key. The
corresponding SIG(0) public key should preferably be published in DNS,
but it doesn't have have to be. The SIG(0) public key only needs to be
available to the parent DNS UPDATE Receiver. Keeping all the public
SIG(0) keys for different child zones in some sort of database is
perfectly fine.

## Bootstrapping the SIG(0) Public Key Into the DNS UPDATE Receiver

Bootstrap of the child public SIG(0) key to be trusted by the UPDATE
Receiver may be done either manually or automatically. Manually
may in various cases be the preferred method, especially in the case
of non-registry parents with a small number of child delegations.

If the UPDATE Receiver only supports manual bootstrap, then that is
what will happen (apart from informing the child about this
policy). If the child wants to enforce manual bootstrap it needs to
request this from the UPDATE Receiver.

In those cases there is by definition some mechanism in place to
communicate information from the child to the parent, be it email, a
web form, pieces of paper or something else. The same mechanism can be
extended to also be used to communicate the initial SIG(0) public key
from the child to the parent.

Regardless of whether manual bootstrap or automatic bootstrap is to be
used (subject to the UPDATE Receiver policy), the bootstrap must be
initiated. This is done by the child issuing a self-signed DNS UPDATE
to the parent containing:

    DEL child.parent. {ttl} ANY KEY 
    ADD child.parent. {ttl} IN  KEY ... 

The first record is an instruction to delete any previous keys for 
this child (CLASS=ANY in a DNS UPDATE is a request to delete an entire
RRset). The second is an instruction to add the new key.

When receiving such a message (where the self-signature validates) the
parent UPDATE Reciever SHOULD mark that key as "known" (but not yet
trusted) and then either put that key into a queue for later look up
and validation of the corresponding KEY record (if supporting
automatic bootstrap) or put that key into a queue for subsequent
manual validation and verification.

### Signalling That Manual Bootstrap is Required or Requested

{{?I-D.berra-dnsop-keystate}} describes a mechanism by which the
child may communicate "Key State" to the UPDATE Receiver. If the child
supports this mechanism then a KeyState OPT MUST be included in the
initial key upload (the self-signed UPDATE containing the public
key). The child MAY include a KeyState OPT containing the KEY-STATE for
"Manual bootstrap requested" (value=1). The UPDATE Receiver SHOULD
honour this request and not perform automatic bootstrap for this child
and this key.

Likewise, in the response to the initial key upload, if the UPDATE
contained a KeyState OPT, then the UPDATE Receiver has the ability to
signal the requirement that the child SIG(0) key is manually
bootstrapped (verified) by including a KeyState OPT containing the
KEY-STATE for "Manual bootstrap required" (value=11). If a KeyState OPT
from the UPDATE Receiver is included in the response then the child
MUST validate this message using the SIG(0) public key of the UPDATE
Receiver. If the response is unsigned it MUST be ignored. If the
response valdiates then the child SHOULD signal this information to an
operator, to resolve manually. 

### Automatic Bootstrap of the SIG(0) Public Key

Automated bootstrapping is also possible, subject to the policy of the
UPDATE Receiver. The basic idea is to publish the public SIG(0) key as
a KEY record at the child apex prior to the initial key upload. Then
the UPDATE Receiver (or an agent) may look that KEY up for subsequent
validation.

### Automatic Bootstrap When Child Zone is DNSSEC-signed

If the child zone is DNSSEC-signed (including a signed delegation via
a DS record), then the KEY record containing the SIG(0) public key can
be looked up and validated by the DNS UPDATE Receiver.

    child.parent. IN KEY ...
    child.parent. IN RRSIG KEY ...

In case of validation success the key SHOULD be promoted to "trusted"
by the UPDATE Receiver. At this point any old keys should be deleted.

In case of validation failure (or absence of a DNSSEC signature) the
SIG(0) SHOULD NOT be promoted to the set of keys that the UPDATE
Receiver trusts and any old keys MUST be kept.

### Automatic Bootstrap When Child Zone is unsigned

The bootstrap problem in the unsigned case is essentially identical to
the "automatic DNSSEC Bootstrap via CDS" service a la {{!RFC8078}}
that multiple TLD registries provide today. 

Hence, to bootstrap the public SIG(0) key for a child zone it is
possible for the parent use a "bootstrap policy" a la:

* Look up the KEY RRset in the child zone. Compare to the child KEY
  received in the self-signed DNS UPDATE.

* To mitigate possible spoofing, do the look up of the child KEY RRset
  from multiple vantage points, at multiple times. The child KEY RRset
  must be consistent over time and space.

* Make spoofed responses even more difficult by adding a requirement
  to use a more stable transport, like TCP (or in the future DOT, DOH
  or DOQ once those become more generally available for DNS queries to
  authoritative servers) in addition to UDP. The child KEY RRset must
  be consistent over all tested transports.

* If the received KEY RRset is consistent from multiple vantage points
  and multiple times then it is considered authentic and promoted by
  the parent's UPDATE Receiver from "known" to "trusted" SIG(0) key
  for the child. At this point any old SIG(0) public keys for the
  child should be deleted.

Should a "registry" parent want to support this mechanism (as a
service to its unsigned children) then a likely model is that the
target of the DNS UPDATE is operated by the registrar (or possibly
that the DNS UPDATE is forwarded to the registrar). The registrar
performs its normal verifications of a change and then transforms the
update into an EPP {{!RFC5730}} transaction to communicate it to the
registry.

## Rolling the SIG(0) Key 

Once the parent (or registrar) DNS UPDATE Receiver has the key, the
child can update it via a DNS UPDATE just like updating the NS RRset,
the DS RRset or the glue in the parent zone (assuming a suitable DNS
UPDATE policy in the parent). I.e. only the initial bootstrapping of
the key is an issue.

Note, however, that the alternative of re-bootstrapping (by whatever
bootstrapping mechanism was used) in case of a key compromise may be a
better alternative to the parent supporting key rollover for child
SIG(0) keys. The decision of whether to allow SIG(0) key rollover via
DNS UPDATE is left as parent-side policy.

## Re-bootstrapping In Case of Errors

Sometimes things get out of sync in spite of all efforts. In this case
it could be an operator error in the parent end losing the child
public key or it could be an error in the child end losing the private
key. Another possibility could be a key rollover that for some reason
didn't sync correctly. 

If {{?I-D.berra-dnsop-keystate}} is supported then the child can
inquire with the UPDATE Receiver about the KeyState of the SIG(0) key
it would use in an UPDATE request. 

In all such cases, as soon as the child becomes aware of the problem
it should simply re-bootstrap by the same mechanism as used
initially. The self-signed DNS UPDATE that starts the bootstrapping
process contains a 

    DEL child.parent. {ttl} ANY KEY

and that is an instruction to the parent UPDATE Receiver to delete any
SIG(0) public keys for this child (and after that start the process to
validate the new key).

Note that when receiving such a self-signed DNS UPDATE the parent MUST
NOT delete any old keys until the new key has been validated and
therefore promoted to "trusted". This is needed and important to avoid
the potential attack vector of an adversary causing a parent to
invalidate the child key by just sending a self-signed bootstrap
UPDATE (which will not validate, but if the old key is deleted then
the harm would already be done).

# Communication Between Child and Parent UPDATE Receiver 

There are two cases where communication between child and parent
UPDATE Receiver would benefit greatly from some additional information.

## Communication in Case of Errors

An error response from the parent UPDATE Receiver would be improved by
more detail provided via a set of new Extended DNS Error Codes {{!RFC8914}}. In
particular, it would be useful to be able to express the following
"states":

* "SIG(0) key is known, but not yet trusted": indicating that bootstrap of 
  the key is not yet complete. Waiting may resolve the issue.

* "SIG(0) key is known, but validation failed": indicating that
  bootstrap has failed and waiting will not resolve the issue.

* "Automatic bootstrap of SIG(0) keys not supported; manual bootstrap
  required": indicating that while the parent does support delegation
  sychronization via DNS UPDATE, it does only support manual
  bootstrap.
  
## Communication To Inquire State

Extended DNS Errors {{!RFC8914}} provides an excellent mechanism
for adding more detail to error responses. However it is,
intentionally, limited to:

* returning extra information from the receiver to the original
  sender. It is not possible to "send" information, only "return"
  information.

* no information except the actual error code is meant for automatic
  processing. it is therefore not possible to communicate things like,
  eg. a KeyId via EDE.
  
The communication between child and parent would gain from the
addition of the ability to also send inquiries to the parent:

* For the child to be able to inquire about the state of the
  parent. I.e. "I operate under the assumption that the key {key} is
  known and trusted by you (the parent). Is this correct?"
  
* For the child to inquire about things: "Do you (parent) support
  automatic bootstrapping or not?"
  
{{?I-D.berra-dnsop-keystate}} is being proposed as a mechanism to
improve the communication between child and parent, both in the error
case and in the inquiry case. If that draft is supported then all of
the above examples would travel as a new "KeyState codes" in a
KeyState OPT as specified in {{?I-D.berra-dnsop-keystate}}.

# Scalability Considerations

The primary existing mechanism for automatic synchronization of DNS
delegation information is based on parent-side "scanning" of the child
zones for CDS and/or CSYNC RRsets, performing DNSSEC validation on the
result and then, in the CSYNC case, based on the result, issue and
validate a potentially large number of additional DNS queries, all of
which must be DNSSEC validated. This makes a CDS/CSYNC scanner for a
parent with a significant number of delegations a complex and resource
consuming service. Among the issues are rate-limiting by large DNS
operators and inherent difficulties in issuing millions of recursive
DNS queries where all received data must be validated.

By comparision, the DNS UPDATE based mechanism for automatic
synchronization shifts most of the effort to the child side. It is the
child that is responsible for detecting the need to update the
delegation information in the parent zone (which makes sense as it is
the child that has made a change and therefore, in many cases, already
"knows"). It is the child rather than the parent that computes what
records should be added or removed. All of this is good for
scalability.

Furthermore, the information collection and validation effort for the
UPDATE Receiver is restricted to validation of a single DNS message,
using a SIG(0) key that the UPDATE Receiver already has.

Hence, as the data collection and validation is much simplified the
task of the UPDATE Receiver is mostly focused on the policy issues of
whether to approve the UPDATE or not (i.e. the same process that a CDS
and/or CSYNC scanner follows).

# Security Considerations.

Any fully automatic mechanism to update the contents of a DNS zone
opens up a potential vulnerability should the mechanism not be
implemented correctly. 

In this case the definition of "correct" is a question for the
receiver of the DNS UPDATE. The receiver should validate the
authenticity of the DNS UPDATE and then do the same checks and
verifications as a CDS or CSYNC scanner does. The difference from the
scanner is only in the validation: single SIG(0) signature by a key
that the UPDATE Receiver trusts vs DNSSEC signatures that chain back
to a DNSSEC trust anchor that the validator trusts.

Another issue of concern is whether a parent-side service that
provides support for changes to child delegation information via DNS
UPDATE is open for potential denial-of-service attacks. The answer is
likely no, as it is possible to have a very strict rate-limiting
policy based on the observation that no child zone should have a
legitimate need to change its delegation information frequently.

Furthermore, as the location of the UPDATE Receiver can be separated
from any parent name server even in the worst case the only service
that can be subject to an attack is the UPDATE Receiver itself, which
is a service that previously did not exist.

# IANA Considerations.

IANA is requested to assign a new "scheme" value to the registry for
"DSYNC Location of Synchronization Endpoints" as follows:

Reference
: (this document)

| RRtype | Scheme | Purpose               | Reference       |
| ------ | ------ | --------------------- | --------------- |
| ANY    |      2 | Delegation management | (this document) |

-------

# Acknowledgements.

* Peter Thomassen and I together came up with the location mechanism
  for the generalized notifications, which this draft relies upon.

* Mark Andrews provided the initial inspiration for writing some code
  to experiment with the combination of the location mechanism from
  the generalised notifications with DNS UPDATEs across zone cuts.

* Stefan Ubbink helped me realize the need to also cater for the case
  of re-bootstrapping if or when things got out of sync for some reason.
  
--- back

# Change History (to be removed before publication)

* draft-johani-dnsop-delegation-mgmt-via-ddns-04

> Reworked the section on automatic bootstrapping a la RFC8078.

> Added a section on re-bootstrapping the SIG(0) key with the parent
> after problems.

> Added text on the importance of augmenting error responses using EDE
> (RFC8914).

> Added text on the insufficiency of RFC8914 for child-to-parent
> communication and a reference to the (upcoming) draft on a new OPT
> code to alleviate this. 

* draft-johani-dnsop-delegation-mgmt-via-ddns-03

> Update the draft based on the excellent dnsdir review of 
> draft-ietf-dnsop-generalized-notify-01 by Patrick Mevsek.

> Expand the section on alternatives for initial bootstrap
> to suggest a mechanism similar to what is used for automatic
> bootstrap of DNSSEC signed delegations via multiple queries
> for child the CDS RRset.

> Added a section on scalability considerations.

> Expanded the Security Considerations section with a paragraph on
> the potential for DDOS attacks aimed at the UPDATE Receiver.

* draft-johani-dnsop-delegation-mgmt-via-ddns-02

> Update the references to the (updated) I-D for generalized
> notifications.

> Remove duplicated descriptions between the two drafts. It is
> sufficient that the generalized notification draft describes the
> mechanics.

* draft-johani-dnsop-delegation-mgmt-via-ddns-01

> Change the RRtype from the original "NOTIFY" to the proposed "DSYNC"

> Expand the description of how to interpret different RCODE responses
> to the UPDATE. Expand the description of bootstrapping
> alternatives. Change the mnemonic of the RR type used from "NOTIFY"
> to "DSYNC" in the examples.

* draft-johani-dnsop-delegation-mgmt-via-ddns-00

> Initial public draft.
