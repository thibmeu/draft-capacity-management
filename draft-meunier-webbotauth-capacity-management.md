---
title: "PACE: Capability Management for Agents"
abbrev: "PACE"
category: info

docname: draft-meunier-webbotauth-capacity-management-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: false
v: 3
# area: "Web and Internet Transport"
# workgroup: "Web Bot Auth"
keyword:
 - anonymous rate limiting
venue:
#  group: "Web Bot Auth"
#  type: "Working Group"
#  mail: "web-bot-auth@ietf.org"
#  arch: "https://mailarchive.ietf.org/arch/browse/web-bot-auth/"
  github: "thibmeu/draft-capacity-management"
  latest: "https://thibmeu.github.io/draft-capacity-management/draft-meunier-webbotauth-capacity-management.html"

author:
 -
    fullname: Thibault Meunier
    organization: Cloudflare
    email: ot-ietf@thibault.uk
 -
    fullname: Christopher A. Wood
    organization: Cloudflare
    email: caw@heapingbits.net

normative:

informative:
  CIVIL: I-D.draft-meunier-bcp-civil
  AAUTH: I-D.draft-hardt-oauth-aauth-protocol
  CONNECT-UDP: RFC9298
  CONNECT-IP: RFC9484
  OHTTP: RFC9458
  PRIVACY-PARTITIONING: RFC9614
  HTTP: RFC9110
  HTTPSIG: RFC9421
  OAUTH: RFC6749
  BEARER: RFC6750
  DPOP: RFC9449
  MTLS: RFC8705
  DIGEST: RFC9530
  EC-ACT: I-D.draft-schlesinger-mole-ec-act
  PRIVACYPASS: RFC9576
  PRIVATETOKEN: RFC9577
  OIDC:
    title: "OpenID Connect Core 1.0"
    target: "https://openid.net/specs/openid-connect-core-1_0.html"
  INTERNET-FOR-END-USER: RFC8890
  IPv6: RFC8200
  MOLE-ARCH: I-D.draft-jms-mole-architecture
  MOLE-PROTOCOLS: I-D.draft-jms-mole-protocols
  MOLE-HTTP: I-D.draft-jms-mole-http-transport
  OAUTH-XAA: I-D.draft-ietf-oauth-identity-assertion-authz-grant
  WAG: I-D.draft-carleton-workload-authz-grant
  POPPY:
    title: "Personal Agent Protocol, draft 0.3"
    target: "https://github.com/personal-agent-protocol/personal-agent-protocol/blob/1de1990e4978b42179fe0e3e0ae554271808c308/content/versions/0.3/spec.mdx"
  AAP:
    title: "Agent Attestation Protocol"
    target: "https://github.com/agentic-protocols/agentic-attestation-protocol/blob/1328a6bbfda24ea18bcd8996a5417a6702c8f274/draft-agent-attestation-protocol.md"
  USER-AGENT-HISTORY:
    title: "WebAIM: History of the browser user-agent string"
    target: "https://webaim.org/blog/user-agent-string-history/"
  WBA-ABA: I-D.draft-rescorla-anonymous-webbotauth
  WBA-HTTPSIG: I-D.draft-ietf-webbotauth-httpsig-protocol
  SIGNED-AGENT-METADATA: I-D.draft-vilgalys-webbotauth-signed-agent-metadata
  WIMSE-AIMS: I-D.draft-ietf-wimse-aims
  WIMSE-ARCH: I-D.draft-ietf-wimse-arch

...

--- abstract

This document addresses the noisy neighbour problem without requiring a
stable client identifier at the origin. Separately allocated budgets let
origins limit consumption beneath a shared platform identity. It describes
PACE: a framework for managing budget continuity across requests using
a composition of the Web Bot Auth with the Moderation of unLinkable Endorsements
(MOLE) architecture {{MOLE-ARCH}} and protocols {{MOLE-PROTOCOLS}}. This composition allows for
per-client capacity management in a way that allows origins to serve clients
above an otherwise constrained public baseline.

--- middle

# Introduction {#intro}

Agents are increasingly used in business and user workflows, including AI
assistants, search indexing, content aggregation, and automated testing. Their
relationship with origins is symbiotic. They need information from origins, and
origins need signals to apply rate-limit policies. These can be binary, like
allow or block when an origin cannot serve traffic, or resource-based to
allocate capacity among clients.

