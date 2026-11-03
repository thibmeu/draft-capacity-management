---
title: "Attributable Platforms and Anonymous Agents"
abbrev: "Brian"
category: info

docname: draft-meunier-webbotauth-brian-latest
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
  github: "thibmeu/draft-brian"
  latest: "https://thibmeu.github.io/draft-brian/draft-meunier-webbotauth-brian.html"

author:
 -
    fullname: Thibault Meunier
    organization: Cloudflare
    email: ot-ietf@thibault.uk

normative:

informative:
  AAUTH: I-D.draft-hardt-oauth-aauth-protocol
  CONNECT-UDP: RFC9298
  CONNECT-IP: RFC9484
  OHTTP: RFC9458
  PRIVACY-PARTITIONING: RFC9614
  HTTP: RFC9110
  INTERNET-FOR-END-USER: RFC8890
  IPv6: RFC8200
  MOLE-ARCH: I-D.draft-jms-mole-architecture
  MOLE-PROTOCOLS: I-D.draft-jms-mole-protocols
  MOLE-HTTP: I-D.draft-jms-mole-http-transport
  OAUTH-XAA: I-D.draft-ietf-oauth-identity-assertion-authz-grant
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
  WIMSE-AIMS: I-D.draft-ietf-wimse-aims
  WIMSE-ARCH: I-D.draft-ietf-wimse-arch

...

--- abstract

Origins rate limit requests by address or by the platform they come through.
Behind a shared platform, such as an AI agent service, a VPN, or a network
operator, every user then shares the same fate. This document describes the
problem of limiting consumption without requiring users or their agents to
disclose a stable identifier. It sketches a direction in which providers
endorse their clients, origins apply anonymous budgets, and platforms can
identify themselves for separate aggregate limits.

--- middle

# Introduction {#intro}

Agents are increasingly used in business and user workflows, including AI
assistants, search indexing, content aggregation, and automated testing. Their
relationship with origins is symbiotic. They need information from origins, and
origins need signals to apply rate-limit policies. These can be binary, like
allow or block when an origin cannot serve traffic, or resource-based to ensure
fair access to everyone.

Historically, most of these signals have been enshrined into deployments of
internet protocols. An agent needs an IP address to receive a response from an
origin. Its TLS stack and browser behavior provide further signals. It provides
a familiar User-Agent because middleboxes might otherwise filter its traffic
(see {{USER-AGENT-HISTORY}}).

Recent developments break these patterns. Agents use proxies
({{CONNECT-UDP}}, {{CONNECT-IP}}, {{OHTTP}}, or plain {{HTTP}}).
As platforms make requests on behalf of users, the user's TLS stack may no
longer be the one visible to the origin. AI agents, on the user's device or
hosted remotely, fetch and render information in place of browsers. User-Agent
strings are still spoofed.

Origins need to apply policy despite these changes. Many approaches rely on
obtaining a stable identifier. An address, including an {{IPv6}} address, is not
necessarily a user. Blocking a shared IP range can affect everyone behind it.
In enterprise settings, an agent can identify as a workload {{WIMSE-AIMS}}.
Platforms that can tolerate attribution can identify using {{WBA-HTTPSIG}}.
Other approaches involve user authentication or authorization, such as {{AAUTH}}
and {{OAUTH-XAA}}.

While identification solves many problems, it should not be necessary just to
limit an agent's consumption. The problem is to give origins that control
without requiring users or their agents to disclose a stable identifier. This
supports the user interests discussed in {{INTERNET-FOR-END-USER}}.

## Motivating use cases

In the cases below, traffic passes through platforms with existing user
relationships. They have means to provide scarce endorsements for their
users, which origins could use to apply rate-limit policies without learning
who the user is.

1. AI agent services host remote agents and concentrate their traffic on a
   small set of IP addresses. One user's activity can affect others when an
   origin can only limit the shared address or the platform as a whole.
2. Network operators carry traffic from device-resident agents behind shared
   IP ranges. Policies applied to an address range or an entire ASN give these
   users a shared fate.
3. VPN providers and enterprise ZTNA gateways have a noisy neighbour problem.
   They may know their users, but should not have to expose them to each
   origin. An origin able to tell employees apart could single out an
   executive's activity for instance.

## Related work

{{WBA-ABA}} shares the anonymous rate-limiting goal and the MOLE construction.
This document is a companion to it. It focuses on platforms that already have
a relationship with their users, and on composing with a platform's
{{WBA-HTTPSIG}} identity.

Poppy {{POPPY}} uses User IDs and sessions, including for anonymous browsing.
{{AAP}} uses anonymous admission tokens and recommends a session afterwards,
though it also allows a fresh token per request. Here, the aim is to maintain a
per-agent budget without linking its presentations.

