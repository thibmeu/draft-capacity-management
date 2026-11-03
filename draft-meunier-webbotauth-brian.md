---
title: "Attributable Platforms and Authorized Agents"
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
  HTTPSIG: RFC9421
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
disclose a stable identifier. It sketches a direction in which Anchors
endorse agents, origins authorize those agents through anonymous budgets
rather than identities, and platforms identify themselves for separate
aggregate limits.

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

# Conventions and Definitions {#terminology}

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

User:
: A person on whose behalf requests are made. A user has an existing
  relationship with an Anchor, such as an account, a subscription, or an
  employment contract.

Agent:
: Software that makes requests for one user at a time, either on the user's
  device or as a remote agent within a tenant's workload on a platform.

Bot:
: An agent with no user behind it, such as a crawler or a monitor, or a
  workload made of such agents. Having no user to shield, a bot identifies
  rather than being moderated anonymously.

Anchor:
: The MOLE role that grants Endorsements to Clients {{MOLE-ARCH}}.

Moderator:
: The MOLE role that issues Credentials and evaluates their presentations
  {{MOLE-ARCH}}.

The parties above form a hierarchy of layers. {{role-hierarchy}} shows the
delineation. Anchors are not in it: an Anchor is a role that any party with
a relationship to a layer can play.

~~~aasvg
Platform P
  |
  +-- Tenant T
  |     |
  |     +-- Workload W
  |     |     |
  |     |     +-- Agent A, acting for User U
  |     |
  |     +-- Bot workload B
  |
  +-- Intermediary