Historically, most of these signals have been enshrined into deployments of
internet protocols. An agent needs an IP address to receive a response from an
origin. Its TLS stack and browser behavior provide further signals. It provides
a familiar `User-Agent` because middleboxes might otherwise filter its traffic
(see {{USER-AGENT-HISTORY}}).

Recent developments break these patterns. Agents use intermediaries to drive
traffic, including running workloads in cloud platforms or by using connection
proxies ({{CONNECT-UDP}}, {{CONNECT-IP}}, {{OHTTP}}, or plain {{HTTP}}).
As workloads shift to platforms making requests on behalf of users, traditional
signals used for moderating traffic are no longer viable.

An address, including an {{IPv6}} address, is not necessarily a user. Blocking
a shared IP range can affect everyone behind it. A platform signing requests under one
{{WBA-HTTPSIG}} identity creates a similar shared fate: when one tenant's
agent overloads an origin, limiting the platform also limits its other
tenants. {{SIGNED-AGENT-METADATA}} describes this problem and proposes
signed product, project, and session identifiers to let an origin isolate
traffic below the platform level.

More granular identifiers are useful when the origin needs to recognize a
particular product or project, investigate abuse with its platform, or follow
a task within a session. {{SIGNED-AGENT-METADATA}} advocates for such
identifiers. While they are pseudonymous, the receiving origin can link
requests within the scope and lifetime of those identifiers. This has several
privacy costs:

- Discriminatory treatment: Explicit identifiers allow for discriminatory
  treatment of traffic based on those identifiers. An origin can give
  preferential treatment to some products or projects, or block them
  regardless of their requests' behavior.
- Linkability: Explicit identifiers allow for linking traffic to that identifier.
  The scope and lifetime of the identifier, together with other request data,
  determine how much activity the origin can associate with it.
- Relationship disclosure: Explicit identifiers reveal information about the
  relationship between an intermediary and its clients, such as which requests
  belong to the same project. Clients may have no clear view of how that
  information will be processed, stored, or shared.

This presents a natural tension between the needs of an origin and the needs
of a user. Identifiers give an origin a way to apply capacity and abuse
controls at the granularity provided. This depends on the platform assigning
them reliably. Agents using those controls bear the cost of attribution. An origin
has to be able to apply policies to traffic whether it finds it abusive or
acceptable. Bounding consumption does not always require more fine-grained
identifiers.

This document describes PACE, a framework for managing capacity for agents with
a combination of Web Bot Auth and MOLE, an architecture for private moderation {{MOLE-ARCH}},
in a way that does not reduce to the shared fate, noisy neighbor problem.
In PACE, budgets are allocated to clients through MOLE, where a client can can
be a tenant on a platform, a tenant or agent workload, or a user behind any of
these, and origins moderate capacity based on those budgets. Traffic is similarly
attributed to platforms through Web Bot Auth signatures. Composing these two
technologies balances the needs of all parties: origins can establish policies on
a per-platform and then per-client basis within that platform, choosing the
granularity of client using their anchor set.

Avoiding unnecessary disclosure supports the end-user interests discussed in
{{INTERNET-FOR-END-USER}}. For these capacity decisions, the origin need not
receive product, project, or session identifiers. When used, a platform
signature remains visible, along with public spending terms and other request
data. This approach does not prevent discrimination based on that information.

Such a mechanism is useful in a number of cases. Some examples follow:

1. An AI agent service that hosts remote agents and concentrates their traffic
   on a small set of IP addresses. One user's activity can affect others when
   an origin can only limit the shared address or the platform as a whole.
2. Network operators carry traffic from device-resident agents behind shared
   IP ranges. Policies applied to an address range or an entire ASN give these
   users a shared fate.
3. VPN providers and enterprise zero trust gateways have a noisy neighbour
   problem. They may know their users, but should not have to expose them to
   each origin. An origin able to tell employees apart could single out an
   executive's activity for instance.

## Related work

{{SIGNED-AGENT-METADATA}} addresses shared fate under a platform's Web Bot
Auth signature with signed, pseudonymous sub-identifiers. It lets an origin
apply policy to a product, project, or session while the platform can map
those identifiers back to its customers. Origin-scoped session values
reduce cross-origin correlation ({{Section 7.2 of SIGNED-AGENT-METADATA}}). However,
each identifier still links requests within its own scope and lifetime.
As described in {{scenarios}}, this document does not specify tenant-specific
abuse reporting. An Anchor can constrain project allocations, but the origin
cannot recognize
and selectively block all of a project's independent Credentials from their
presentations alone.