# Conventions and Definitions

This document uses the HTTP terminology of {{HTTP}}, the MOLE terminology of
{{MOLE-ARCH}}, and the workload terminology of {{Section 2 of WIMSE-ARCH}}.
For instance:

User agent:
: A client program that initiates a request {{Section 3.5 of HTTP}}.

Intermediary:
: A proxy, gateway, or tunnel {{Section 3.7 of HTTP}}.

Origin server:
: A program that can originate authoritative responses for a target resource
  {{Section 3.6 of HTTP}}.

Workload:
: A logical software entity that may run as one or more workload instances
  {{Section 2 of WIMSE-ARCH}}.

Platform:
: A provider that hosts or carries traffic for clients.

Tenant:
: A customer that deploys workloads on a platform.

Anchor:
: The MOLE role that grants Endorsements to Clients {{MOLE-ARCH}}.

Moderator:
: The MOLE role that issues Credentials and evaluates their presentations
  {{MOLE-ARCH}}.

# Principles {#principles}

Finer-grained identity can give origins a finer-grained policy key, but it
also introduces four problems.

Security:
: An agent can spoof an unauthenticated identifier. Authenticating it
  establishes who supplied a request, not whether that request is harmless.

Equitability and scarcity:
: Origins need scarce budgets to keep clients from evading limits by starting
  over. Access to a trusted identity or an Anchor relationship is uneven,
  however. Scarcity based on those relationships cannot offer everyone the
  same opportunity to obtain a budget.

Privacy:
: A stable identifier lets origins correlate activity across requests,
  agents, and sessions, even when all they need is a limit on consumption.

Agency:
: Users should choose whether their agents disclose identity or present an
  anonymous Credential. Requiring disclosure for any access to public content
  makes that choice a condition of participation.

The design goal for public resources is therefore a baseline level of access
without an identity, a platform signature, or a MOLE presentation. This floor
may be heavily rate limited and remains subject to available capacity and
request behavior. Scarce endorsements can support access above that floor.
Here, zero disclosure means no additional authentication signal. Ordinary
HTTP and network metadata remain visible.

Privacy partitioning {{PRIVACY-PARTITIONING}} provides the framing: separate
knowledge of who a user is from knowledge of what they do. An Anchor can use
its relationship with a user to endorse a client, while an origin evaluates
consumption without learning that identity from the presentation. A provider
that also forwards requests may see both. This separation is a privacy goal,
not a guarantee against that provider or its collusion with the origin.

Moderation granularity should follow differences in behavior. Shared fate
within a workload may be acceptable while its agents behave alike. When one
user's behavior causes unrelated users to be throttled, the noisy neighbour
problem remains, even if its scope is smaller than a whole platform. Platforms
and workloads can help origins moderate those smaller units without requiring
stable identifiers for each of them. How to compose these limits remains an
open question.

## Capacity and identity

Capacity control and authorization answer different questions. For a public
resource, knowing an agent's identity does not establish how much it should
consume or replace an anonymous budget. For a permissioned resource, a MOLE
budget does not establish the principal or permissions needed for access.
An origin can require application authentication and authorization for such a
resource and apply capacity limits as well. The public-access floor does not
grant access to permissioned resources.

The two are not interchangeable in either direction. Demanding identity on a
public resource is the coerced disclosure that the agency principle warns
against, and hands the origin a linkable key it did not need. An origin may
need both, for instance a permissioned API that also wants a per-user budget,
and then asks for both. The HTTP transport for MOLE {{MOLE-HTTP}} marks the
difference: a Challenge on a 401 response means a presentation is required,
and a Challenge on a 200 response means it is optional and would, for
instance, raise a limit.

# Attribution and moderation {#system}

A platform can identify itself to origins using {{WBA-HTTPSIG}}, rather than
leaving origins to infer a platform from IP addresses. Whether a party
identifies depends on whether it can tolerate attribution. Concentrated
traffic from one operator, such as a crawler or a hosting platform, usually
can. Traffic distributed across users, from a person or from an agent acting
for one, usually cannot, whatever software sends it. This attribution can
coexist with anonymous moderation of the clients behind it.

An Anchor endorses a client. The client redeems its Endorsement for a Credential
from the Moderator, then presents and updates that Credential as it makes
requests ({{MOLE-PROTOCOLS}}). The origin can apply a rate limit or spendable
budget without linking the Credential's presentations. Where an intermediary
identifies its forwarded traffic with Web Bot Auth, the origin can also limit
that traffic in aggregate.

Each limit constrains a different party. The aggregate limit constrains the
signer. It says nothing about an Anchor that is a different party. A workload
budget confines shared fate to that workload rather than the whole platform.
A user budget gives fairness among an honest Anchor's users. An exhausted
aggregate budget still affects everyone sharing it.