~~~
{: #role-hierarchy title="Layers of the hierarchy"}

This document also uses the following notation throughout.

Anchor X:
: An Anchor that endorses parties at layer X. For example, Anchor T endorses
  tenants, Anchor W endorses workloads, and so on.

MOLE(X):
: A MOLE Credential (or a presentation of one) derived from an Endorsement by
  Anchor X. It lets an origin moderate one party at layer X among all the
  parties at that layer that share Anchor X, without distinguishing them.
  For example, MOLE(T) is a Credential over all tenants endorsed by Anchor T.

WBA(X):
: A {{WBA-HTTPSIG}} signature by entity X, such as a platform, over a
  request. It lets an origin attribute the request to X.

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

Altogether, this leads to the following principles:

1. A platform can identify itself to origins, using {{WBA-HTTPSIG}}, rather
   than leaving origins to infer a platform from IP addresses. Bots and bot
   workloads on the platform identify under that identity or under a
   per-tenant key. Bots elsewhere identify with their own {{WBA-HTTPSIG}}
   identity.
2. A platform helps origins moderate its tenants and their workloads. It is
   the natural Anchor for them, endorsing each tenant, each workload whose
   agents act for users, or both; where another Anchor endorses them
   instead, the platform lets their presentations through. The endorsed
   party exchanges its Endorsements for Credentials that the origin
   moderates. A tenant identifies itself or its workload only if asked, and
   with consent of the tenant.
3. A workload helps origins moderate its individual users. The user's agent
   presents a Credential derived from an Endorsement by the user's Anchor,
   and the workload lets that presentation reach the origin. The user's
   Anchor may be the tenant, but need not be. A user identifies themselves
   only if asked, and with consent.

Implementing these principles requires no new protocol mechanism. MOLE
{{MOLE-ARCH}} and Web Bot Auth {{WBA-HTTPSIG}} exist and compose to
authenticate a request with graduated disclosure of identifying information.

## Capacity and identity {#capacity}

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

## Attribution and authorization {#system}

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

# Moderating access {#processing-steps}

In practice, an agentic stack will differ in many ways:
- How many and what arrangement of users, agents, workloads, tenants, and platforms exist;
- Who signs, who endorses, and how many intermediaries sit on the path; and
- Whether the agent runs on the user's device or on a platform.

By default, none of this information is known to the origin that is moderating
access to its site, and none of it needs to be. The origin sees a request that
offers some amount of identifying information by default. It can moderate on the
basis of this information, or request that more information be disclosed (through
identities or MOLE credentials).

This is what lets one implementation serve many scenarios. An origin does not
configure itself per platform or per topology. It configures a Moderator
policy, including an Anchor Set, one or more credential contexts, and the
amounts it charges and refunds. Everything else follows from an exchange,
which has four steps:

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

Every request is an exchange in which the client offers and the origin either
accepts or challenges. The client offers whatever it is willing to disclose
on this request. The origin accepts the offer if it is enough for what this
request needs, and otherwise challenges for more. What it challenges for
depends on the need, not on the topology behind the request. In this section,
the specifics of this transaction and how origins applies moderation is discussed
further.

## Client offer {#offers}

A request carries some combination of the following, any of which may be
absent.

Nothing:
: A bare request. This is the zero-disclosure baseline that every origin
  serves at some level, possibly heavily rate limited.

A signature:
: A {{WBA-HTTPSIG}} signature from a platform or bot operator. It is offered
  opportunistically. A platform adds one if it has one to add; a
  device-resident agent connecting directly has none to offer. It discloses
  the signer, not the agent, tenant, or user.

A MOLE presentation:
: Proof that the agent holds a Credential whose balance covers the spend the
  origin asked for, at tenant, workload, or user grain, or several. It discloses
  nothing about who, only that a budget exists and is being spent. It is
  offered only in answer to a presentation Challenge.

A CredentialRequest:
: A Redemption of an Endorsement from some Anchor in the Anchor Set, with a
  request for a Credential. It discloses membership of the set, not the
  Anchor. It is offered only in answer to a redemption Challenge.

Identity:
: Who the entity is, through authentication or authorization such as {{AAUTH}}
  or {{OAUTH-XAA}}; which workload this is, through a workload identity such
  as {{WIMSE-AIMS}} or a tenant identifying its workload; or which session this
  is. Identity is offered only when asked, and with the consent of the party it
  identifies, as {{principles}} requires.

## Offer acceptance {#accepting}

The origin first verifies what was offered, then decides whether it is
enough.

1. A signature, if present, is verified. A valid one attributes the request
   to its signer and places it under that signer's aggregate limit. An absent
   or invalid signature leaves the request unattributed, under whatever
   address-based limit the origin applied before. This is not a failure. An
   origin can ask for a signature with the Accept-Signature field of
   {{HTTPSIG}}, but cannot count on receiving one. When a signature and a
   presentation arrive together, the signature is expected to cover the
   field carrying the presentation, so that the signer is accountable for
   the pairing; see {{security}}.
2. A CredentialRequest, if present, is the answer to a redemption Challenge.
   The origin verifies the Redemption against the Anchor Set, checks the
   Endorsement's nullifier has not been redeemed, chooses the initial balance
   from policy, records the nullifier, and returns a Credential. This
   exchange yields a Credential, not content.
3. A presentation, if present, is the answer to a presentation Challenge.
   The origin determines which Challenge from the credential context and the
   request, confirms the key and context are current, rejects a replayed
   nullifier, checks the spend and top-up match the Challenge, and verifies
   the proof and the predicate that the balance covers the spend. A failed
   presentation is treated as not offered. Whenever an intermediary is on
   the path, the Moderator uses the request-bound profile of
   {{MOLE-PROTOCOLS}}, with a request context that covers the method, the
   target, and the body, so that a presentation seen in transit cannot be
   attached to a different request; see {{security}}.
4. Identity, if present, is verified by whatever mechanism carried it.

The offer is enough when it covers what the request class needs. For most
content, the origin needs only to manage capacity and keep users from
affecting one another, and a valid presentation at the grain it cares about
is enough. For a request that nothing in the offer covers, the origin serves
the baseline, which may be very little, and challenges.

## Offer challenge {#challenging}

The challenge names what the origin needs. Two needs are distinct, and the
origin asks for the one it has.

Capacity:
: The origin needs to limit consumption per workload or per user without
  knowing who they are. It challenges for MOLE. Because it cannot tell
  whether the agent lacks a Credential or merely did not present one, it
  issues both Challenges together: a redemption Challenge carrying a
  redemption context that names the Moderator, its policy, and a time
  window, and a presentation Challenge carrying the credential context and
  the terms of a presentation, spend s and top-up a. An agent with an Endorsement
  but no Credential answers the first and then the second. An agent with a
  Credential answers the second. An agent with neither keeps receiving the
  baseline. The origin challenges at the level of granularity it wants:
  tenant, workload, user, or several, and may offer several endorsement or credential
  types at once; the agent ignores types it does not support and answers
  one it does.

Identity:
: The origin needs to know who, because the resource is permissioned: an
  account-bound action, a paid API, a resource with per-principal access
  control. It challenges for identity, using the authentication or
  authorization scheme appropriate to the principal, and serves according to
  that identity once offered. MOLE does not satisfy this need, and the origin
  should not pretend it does by inferring identity from a presentation.

The two are not interchangeable in either direction, as {{capacity}}
explains. An origin may need both and then asks for both. What it must not do
is ask for more than the request needs, and the agent is free to decline any
challenge. A public resource keeps serving the baseline, since that floor is
what makes disclosure voluntary rather than coerced. A permissioned resource
has no public floor and may refuse.

The challenge is interactive. The origin asks in response to a specific
request, and the agent answers on the request that follows. Nothing is
published in advance, and the origin chooses when, and of whom, to ask. An
origin that sees nothing of concern may never challenge. One under load may
challenge every request to a given path.

## Serving and charging {#serving}

Having accepted or challenged, the origin serves the request at the level
the accepted offer warrants and attaches an Update for each valid presentation
with the refund t it has chosen. If it wants the agent's next request
moderated, it also attaches the next presentation Challenge, since no
presentation is made without one. It records each presentation's result
atomically so a retry returns the same result and a copy of the Credential
is refused.

## Request flow {#flow}

{{sequence}} shows the exchange of {{processing-steps}} for an origin that
needs a fair share and a client that starts by offering only a signature, as
a request moves from a remote agent through its platform's intermediary to an
origin's Moderator. The exchanges follow {{MOLE-PROTOCOLS}} (the message names
below are those of that document).