{{WBA-ABA}} also uses MOLE for anonymous bot access and rate limiting.
Its deployment relies on Anchor vetting and origin acceptance of issuance
policies ({{Sections 3 and 4.1 of WBA-ABA}}). It also discusses convergence on
accepted policies ({{Section 4.3 of WBA-ABA}}) and the risk of cheap registration
yielding multiple credentials ({{Section 8.2 of WBA-ABA}}). These are governance
dependencies between Anchors and sites. This approach shares those dependencies
and focuses on allocations beneath a platform's identity, where the platform
can use its existing tenant relationships to constrain new budgets.

Poppy {{POPPY}} uses User IDs and sessions, including for anonymous browsing.
{{AAP}} uses anonymous admission tokens and recommends a session afterwards,
though it also allows a fresh token per request. Here, the aim is to maintain a
scarce, independently allocated budget without linking its presentations.

{{WIMSE-AIMS}} profiles WIMSE and OAuth for agents within a defined trust domain.
It treats an agent as a workload with its own identifier and credential. It
carries the user, when there is one, only as the subject of an OAuth access token, so that the two
identities are separate and compose on one request. {{WAG}} lets a
platform-hosted workload obtain such a token for itself without per-workload
registration. This document places both among the options an origin can ask
for, next to attribution and anonymous capacity.

# Conventions and Definitions {#terminology}

This document uses the HTTP terminology of {{HTTP}} and the MOLE terminology
of {{MOLE-ARCH}}. The roles can be operated by separate parties or combined.

## Parties and hosting relationships {#parties}

**Platform:**
: A service that hosts workloads for tenants.

**Tenant:**
: A customer organization, developer account, or billing workspace that runs
  workloads on a platform, often organized as a project.

**Workload:**
: A deployed application or service, as described in {{WIMSE-ARCH}}.
  A software product or service can comprise several workloads.

**Agent:**
: Software that performs tasks, potentially on behalf of a user.
  A session is a bounded task execution or workflow run, not the agent itself.

**User:**
: A person on whose behalf an agent may act.

A typical hosted deployment has this hierarchy:

~~~
Platform
  Tenant
    Workload
      Agent
~~~

A tenant runs workloads that execute agents. Users can use several agents,
and agents can serve several users or run unattended. Device-resident agents
need no hosting tenant.

## MOLE roles and budget scopes

**Client:**
: The MOLE participant that obtains an Endorsement, redeems it for a
  Credential, and presents that Credential when requesting a resource.
  An agent or a shared workload component can implement this protocol role.

**Anchor:**
: The party that grants Endorsements under a policy the Moderator accepts.
  Its relationship with a Client constrains access to new budgets.

**Moderator:**
: The party that issues Credentials and verifies their presentations. It
  sets the budget policy and may be operated by the origin or on its behalf.

**Origin:**
: The server that controls access to the requested resource.

**Intermediary:**
: A proxy, gateway, or tunnel on the request path. An intermediary that
  terminates HTTP can sign forwarded requests under its own identity.

A Client may hold several budgets, and an account or workload may receive
budgets for several Clients. Those allocations depend on Anchor and Moderator
policy. A Credential does not establish a unique account, workload, device,
or other principal.

MOLE(X) denotes a presentation for budget scope X. MOLE(C) is a Client's
budget, while MOLE(T), MOLE(W), and MOLE(U) denote tenant, workload, and
account scopes. These labels do not appear as identifiers in presentations.
WBA(Y) denotes a Web Bot Auth signature from intermediary Y.

## Endorsement examples {#endorsement-examples}

An accepted Anchor could allocate Endorsements at each scope:

* Platform: A vetting service grants a hosting provider a finite pool.
* Tenant: The platform caps a retailer's workspace allocation under its
  subscription, as in {{scenarios}}.
* Workload: A deployment service authenticates the shopping-assistant workload
  and caps its allocation across replicas and restarts.
* Agent: The workload operator allocates a shopping agent's task budget
  against the workload's allowance. New tasks do not reset that allowance.
* User: An account provider authenticates a customer and caps allocations
  across that account's agents. This limits accounts, not unique people.