The Anchor limits how many Endorsements it grants, based on a relationship
that is costly to duplicate. The origin chooses which Anchors it accepts.
An origin that accepts an Anchor it cannot evaluate is trusting it with the
scarcity of its own budgets.

## Request flow

A simple exchange has four steps:

1. An Anchor grants the client an Endorsement based on their existing
   relationship.
2. The client redeems that Endorsement at the Moderator for a Credential
   with an initial budget.
3. The client answers a Moderator challenge with a presentation on its
   request. An HTTP intermediary may add its platform signature.
4. The Moderator verifies the presentation and applies its budget policy.
   It returns an Update that the client uses to obtain the Credential for
   its next presentation. The origin can also apply an aggregate limit to
   an identified signer.

The protocol exchanges and HTTP carriage are defined in {{MOLE-PROTOCOLS}}
and {{MOLE-HTTP}}. This example uses one budget; composing workload and user
budgets remains an open point below.

## What the origin sees {#origin-view}

The topologies below differ in who signs, who endorses, how many
intermediaries sit on the path, and whether the agent runs on the user's
device or on a platform. None of that needs to be visible to the origin. It
sees a request that carries zero or more {{WBA-HTTPSIG}} signatures and zero
or more MOLE presentations, which provide the signals for these budgets and
aggregate limits.

The table describes what these cryptographic signals disclose, not everything
an origin can learn from a request. Cookies, application authentication,
source addresses, and other metadata may identify a user or link requests
(see {{security}}).