~~~
Anchor       Agent      Intermediary (P)       Origin
  |            |                   |              (Moderator)
  | Endorsement|                   |                 |
  |----------->|                   |                 |
  |            |                   |                 |
  |            | (1) request, no presentation        |
  |            |------------------>| add WBA(P)      |
  |            |                   |---------------->| verify WBA(P),
  |            |                   |                 | aggregate limit
  |            | (2) baseline response + Challenges  |
  |            |<------------------|<----------------|
  |            |   redemption: redemption_context    |
  |            |   presentation: spend s, top-up a   |
  |            |                   |                 |
  |            | (3) CredentialRequest               |
  |            |------------------>|---------------->| verify Redemption
  |            |   Redemption over Anchor Set        | over Anchor Set,
  |            |   + IssuanceRequest                 | check nullifier,
  |            |                   |                 | balance c, record
  |            | (4) CredentialResponse              |
  |            |<------------------|<----------------|
  |            | Credential (c)    |                 |
  |            |                   |                 |
  |            | (5) request + Presentation(s, a)    |
  |            |------------------>| add WBA(P)      |
  |            |                   |---------------->| verify WBA(P),
  |            |                   |                 | aggregate limit;
  |            |                   |                 | nullifier lookup;
  |            |                   |                 | VerifySpend:
  |            |                   |                 |  proof, c >= s;
  |            |                   |                 | choose refund t;
  |            |                   |                 | record; serve
  |            | (6) response + Update + Challenge   |
  |            |<------------------|<----------------|
  |            | Credential (c-s+t)|                 |
  |            |                   |                 |
  |            | (7) repeat from (5), answering each Challenge