Clients redeem these Endorsements for Credentials. The Anchor accounts for the
allocation scope, which presentations do not reveal. Capacity evidence does
not grant permission. Combining budget proofs remains undefined ({{composition}}).

# Scope and principles {#principles}

The goal of PACE is to bound consumption on a per-platform and per-client basis
through independently allocated and managed budgets. Exhausting one budget
should not exhaust another allocation, provided Anchor issuance and Moderator
spending policies keep them separate.

The purpose is to allow origins to offer better public baseline services for its
users. Without per-client enforcement, origins are left with overly restrictive
baselines (e.g., block everything) because honest clients cannot prove their capacity.
In other words, the deployment goal is a baseline level of access without a credential,
signature, or some other mechanism that demonstrates capacity. Absence of an active
signal does not itself establish abuse {{CIVIL}}.

Permissioned (non-public) resources may require evidence of a right before
serving any content. A capability can establish that right without identifying
its holder. Where a resource's policy depends on a particular principal and
its permissions, authenticating the principal with a stable identifier is
appropriate. Capacity evidence alone establishes neither identity nor a
right to perform a permissioned operation.

Clients choose which evidence to offer based on their needs and the needs of
the origin. Origins should ask only for evidence needed by the requested operation.
A verified identity does not establish that a request is harmless, and a valid
budget does not grant every permission. Moreover, client evidence should reveal
only what the origin needs to apply its policy. Network metadata and other
request fields can still identify or correlate clients.

# Attribution, permission, and capacity {#system}

PACE is centered around moderation through attribution and authorization.
Specifically, an origin decides whether to serve an operation and how much
capacity to allocate to it using evidence presented from the client. This evidence
can establish attribution (who is responsible for this traffic?), permission
(is the party responsible for this traffic allowed to access the resource?),
or capacity (does the party responsible for this traffic have budget for
accessing the resource?). A request may need more than one form of evidence.

## Attribution

Attribution addresses the question of who is responsible for traffic,
and there are many layers at which this attribution can occur (see {{parties}}).
For example, a verified WBA(P) signature attributes its covered components to
a key published by P {{WBA-HTTPSIG}}. The origin can apply an aggregate limit to P
without receiving an identifier for each Client behind it. The signature does
not establish permission unless origin policy grants it to that signer.
A workload can authenticate using a variety of schemes, e.g., with a WIMSE
Proof Tokens, HTTP message signatures, or even transport-layer authentication
(see {{WIMSE-AIMS}}). Users can authentiate using {{OIDC}} or related techniques.

## Permission

Permission addresses the question of whether the attributed party can access
the resource in question. As such, permission depends on the resource's policy.
For example, an OAuth access token can convey an authorization grant {{OAUTH}}.
The origin can check the token as its authorization server requires, including
its audience, validity, permissions, and any proof of possession. Identity
obtained through {{OIDC}} or workload authentication can also be an input
to access policy.

A workload can obtain a token using {{WAG}}, and an existing federation can
support delegated access through {{OAUTH-XAA}}. These grants are exchanged at
the authorization server, which decides what access token to issue. They do
not define a common token claim layout. {{AAUTH}} defines a separate signed
authorization protocol with its own challenges and credentials.

Notably, authentication alone does not determine the
permitted operations. In fact, an origin can accept a capability without
identifying its holder, including a Privacy Pass token when the issuer's
attestation satisfies that policy {{PRIVACYPASS}}.

## Capacity

Capacity addresses the question of whether the attributed party has available
budget for accessing a resource. Budget can be allocated and managed using
granular identities such as a session ID, user ID, or platform ID. Budget
can also be allocated and managed using MOLE, which permits clients to hold
a hidden balance `c` and then update it through interaction with the origin
and its moderator. (This is referred to as presenting a MOLE credential.)
The origin learns the public amounts and that the spend
was possible, without learning the balance or a stable client identifier
{{MOLE-PROTOCOLS}}.

For example, a policy can issue `B` credits, charge one per accepted
presentation, and return zero with no top-ups. Each issuance then allows at most
`B` fresh accepted operation attempts. Further issuances increase that
allocation. Bounding consumption over time also requires policy for replenishment
and expiry. Refunding every spend in full leaves successful consumption
unbounded.

## Composition {#composition}