| The origin sees                         | The origin does not see       |
|-----------------------------------------|-------------------------------|
| Which platform or operator signed       | Where the agent runs          |
| A valid presentation at workload grain  | Which workload or tenant      |
| A valid presentation at user grain      | Which user                    |
| That the Anchor is in the Anchor Set    | Which Anchor in the set       |
| The spend the presentation answers      | The hidden balance            |
| The outcome of its policy for a request | Earlier requests of the agent |
{: #origin-view-table title="What an origin sees"}

An origin does not configure itself per platform or per topology. It
configures a Moderator policy, including an Anchor Set and the amounts it
charges and refunds. Everything else follows from what each request carries.

## Deployment examples

The following topologies separate the traffic path from the endorsement
relationship. An Anchor may be the platform, a tenant, or an independent party
accepted by the origin. The origin's Moderator may be operated by the origin
or delegated to a provider acting on its behalf.

| Topology | Possible signer | Budget scope |
|----------|-----------------|--------------|
| Device agent via a platform | HTTP intermediary | User or agent |
| Device agent connecting directly | None needed | User or agent |
| Consumer AI service | Service operator | User or agent |
| Hosted tenant workloads | Platform | Workload |
| Hosted agents with stacked limits | Platform | Workload and user (open) |
| Enterprise gateway | HTTP gateway | Employee or agent |
| CDN-operated Moderator | As above | As above, across origins |
| Chained intermediaries | Each HTTP-terminating hop | As above |
{: #topologies title="Recurring deployment topologies"}

The budget scopes describe the intended allocation, not identities revealed
to the origin. A platform signature allows a separate aggregate limit. Signing
forwarded requests needs the profile discussed below. A tunnel carrying the
client's TLS session untouched cannot add such a signature.

In {{fig-intermediary}}, a device agent presents through an HTTP intermediary.
The Anchor is drawn separately to show that it need not operate that
intermediary. A consumer AI service can combine these roles and host the agent.

~~~aasvg
+----------+
|  Anchor  |
+----+-----+
     | Endorsement
     v
+----------+           +--------------+          +-----------+
|  Device  +-- MOLE -->| Intermediary +-- MOLE ->|   Origin  |
|  agent   |           |  (platform)  |  + WBA   | Moderator |
+----------+           +--------------+          +-----------+
~~~
{: #fig-intermediary title="A device agent reaching an origin through a platform"}

In {{fig-direct}}, the agent connects directly. It still has an Anchor
relationship and can present a Credential without an intermediary signature.

~~~aasvg
+----------+
|  Anchor  |
+----+-----+
     | Endorsement
     v
+----------+                                    +-----------+
|  Device  +--------------- MOLE -------------->|   Origin  |
|  agent   |                                    | Moderator |
+----------+                                    +-----------+
~~~
{: #fig-direct title="A direct connection with an Anchor off the traffic path"}

A platform can endorse a hosted workload, and an Anchor with a user
relationship can endorse its users' agents. {{fig-stacked}} illustrates the
intended scopes of a stacked arrangement. MOLE(W) and MOLE(U) denote workload
and user budgets. They do not denote visible identifiers. This is a deployment
goal, not a defined composition of presentations or a shared-budget mechanism.

~~~aasvg
+---------------------+       +---------------------+
|   Workload Anchor   |       |     User Anchor     |
+----------+----------+       +----------+----------+
           | Endorsement                 | Endorsement
           v                             v
+--------------------------------------------------+
| Platform                                         |
|  +--------------------------------------------+  |
|  | Tenant workload with user agents           |  |
|  +----------------------+---------------------+  |
|                         |                        |
|                         v                        |
|                 +--------------+                 |
|                 | Intermediary +-----------------+---+
|                 +--------------+                 |   |
+--------------------------------------------------+   |
                                                       | MOLE(W)
                                                       | MOLE(U)
                                                       | + WBA
                                                       v
                                                 +-----------+
                                                 |   Origin  |
                                                 | Moderator |
                                                 +-----------+
~~~
{: #fig-stacked title="Workload and user budget scopes in a proposed stacked deployment"}

An origin may delegate the Moderator to a CDN or similar service that fronts
it. Any of the topologies above can sit in front of it. The Moderator's Anchor
Set and policy are then shared across the origins the CDN serves under that
configuration, which makes the anonymity set larger. A Credential issued by
the CDN is valid at every origin it moderates under the same context. The CDN
sees presentations across those origins. MOLE keeps them unlinkable, but
source addresses and timing remain visible to it (see {{security}}).

A request may also cross more than one intermediary, for instance a device
agent behind an enterprise ZTNA gateway whose traffic then leaves through a
consumer VPN. Only an intermediary that terminates HTTP can sign. The origin
can attribute the request to every intermediary that signs, and sees a tunnel
only as a source address. Aggregate limits apply per signer. The MOLE
presentations are unaffected, because the Anchors that endorsed the workload
and the user are not tied to any intermediary on the path.

\[\[ Editor's note: Open points include:

* Budget allocation: how to apply limits to users and individual agents,
  including a user with several agents or a workload with many users.
* Concurrent requests: one Credential chain cannot be used in parallel.
  Several Credentials provide separate budgets. Splitting a shared budget
  and enforcing workload and user limits together need further work. One
  option is for an Anchor to grant a workload k Endorsements per epoch,
  redeemed into a pool of k Credentials, one per concurrent agent. How to
  choose k, and whether the origin should see it, is open.
* Intermediary signing: signing forwarded requests on behalf of others may
  require a profile of {{WBA-HTTPSIG}}, including its binding to MOLE
  presentations (see {{security}}).
* Chained intermediaries: when several HTTP-terminating intermediaries sign,
  which signers' aggregate limits the origin should apply.
  \]\]

# Security Considerations {#security}

The security of MOLE itself is treated in {{MOLE-ARCH}} and
{{MOLE-PROTOCOLS}}.

## Provider collusion and metadata

Anonymous presentations do not prevent a forwarding
provider that knows the user from disclosing which requests are theirs to the
origin. The deployment does not protect against this provider-origin collusion.

Unlinkable presentations do not hide source addresses, cookies, request
contents, or timing. A platform signature also narrows the anonymity set.
Origins must not infer that issuer hiding conceals an intermediary already
identified by its signature or address.

## Anchor issuance and trust

An Anchor controls how many Endorsements exist. The origin's recourse against
one that does not constrain issuance is to drop it from the Anchor Set.
Credential lifetime and retirement determine how long existing budgets remain
usable. Endorsement reuse and replay protection depend on the protocol chosen.

## Presentation reuse by intermediaries

An HTTP intermediary that can read a presentation in transit may try to use it
on a request of its own before the client's request arrives. Single-use
verification prevents two accepted uses, but does not decide which request is
authorized. This deployment therefore needs request-bound presentations that
cover all request components affecting authorization, including the body where
relevant. Request binding does not prevent an intermediary from racing an
identical request or withholding it.

## Binding signatures to presentations

A composition profile should also require the Web Bot Auth signature to cover
the field carrying the MOLE presentation, along with the relevant request
components. This binds the signer's attribution to that presentation. It does
not replace request binding against a malicious signer. This document does
not define the profile or claim that existing signatures provide this binding.

## Multiple budgets across Anchors

A user or workload with Endorsements from several accepted Anchors can obtain
several independent budgets. Unlinkable presentations do not let the origin
detect that those budgets belong to the same party. Anchor Set selection and
issuance policy affect the exposure but do not enforce one budget per user
across Anchors.

## Unsigned requests

An absent signature is not evidence of an attack. Direct clients and clients
behind tunnels may have no platform signature to offer. For public resources,
origins should preserve the baseline access described in {{principles}}, while
applying capacity and behavior-based controls.

# IANA Considerations {#iana}

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