~~~
{: #sequence title="A request moving through the system"}

1. The agent sends a request with no presentation. The intermediary adds the
   platform's {{WBA-HTTPSIG}} signature and forwards it.
2. The origin verifies the signature, if there is one, attributes the request
   to the platform, and applies its aggregate limit. Finding no presentation,
   and wanting this request class moderated, it serves the baseline level of
   access, possibly heavily rate limited, and issues both Challenges. The
   redemption Challenge carries a redemption context that names the Moderator,
   its policy, and a time window. The presentation Challenge carries the
   credential context and the terms of a presentation: a spend amount s and
   a maximum top-up a. Because the baseline was served, the Challenges travel
   on a 200 response and are optional.
3. The agent has no Credential for this Moderator, so it answers the
   redemption Challenge first. It fetches the Moderator's configuration from
   the Redeem and Issue endpoint, which lists the Anchor Set, and sends a
   CredentialRequest to that endpoint: a Redemption that proves its
   Endorsement comes from some Anchor in the set, without revealing which,
   and an IssuanceRequest for a Credential.
4. The origin verifies the Redemption, checks that the Endorsement's
   nullifier has not been redeemed before, chooses an initial balance c
   according to its policy, records the nullifier, and returns a
   CredentialResponse. The agent finalizes it into a Credential with hidden
   balance c.
5. The agent now answers the presentation Challenge. It repeats the request
   with a Presentation that proves its balance covers the spend amount s
   without revealing the balance, and includes a request for a top-up of at
   most a. The intermediary signs as before, if there is one.
6. The origin verifies the signature, if any, and applies the aggregate
   limit as in step 2. It then looks up the presentation's nullifier to reject a replay,
   verifies the proof and the predicate that the balance is at least s,
   chooses a refund t between 0 and s + a, records the result under the
   nullifier, and serves the request. The response carries an Update with
   the refund, and a new presentation Challenge for the agent's next
   request. The agent finalizes the Update into a fresh Credential with
   balance c - s + t.
7. Subsequent requests repeat from step 5, each answering the Challenge
   carried on the previous response. Each presentation uses a fresh
   nullifier, so the origin cannot link one request to the next.

Every presentation answers a Challenge. An agent never presents unprompted,
and a presentation that answers no Challenge the origin issued is invalid.
For the chain to continue, each response that the origin wants followed by a
moderated request carries the next presentation Challenge, as step 6 shows.
An origin that stops challenging stops receiving presentations, and the agent
falls back to the baseline. The Challenge is the origin's to issue, when and
to whom it chooses, in response to a request it has seen. Graduated
disclosure, as described in {{principles}}, is the origin's choice of when to
ask.

If the presentation fails verification, or the balance does not cover s, the
origin falls back to the baseline response of step 2. The agent can obtain a
new Credential only by redeeming another Endorsement, which its Anchor grants
scarcely. That is what makes the limit bind.

## Deployment examples {#examples}

The following examples separate the traffic path from the endorsement
relationship, using the notation of {{terminology}}. An Anchor may be the
platform, a tenant, or an independent party accepted by the origin. The
origin's Moderator may be operated by the origin or delegated to a provider
acting on its behalf. A platform signature allows a separate aggregate limit.
Signing forwarded requests needs the profile discussed below. A tunnel
carrying the client's TLS session untouched cannot add such a signature.

In {{fig-intermediary}}, a device agent presents MOLE(U) through the HTTP
intermediary of platform P, which adds WBA(P). Anchor U is drawn separately
to show that it need not operate that intermediary. A consumer AI service can
combine these roles and host the agent.

~~~aasvg
+----------+
| Anchor U |
+----+-----+
     | Endorsement
     v
+----------+           +--------------+          +-----------+
|  Device  +-MOLE(U)-->| Intermediary +-MOLE(U)->|   Origin  |
|  agent   |           | (platform P) |  +WBA(P) | Moderator |
+----------+           +--------------+          +-----------+
~~~
{: #fig-intermediary title="A device agent reaching an origin through platform P, endorsed by Anchor U"}

In {{fig-direct}}, the agent connects directly. It still has a relationship
with Anchor U and presents MOLE(U) without any signature on the request.

~~~aasvg
+----------+
| Anchor U |
+----+-----+
     | Endorsement
     v
+----------+                                    +-----------+
|  Device  +-------------- MOLE(U) ------------>|   Origin  |
|  agent   |             no WBA(X)              | Moderator |
+----------+                                    +-----------+
~~~
{: #fig-direct title="A direct connection with Anchor U off the traffic path"}

Anchor W, typically platform P itself, can endorse a hosted workload, and
Anchor U can endorse its users' agents. {{fig-stacked}} illustrates the
intended scopes of a stacked arrangement: the request carries MOLE(W) and
MOLE(U), which are workload and user budgets, not visible identifiers, and
WBA(P). This is a deployment goal, not a defined composition of presentations
or a shared-budget mechanism.

~~~aasvg
+---------------------+       +---------------------+
|      Anchor W       |       |      Anchor U       |
+----------+----------+       +----------+----------+
           | Endorsement                 | Endorsement
           v                             v
+--------------------------------------------------+
| Platform P                                       |
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
                                                       | +WBA(P)
                                                       v
                                                 +-----------+
                                                 |   Origin  |
                                                 | Moderator |
                                                 +-----------+
~~~
{: #fig-stacked title="Workload and user budget scopes in a proposed stacked deployment"}

An origin may delegate the Moderator to a CDN or similar service that fronts
it. Any of the examples above can sit in front of it. The Moderator's Anchor
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