Attribution, authorization, and capacity evidence can compose in requests.
For instance, an origin might require that requests carry a platform signature
(WBA(P)) and a MOLE presentation corresponding to the user (MOLE(U)) on each
request. An origin might also require both a workload authorization token granting
permission to use a resource and a MOLE presentation for the user behind that workload.
Origins decide what composition is essential based on their needs.

Carriage of a MOLE presentation alongside another authorization scheme or
additional budget presentations, and signaling cumulative requirements, remain
undefined here (see [issue #9](https://github.com/thibmeu/draft-capacity-management/issues/9)).

# Moderating consumption {#processing-steps}

Given mechanisms for establishing attribution, permission, and capacity,
origins can moderate composition accordingly.
The origin's moderator configures its accepted Anchor Set, credential contexts,
allocation and spending policy, and any permission requirements. A client may
send an initial request without evidence. The origin then verifies what is present
and serves the level of access its policy permits. For example, a public response
can offer more capacity through a MOLE challenge. A permissioned resource can
refuse the operation and challenge for the missing authorization.

A MOLE redemption challenge lets a client obtain a credential. A presentation
challenge specifies its spending terms. Clients use the exchanges and
verification rules of {{MOLE-PROTOCOLS}} and {{MOLE-HTTP}}, including replay
checks and request binding. A failed proof supplies no evidence of budget. A
refund replaces the spent Credential with an updated one. Returning zero credits
leaves `c - s` credits. Recording `NoUpdate` ends the chain, as illustrated in
{{scenarios}}. The Moderator records this decision before attempting the
operation, using the evidence available at that point.

After a lost response, the Client can recover the recorded credential result
within the retention period. Recovery does not repeat the protected operation
or establish whether it completed. Application recovery remains separate.
One credential permits one presentation in flight at a time.

## Public resource {#flow1}

The origin offers a budget above its public baseline. Client C already holds
an Endorsement from an accepted Anchor. Intermediary P adds its own signature.
The Moderator is shown at the origin.

~~~aasvg
+-------------+  +-------------+   +---------------+  +---------------+
|   Anchor    |  |  Client C   |   | Intermediary  |  |    Origin     |
|             |  |             |   |       P       |  |   Moderator   |
+------+------+  +------+------+   +-------+-------+  +-------+-------+
       |  Endorsement   |                  |                  |
       +--------------->+                  |                  |
       |                |                  |                  |
       |                |     request      |                  |
       |                +----------------->+                  |
       |                |                  |                  |
       |                |                  |     + WBA(P)     |
       |                |                  +----------------->+
       |                |                  |                  |
       |                |      baseline + redemption and      |
       |                |       presentation challenges       |
       |                +<-----------------+------------------+
       |                |                  |                  |
       |                |           Redeem & Issue            |
       |                +------------------+----------------->+
       |                |                  |                  |
       |                |      Credential with balance c      |
       |                +<-----------------+------------------+
       |                |                  |                  |
       |                |request + MOLE(C) |                  |
       |                +----------------->+                  |
       |                |                  |                  |
       |                |                  |     + WBA(P)     |
       |                |                  +----------------->+
       |                |                  |                  |
       |                |   record spend, attempt operation   |
       |                |        response with Update         |
       |                +<-----------------+------------------+
       |                |                  |                  |

                   Client balance: c - s + t
~~~
{: #sequence title="Public access with carrier attribution and an anonymous budget"}

1. The Client sends a request. The intermediary adds WBA(P), which the
   origin verifies before applying its aggregate limit.
2. The origin serves the public baseline and offers redemption and presentation
   challenges on the 200 response. A Client with a suitable Credential skips
   redemption. Otherwise, it fetches the Moderator configuration and redeems
   its Endorsement at the Redeem & Issue endpoint.
3. The Client presents proof that its balance covers `s` and requests a top-up
   of at most `a`. The Moderator verifies and records the spend result atomically
   before attempting the operation.
4. The response returns an Update with `0 <= t <= s + a`, continuing the chain
   with balance `c - s + t`. Each subsequent spend answers an applicable
   challenge and uses a fresh nullifier, without a credential link to the
   previous presentation.

A direct Client uses the same exchange without WBA(P). A Client unable or
unwilling to present continues at the public baseline, subject to origin policy.
A failed presentation does not authorize the budgeted level of access.

# Budget limits and abuse {#scenarios}

Consider the following scenario. A platform hosts agents for two tenants. Tenant A sends excessive requests
while tenant B runs a shopping agent within its allocation. Both tenants'
requests carry the same platform signature. Blocking that signature would
interrupt both tenants. The origin needs to bound A's consumption and stop
abusive requests without requiring either tenant to disclose a stable
identifier.

Suppose the platform also acts as an Anchor accepted by the origin. It grants
at most two Endorsements per tenant per epoch. Each Endorsement can be redeemed
once at this Moderator for an EC-ACT Credential with ten credits. The Moderator
charges one credit per accepted operation attempt and returns an updated
Credential with the remaining balance. It returns no credits and offers no
top-ups. Credentials expire at the epoch boundary.

In this scenario, credits represent capacity granted by the Moderator under its
own access policy. Twenty credits correspond to the capacity allocated to a
tenant per epoch. A system based on fingerprints might instead apply a rate limit
per IP address, `User-Agent`, or TLS fingerprint.

Each tenant's allocation therefore permits at most twenty budgeted operation
attempts in the epoch. Starting another task does not reset the allowance.
This bound depends on the Anchor enforcing the allocation and on other
accepted Anchors not supplying additional budgets. The origin verifies each
spend without learning which tenant's allocation it consumes.

If the origin detects abuse on A's presented request, its Moderator can record
`NoUpdate` and end that credential chain. The chain is the sequence of
Credentials obtained by updating one initial Credential. Its remaining credits
are forfeited. B's independently allocated Credentials remain usable.
{{fig-budget-ban}} shows this decision. The tenant labels explain the example and
are not identifiers carried in presentations.

~~~aasvg
+-----------------------------------------------------------+
|                 Platform acting as Anchor                 |
|           At most two Endorsements per tenant             |
+-------------+-------------------------------+-------------+
              |                               |
              v                               v
+---------------------------+   +---------------------------+
|         Tenant A          |   |         Tenant B          |
|    Two 10-credit budgets  |   |    Two 10-credit budgets  |
+-------------+-------------+   +-------------+-------------+
              |                               |
       Abusive request                 Allowed request
       + presentation                  + presentation
              |                               |
              v                               v
+-----------------------------------------------------------+
|                   Origin and Moderator                    |
|          Both requests carry the platform signature       |
|          Each presentation spends a separate budget       |
+-------------+-------------------------------+-------------+
              |                               |
          NoUpdate                      Update (t=0)
              |                               |
              v                               v
+---------------------------+   +---------------------------+
|   Presented chain ends    |   |      Chain continues      |
|   Remaining credits lost  |   |      One credit spent     |
+---------------------------+   +---------------------------+
~~~
{: #fig-budget-ban title="Ending an abusive chain while an independent budget remains usable"}

This ban applies to the presented chain. It does not revoke A's other
Credential or prevent A from obtaining a later allocation. A replacement must
count against the tenant's allowance. Exhaustion also prevents further spending
from a budget, but an update returning zero credits does not end a chain that
still has a sufficient balance. Neither a task boundary nor a tenant boundary
is established by the presentation itself.

The platform's aggregate limit and the origin's available capacity still apply
to both tenants. Public-baseline requests are outside the illustrated budget.
Missing budget evidence does not itself establish abuse.

For the origin to ask the platform to investigate a tenant or change its future
allocations, they need a feedback arrangement. One possible extension would
require a fresh platform Endorsement before accepting an origin-issued
Privacy Pass token, using a reverse-flow exchange. Feedback could also occur out of band. These
are possible deployment arrangements, not exchanges specified here. Their
binding to the affected allocation, enforcement, and disclosure to each party
remain to be defined. They need not require a persistent tenant identifier at
the origin.

# Deploying PACE {#examples}

A deployment of PACE admits two kinds of capacity control: (1) through
an intermediary which identify itself for an aggregate, platform-specific limit,
and (2) through smaller, anonymous budgets that are allocated to indivdual
clients (tenants, workloads, agents, or users) based on the chosen anchor
set. {{fig-intermediary}} shows one example where an Anchor endorses Client C
while intermediary platform P carries its requests. The same provider can operate
both the role of anchor and intermediary, but need not.

~~~aasvg
+----------+
|  Anchor  |
+----+-----+
     | Endorsement
     v
+----------+           +--------------+           +-----------+
| Client C |  MOLE(C)  | Intermediary |  MOLE(C)  |   Origin  |
|          +---------->|      P       +---------->| Moderator |
|          |           |              |  +WBA(P)  |           |
+----------+           +--------------+           +-----------+
~~~
{: #fig-intermediary title="Separate endorsement and forwarding relationships"}

The origin can attribute the forwarded request to P through WBA(P) and
charge C's separate budget through MOLE(C), without learning C's identity
from that presentation. The allocation relationship, rather than P's
signature, establishes which smaller unit bears the spend. The following
subsection applies this separation to product, project, and session
capacity limits.

## Capacity scopes

Using the terms in {{parties}}, a platform P hosts tenants T with workloads W.
An account U may authorize several agents. {{endorsement-examples}} describes
allocations at these scopes. The noisy-neighbour
boundary depends on which of these shares a budget. A tenant budget separates
tenants but leaves workloads within a tenant sharing its allocation. A
workload budget separates workloads but still groups their Clients. Account
budgets can isolate those allocations further, while several Clients sharing
one account budget remain subject to the same limit.

In {{fig-stacked}}, Anchor W endorses workload-scoped allocations and Anchor U
endorses account-scoped allocations. Client components obtain and spend the
corresponding Credentials. The request carries both budget proofs alongside
the platform's signature.
The Anchors need not operate the forwarding intermediary. These are intended
scopes, pending the multiple-presentation carriage in {{composition}}.

~~~aasvg
+---------------------+       +---------------------+
|      Anchor W       |       |      Anchor U       |
+----------+----------+       +----------+----------+
           | Endorsement                 | Endorsement
           v                             v
+--------------------------------------------------+
| Platform P                                       |
|  +--------------------------------------------+  |
|  | Tenant workload with Clients               |  |
|  +----------------------+---------------------+  |
|                         |                        |
|                         v                        |
|                 +--------------+                 |
|                 | Intermediary +-----------------+---+
|                 +--------------+                 |   |
+--------------------------------------------------+   |
                                                       | MOLE(W)
                                                       | MOLE(U)
                                                       | +WBA(P)
                                                       v
                                                 +-----------+
                                                 |   Origin  |
                                                 | Moderator |
                                                 +-----------+
~~~
{: #fig-stacked title="Workload and account budget scopes in a proposed stacked deployment"}

## Composing scopes

A client can receive several independent budgets, for example one per
tenant, workload, agent, or user. Authorization determines what each client
may do, while issuance policy constrains their combined allocation. Adding
Clients must not yield unbounded replacement budgets. Aggregate platform
limits should still apply when individual budgets remain available.

With MOLE, concurrent requests need separate credentials, since each request
effectively "locks" a credential while in use. Several Credentials provide
separate balances, so assigning more Credentials can increase both
concurrency and total consumption. Shared-budget splitting and simultaneous
checks of aggregate and individual anonymous budgets remain open.

A CDN can operate a Moderator for several origins. All servers accepting the
same credential key and context must share atomic replay state. Independent
stores need disjoint keys or contexts. Sharing a credential context also
shares its budget scope and can enlarge the anonymity set, but activity at
one origin can then reduce the balance available at another.

Anchors constrain new allocations, but the Moderator cannot identify the
issuing Anchor from a redemption. {{MOLE-ARCH}} leaves privacy-preserving
Anchor feedback unresolved. Removing an Anchor requires evidence from outside
that anonymous redemption, and changing sets to probe Clients can reduce
privacy. Endorsements from several accepted Anchors can yield several budgets.

Deployment also needs bounded verification work and replay storage. Invalid
proofs can consume verification capacity, and replay markers must remain while
their Credentials are accepted. Aggregate controls remain necessary for
availability, even though they can reintroduce shared fate.

# Security considerations {#security}

The security goals and limitations of MOLE are described in {{MOLE-ARCH}},
{{MOLE-PROTOCOLS}}, and {{EC-ACT}}.

## Request integrity

An intermediary that sees a presentation can race the Client to spend it.
The request-bound profile of {{MOLE-PROTOCOLS}} must cover every component
that affects authorization, including the body where relevant. Single-use
verification alone determines only which spend wins. Binding does not prevent
an intermediary from withholding or racing an identical request.

A carrier signature should cover the MOLE presentation and the relevant
request components. It attributes that pairing to the carrier. Protection
against the carrier itself requires binding the Client's evidence to the
operation. This document does not define the intermediary signing profile.

Sender-constrained access tokens limit reuse of stolen tokens, but DPoP does
not protect the request body {{DPOP}}. Mutual TLS authenticates the connection
at its TLS endpoint {{MTLS}}, which may be an intermediary. Application-layer
signatures can preserve integrity across TLS termination {{WIMSE-AIMS}}.
For signed HTTP content, the verifier checks `Content-Digest` against the
received content and verifies a signature covering it, together with the
other components needed to bind the operation {{HTTPSIG}} {{DIGEST}}.

## Correlation and policy

Unlinkable Credentials do not hide addresses, cookies, content, or timing.
A forwarding provider that knows the Client can disclose its requests to the
origin. This deployment does not prevent that collusion.

Credential keys, contexts, and public amounts partition the anonymity set.
They should follow coarse policies shared by many Clients. Per-Client values
or repeated balance probes can make presentations distinguishable. The
privacy requirements of {{MOLE-PROTOCOLS}} apply to allocation, spending,
and refund policy. A platform signature further narrows the observable set.

A Moderator can end a credential chain without identifying its holder. Clients
therefore rely on it to apply the advertised policy. Withholding refunds can
consume scarce Endorsements, while recovery distinguishes a lost response from
a recorded termination only while the result remains available.

# IANA considerations {#iana}

This document has no IANA actions.

--- back

# Acknowledgments

The authors would like to thank Bryan Becker, Marwan Fayed, and Mari Galicer for
discussions that helped shape this document.

# Capacity without identifiers

Signed product, project, and session IDs give an origin visible keys with which
to keep separate counters {{SIGNED-AGENT-METADATA}}. For capacity alone, the
Anchor can constrain Endorsements per product, project, or task allocation
only if it knows and enforces those boundaries. The platform can play this
role. An independent Anchor needs a way to account for the same allocations,
or its Endorsements cannot enforce the platform's project and product totals.

Clients redeem those endorsements for independent MOLE credentials and
spend the credentials on requests. The Moderator checks
each spend and carries the remaining balance into an unlinkable
replacement Credential. The presentation itself does not reveal which
product, project, task, or Client holds a particular budget.

The platform can still sign WBA(P), so the origin can also limit its traffic in aggregate.
Credential contexts and spending terms should be shared across many
endorsements. Unique contexts or amounts would distinguish those units and
shrink their anonymity sets.

The allocation boundary can be chosen without adding a corresponding ID to
requests:

* Product: A platform or other suitable Anchor can reserve a finite pool of
  Endorsements for a product and distribute them among its projects or
  Clients. Spending is limited by the budgets derived from that pool. The
  origin does not learn which product a presentation came from, nor can it
  impose a different per-product policy from presentations alone.
* Project: The platform can cap Endorsements for a tenant workspace and
  allocate them among its Clients. Exhausting one Client's independent
  Credential does not exhaust another's. The platform, not the origin,
  accounts for how many endorsements belong to that project. A single
  project-wide balance shared by many concurrent Clients is a different
  requirement, discussed below.
* Session: An Anchor can allocate Endorsements to a task (like an ephemeral
  workload). The Anchor must count task allocations against a scarce parent
  allowance so starting new tasks does not reset the limit. The allocating
  party and Client must keep the resulting Credentials assigned to those
  tasks: a MOLE presentation does not prove which task is using a Credential.
  Alternatively, a Client can spend the same Credential across tasks when
  task-specific budgets are unnecessary. Neither choice lets the origin link
  requests as one task. Such correlation requires an additional mechanism.

The example in {{scenarios}} bounds a tenant's total allocation while the
origin checks each Credential separately. Applying that policy to a project
requires the Anchor to account for all allocations to that project.

The Anchor can also cap the sum of Endorsements across projects belonging
to one product. The origin enforces each Credential's ten-credit limit
without seeing the project or product. The project and product totals depend
on the Anchor's issuance policy, not on an origin-side counter keyed by an ID.
Additional accepted Anchors or Endorsements increase the available budget
unless their policies constrain the combined allocation. Public-baseline
requests and the platform's aggregate limit remain separate from this bound.

These budgets replace identifiers only for the capacity decisions their
issuance and spending policies implement. An origin cannot use an anonymous
presentation to identify and block a particular project, audit its activity,
or correlate all requests in a session. If it needs those functions, signed
metadata can accompany MOLE, making requests linkable at the metadata's scope.
