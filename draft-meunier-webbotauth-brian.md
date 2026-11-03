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
  PRIVACYPASS: RFC9576
  HTTP: RFC9110
  HTTPSIG: RFC9421
  INTERNET-FOR-END-USER: RFC8890
  IPv6: RFC8200
  MOLE-ARCH: I-D.draft-jms-MOLE-architecture
  MOLE-PROTOCOLS: I-D.draft-jms-MOLE-protocols
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

...

--- abstract

Origins rate limit requests by address or by the platform they come through.
Behind a shared platform, such as an AI agent service, a VPN, or a network
operator, every user then shares the same fate. This document describes the
problem of limiting individual users' agents without requiring a stable
identifier, and sketches a direction built on {{MOLE-ARCH}}: Anchors, often
operated by platforms, endorse agents, and origins apply anonymous per-agent
limits.
{{WBA-HTTPSIG}} can identify platforms for separate aggregate limits. It is a
companion to {{WBA-ABA}}.

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
{{principles}} develops the reasoning behind this position.

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

MOLE will not solve every use case served by identifiers. Identity is still
useful when an application needs it. Both MOLE and identity should be provided
willingly by the user.

# Conventions and Definitions

## Terminology {#terminology}

The system has parties on the user side, parties on the origin side, and a
platform in between. This section defines them, then maps them to the roles of
{{MOLE-ARCH}}.

User:
: A person on whose behalf requests are made. A user has an existing
  relationship with an Anchor, such as an account, a subscription, or an
  employment contract. A platform is one kind of Anchor a user may have a
  relationship with, but not the only kind.

Agent:
: Software that makes HTTP requests, such as a browser, an AI assistant, or
  another automated client. An agent either acts on behalf of one user at a
  time, or is a bot. Agents that act for a user are of two kinds,
  distinguished by where they run.

Device-resident agent:
: An agent that runs on hardware the user controls, such as a browser, a
  local AI assistant, or a command-line tool. It may connect to the origin
  directly, or through a platform's intermediary.

Remote agent:
: An agent that runs on infrastructure operated by a platform, such as a
  hosted AI assistant or a cloud browser. It runs within a workload owned by
  a tenant of the platform, and shares its source addresses, and possibly its
  TLS stack, with the remote agents of other users.

Tenant:
: A customer of a platform that deploys workloads on it. The tenant owns the
  agents that run in its workloads, and has its own relationship with the
  users those agents serve. The tenant sits between the platform, which knows
  the tenant but not its users, and the user, who knows the tenant's
  application but not the platform behind it. A tenant may act as an Anchor
  for its users, but need not.

Workload:
: An application deployed by a tenant on a platform that runs remote agents
  for the tenant's users. A platform typically hosts many workloads from many
  tenants, and a tenant may deploy several. The users of one workload share
  an application context, so their agents tend to behave alike. Agents of
  different workloads do not. The workload is therefore the unit a platform
  endorses: each workload is endorsed separately and has its own budget at an
  origin, even when one tenant owns several. A workload whose agents act for
  no user is a bot workload, and is identified rather than endorsed.

Bot:
: An agent with no user behind it, such as a crawler, an indexer, or a
  monitor, or a workload made of such agents. A bot acts for its operator.
  Having no user to shield, a bot identifies rather than being moderated
  anonymously: with its operator's own {{WBA-HTTPSIG}} identity when it runs
  outside any platform, and, when it runs on a platform, either with the
  operator's identity or with a per-tenant key the platform signs with on
  the operator's behalf. See {{system}}.

Platform:
: An operator that has a relationship with tenants, with users, or with
  both. A platform may host its tenants' workloads, may carry the traffic of
  device-resident agents through an intermediary, or may do neither and only
  have a relationship with users whose agents connect to origins directly.
  Examples are AI agent services, network operators, VPN providers, and
  enterprise ZTNA gateways. A platform identifies its intermediary to origins
  with {{WBA-HTTPSIG}}, and is well placed to operate a MOLE Anchor for its
  tenants' workloads or for its users.

Intermediary:
: The platform component that handles requests on behalf of agents, such as a
  reverse proxy, a gateway, or a tunnel endpoint. Intermediaries can be
  chained: a request from a device-resident agent may pass through the
  intermediaries of several platforms before it reaches an origin. The origin
  can attribute the request to the last intermediary on the path, and to any
  earlier intermediary that identifies itself with {{WBA-HTTPSIG}}. Only an
  intermediary that terminates HTTP can sign; a tunnel endpoint cannot.
  {{WBA-HTTPSIG}} defines agents signing their own requests. This document
  assumes an intermediary may also sign requests it forwards on behalf of
  others, which may require a profile of that protocol.

Origin:
: The server that receives agents' requests and applies rate-limit policy, in
  the sense of {{Section 3.6 of HTTP}}. An origin operates a MOLE Moderator,
  or delegates that function to a party acting on its behalf such as a CDN.

MOLE Anchor:
: The {{MOLE-ARCH}} role that grants Endorsements to Clients. An Anchor
  grants scarce Endorsements on some basis of its own. A platform acting as
  Anchor endorses its tenants' workloads, based on its relationship with the
  tenant. An Anchor for users endorses users' agents, based on a relationship
  with the user. That Anchor may be the tenant, a platform the user
  subscribes to such as a network operator or a VPN provider, or a party
  independent of any platform, as in {{WBA-ABA}}.

  One Endorsement yields one Credential per Moderator per issuance epoch:
  its redemption nullifier is scoped to the Moderator and the epoch, so the
  same Endorsement can be redeemed once at each Moderator and again in the
  next epoch ({{MOLE-PROTOCOLS}}). The Anchor sets how many Endorsements a
  user or workload receives per epoch, and {{MOLE-ARCH}} requires it to
  constrain that number. An Anchor in an origin's Anchor Set is expected to
  grant Endorsements on the basis of a relationship that is costly to
  duplicate, so that obtaining a second budget means obtaining a second
  account, subscription, or employer. The origin's recourse against an
  Anchor that does not is to drop it from the Anchor Set.

MOLE Moderator:
: The {{MOLE-ARCH}} role that accepts Endorsement redemptions, issues
  Credentials, and evaluates Credential presentations against a policy. Here
  the Moderator is operated by or for an origin.

# Principles {#principles}

As Web Bot Auth finds adoption beyond crawlers, there have been many proposals
to ask agents for more granular identity, ranging from a session identity to a
user identity. The motivation is origins wanting to manage capacity and
traffic at finer granularity.

This is not a new problem. Origins have treated IP
addresses as identities for years and made access decisions (allow, block, or
throttle) on that basis. {{WBA-HTTPSIG}} was motivated by this problem,
so that origins could key policy off a platform's identity rather than its addresses.

It is natural for origins to want to refine that policy down to an individual
workload, agent, or user. However, asking agents to disclose finer-grained identity
runs into four problems.

Security:
: Absent authentication, agents can lie about or spoof identifying
  information. Unauthenticated identities are hints, not the basis for policy.
  There are numerous examples of this in practice, with User-Agent spoofing
  being the most prevalent.

Equitability and scarcity:
: Authenticating identity requires a root of trust. That discriminates
  against agents that cannot obtain one or otherwise refuse to authenticate.
  At the same time, origins need scarcity at the source to prevent abuse,
  and scarcity relies on a root of trust. A system cannot be universally
  equitable while enforcing scarcity.

Privacy:
: Revealing more about the user lets origins track that user's activity
  across agents and across sessions. Identities, once disclosed, can be
  linked.

Agency:
: Requiring agents to disclose identifying information before accessing
  content does not let users experience the web on their own terms. Signals
  must come from agents with consent, but that consent is coerced when an
  origin strictly requires the signal to grant any access at all.

The last two problems fix the floor of any acceptable design. Every public
resource serves a baseline level of access to a request that discloses
nothing, possibly heavily rate limited, and scarcity applies only above that
floor. An agent with no root of trust is limited, never excluded. That is the
equitability the system can offer while still enforcing scarcity. Permissioned
resources, which require a principal, have no public floor and are discussed
in {{processing}}.

Naturally, this casts privacy and security at odds with one another. However,
it does not need to be this way. Privacy partitioning {{?RFC9614}}, the process of separating
the verifiable who from the what, offers a way forward. With privacy partitioning,
the parties that know who a user is, namely the user and their Anchor, are
separated from the party that sees what the user does, namely the origin.

Once the identity and action are separated, there are two ways to deal with
what the origin can no longer see: (1) either some party assumes responsibility
for the information withheld, as a relay that must itself prevent abuse, or (2)
there is a privacy-preserving way for the origin to obtain the property it
needs without the information itself. MOLE, and related technologies like
Privacy Pass, take the second route: an Anchor endorses agents, and an origin
moderates them without learning which agent it is moderating. For this to
hold, Endorsements must be scarce: an Anchor limits how many it grants per
user or workload per epoch, and grants them on the basis of a relationship
that is costly to duplicate. Scarcity is a requirement on Anchors, and an
origin enforces it by choosing which Anchors it accepts.

If origins moderate without identifying, the question then becomes at what granularity
moderation is reasonable. Platform identity with workload moderation? Workload
identity with user moderation? A workload is used by many users, so moderating
only at the workload brings back the noisy neighbour problem, though confined
to that workload rather than the whole platform.

The useful question is therefore what makes a noisy neighbour unacceptable. Two
otherwise identical users sharing the same workload is different from two users
sharing the same IP address because they use the same agent hosting platform. This
is also different from two differently behaving users sharing the same workload.
The boundary for moderation is quite reasonably where behaviour starts to differ.
In other words, the noisy neighbour is acceptable up to the point where the
behaviour of one affects the behaviour of another.

Intermediaries that withhold identifying information have an active role in keeping
the behaviour behind them uniform. Where an intermediary cannot do so, such as a remote
agent platform which cannot enforce uniformity across workloads it does not control,
assisting in moderation of the granular units is essential. Platforms therefore assist
in moderation of workloads, and workloads assist in moderation of their users.

Altogether, this leads to the following principles:

1. A platform can identify itself to origins, using {{WBA-HTTPSIG}}, rather than
   leaving origins to infer a platform from IP addresses. Bots and bot
   workloads on the platform identify under that identity or under a
   per-tenant key. Bots elsewhere identify with their operator's own
   {{WBA-HTTPSIG}} identity.
2. A platform helps origins moderate its tenants' workloads. As an Anchor it
   endorses each workload whose agents act for users, and the workload
   exchanges its Endorsements for Credentials that the origin moderates. A
   tenant identifies its workload only if asked, and with consent of the
   tenant.
3. A workload helps origins moderate its individual users. The user's agent
   presents a Credential derived from an Endorsement by the user's Anchor,
   and the workload lets that presentation reach the origin. The user's
   Anchor may be the tenant, but need not be. A user identifies themselves
   only if asked, and with consent.

Implementing these principles requires no new protocol mechanism. MOLE {{MOLE-ARCH}},
Privacy Pass {{PRIVACYPASS}}, and Web Bot Auth {{WBA-HTTPSIG}} exist and compose to
authenticate a request with graduated disclosure of identifying information.

# Architecture shape {#system}

Applying these principles to a system yields one with the following simple
property: bots are moderated with identities and user-associated agents
are moderated with MOLE. The table below elaborates.

| Party                                  | Identifies or is moderated with                        | Origin moderates                    |
|----------------------------------------|--------------------------------------------------------|-------------------------------------|
| Platform                               | {{WBA-HTTPSIG}} signature from its intermediary        | the platform's traffic in aggregate |
| Bot running on a platform              | the platform's {{WBA-HTTPSIG}} signature               | as part of the platform             |
| Bot running elsewhere                  | its operator's {{WBA-HTTPSIG}} signature               | the operator's traffic in aggregate |
| Bot workload on a platform             | the operator's, or a per-tenant, {{WBA-HTTPSIG}} signature | that workload's traffic in aggregate |
| Agent for a user, workload granularity | MOLE Credential from a platform Endorsement of its workload | the workload                   |
| Agent for a user, user granularity     | MOLE Credential from the user's Anchor's Endorsement of the agent | the individual user      |
| Device-resident agent                  | MOLE Credential from the user's Anchor's Endorsement   | the individual user                 |
{: #who-does-what title="How each party identifies or is moderated"}

## Bot moderation {#wba}

A platform signs the requests that leave its intermediary with
{{WBA-HTTPSIG}}. The origin learns which platform a request came through, and
can limit that platform's traffic in aggregate. This replaces the origin's
inference of a platform from its IP addresses.

Bots have no user to shield, and their operators can tolerate attribution. A
bot therefore identifies rather than being moderated anonymously. A bot
running elsewhere signs with its operator's own {{WBA-HTTPSIG}} identity. A
bot workload on a platform identifies too, either with the operator's own
key or with a per-tenant key the platform signs with on its behalf, so that
the origin can limit that workload in aggregate without confusing it with the
platform's user-facing workloads. The rule is about the user, not the
software: a workload with users behind it is moderated with MOLE, a workload
with none is identified. Bots do not use MOLE in this system. The alternative,
moderating bots anonymously for operators who cannot tolerate attribution, is
what {{WBA-ABA}} describes.

## User agent moderation {#mole}

An agent acting for a user does not identify itself, the tenant, or the user.
It presents MOLE Credentials so that the origin can moderate it at one or both
of two grains.

At workload granularity, the platform is the Anchor. It endorses each
workload separately, so a tenant with several workloads has several budgets.
A Credential chain is strictly serialised, with one presentation in flight
at a time, so a workload running many agents cannot share a single
Credential. The platform instead grants a workload k Endorsements per epoch,
the workload redeems them into a pool of k Credentials, and each agent
presents from its own. The value of k is the workload's concurrency budget
and the platform's lever on the tenant. A workload may equally route its
agents' requests through one presenting component that serialises them over
a few chains. Either way the origin moderates the workload: a per-workload
rate limit or budget. A noisy neighbour within the workload affects only
that workload, whose users share an application context, and not the rest of
the platform. The tenant identifies its workload only if asked, and with
consent.

At user granularity, an Anchor with a relationship to the user endorses the
user's agent, typically with one Endorsement per user. The agent redeems its
Endorsement at the origin for a Credential, and presents and updates it as it
makes requests. The origin moderates the individual user. The user identifies
themselves only if asked, and with consent.

The user's Anchor is not tied to the platform the agent runs on. It may be the
tenant whose workload the agent belongs to. It may be a platform the user
subscribes to, such as a network operator, a VPN provider, or a ZTNA gateway.
It may be independent of any platform. For a remote agent, the workload's role
is to let the agent's presentation reach the origin, not to be the Anchor. For
a device-resident agent, the user's Anchor is typically a user-side platform,
and the agent may connect to the origin directly or through that platform's
intermediary.

## Moderation limits {#limits-protect}

Each limit constrains a different party, and each is backed by a different
mechanism.

The aggregate limit constrains the signer, whoever that is: a platform, a bot
operator, or a tenant signing with its own key. It is available only when a
signing intermediary is on the path, and it says nothing about any Anchor.

The Anchor is constrained by three things: membership of the origin's Anchor
Set, which the origin curates; the initial balance and top-up policy, which
the origin can keep cautious for Anchors it trusts less and grow through the
session as {{dynamics}} describes; and the origin's ability to drop an Anchor
from the set or retire a credential context, which expires every Credential
descending from it. In the direct-connection shape only these apply, since
there is no signer, and an origin that accepts independent Anchors is
trusting the set it curates and nothing else.

The per-workload limit constrains a workload, and protects the origin and
the platform's other tenants from one that misbehaves. The per-user limit
gives fairness among an honest Anchor's users, whether or not their traffic
passes through any platform.

An agent holds Endorsements from the Anchors above it, but its requests may
pass through the intermediaries of platforms unrelated to those Anchors, for
instance a device-resident agent endorsed by an AI agent service and connected
through a VPN.

# Deployment shapes {#shapes}

The parties of {{terminology}} combine into a small number of recurring
topologies that can be instantiated according to the principles in {{principles}}.
This section walks through the ones seen in practice. In each, the
question is the same: how does an origin moderate. {{topologies}} summarises them,
and the subsections give details. {{multiple-platforms}} shows two of them side by
side, a device-resident agent connecting directly and a consumer assistant operated
end to end, to illustrate that Anchors of different kinds share one Anchor Set at the origin.

One freedom holds across every shape. The user's Anchor can always be a party
separate from the platform and the tenant. The shapes below name the party
that most naturally plays that role, and the table calls it the typical user
Anchor, but in each case the user's agent may instead hold an Endorsement from
an unrelated Anchor, and the origin cannot tell the difference.

In the figures, MOLE(U) denotes a presentation that lets the origin moderate
user U, and MOLE(W) one that lets it moderate workload W, deployed by tenant
T. The notation names what is moderated, not which Anchor endorsed it.
WBA(P) denotes the {{WBA-HTTPSIG}} signature of platform P, naming the party
the origin can attribute the request to.

| Topology                                  | WBA signer          | Workload Anchor | Typical user Anchor           | Origin moderates            |
|-------------------------------------------|---------------------|-----------------|-------------------------------|-----------------------------|
| Device agent via user-side platform       | user-side platform  | none            | user-side platform            | platform aggregate, user    |
| Device agent connecting directly          | none                | none            | any                           | user                        |
| Consumer AI assistant, one operator       | the operator        | none            | the operator                  | operator aggregate, user    |
| Remote agent platform, workload granularity | the platform      | the platform    | none                          | platform aggregate, workload |
| Remote agent platform, stacked            | the platform        | the platform    | any, separate from platform   | aggregate, workload, user   |
| Enterprise ZTNA                           | ZTNA gateway        | none            | ZTNA gateway                  | enterprise aggregate, employee |
| CDN-operated Moderator                    | as above            | as above        | as above                      | same, across many origins   |
| Chained intermediaries                    | each HTTP-terminating hop | as above  | as above                      | per signer, plus user       |
| Bot, or bot workload                      | operator, or per-tenant key | none    | none                          | aggregate only              |
{: #topologies title="Recurring topologies"}

~~~aasvg
                     +---------------------+
                     |      Platform A     |
                     |  +--------------+   |
     +- Endorsement -+--+   Anchor A   |   |
     |               |  +--------------+   |
     |               +---------------------+
     v
+----------+
|  Device  +----------------- MOLE(U) ------------------+
| agent A  |                                            |
+----------+                                            |
                                                        |
                                                        |  +-----------+
                     +---------------------+            +->|   Origin  |
                     |      Platform B     |            |  |           |
                     |  +--------------+   |            |  | Moderator |
                     |  |   Anchor B   |   |            |  |           |
                     |  +------+-------+   |            |  | Set: A, B |
                     |         |           |            |  +-----------+
                     |         v           |            |
                     |  +--------------+   |            |
                     |  |    Remote    |   |            |
                     |  |   agent B    +---+------------+
                     |  +--------------+   | MOLE(U)+WBA(B)
                     |                     |
                     +---------------------+
~~~
{: #multiple-platforms title="A device-resident agent connecting directly and a remote agent hosted by its platform, reaching the same origin"}

## Device-resident agent through a user-side platform {#shape-user-side}

A browser, a local AI assistant, or a command-line tool runs on the user's
device. The user's traffic leaves through a platform that has a relationship
with the user: a network operator, a VPN provider, or a consumer privacy
proxy. That platform signs the traffic leaving its intermediary with Web Bot
Auth and, in the typical case, acts as Anchor for its users. The user's
Anchor may equally be another party, in which case the platform only signs.
The origin applies an aggregate limit to the platform and a per-user limit
through MOLE, without learning which subscriber is which, and without knowing
whether the signer and the Anchor are the same party. {{single-platform}}
shows this shape. It is the direct answer to the network operator and VPN
cases of {{intro}}.

~~~aasvg
                     +---------------------+
                     |      Platform P     |
                     |  +--------------+   |
     +- Endorsement -+--+    Anchor    |   |
     |               |  +--------------+   |
     |               |                     |
     v               |                     |               +-----------+
+----------+         |  +--------------+   |               |   Origin  |
|  Device  +-MOLE(U)-+->| Intermediary +---+MOLE(U)+WBA(P)>|           |
|  agent   |         |  +--------------+   |               | Moderator |
+----------+         +---------------------+               +-----------+
~~~
{: #single-platform title="A device-resident agent reaching an origin through its endorsing platform"}

## Device-resident agent connecting directly {#shape-direct}

The same agent connects to the origin without any platform on the path. There
is no Web Bot Auth signature and no aggregate limit. The agent still presents
a MOLE Credential derived from an Endorsement by some Anchor with a user
relationship, which may be a platform the user subscribes to even though its
intermediary is not used, or an independent Anchor. The origin can offer this
agent the same per-user limit as any other, which is what lets an origin stop
treating a bare IP address as the user. {{fig-direct}} shows this shape;
platform A in {{multiple-platforms}} plays the same role.

~~~aasvg
+----------+
| Anchor U |
+----+-----+
     | Endorsement
     v
+----------+                                        +-----------+
|  Device  +---------------- MOLE(U) -------------->|   Origin  |
|  agent   |             no WBA signature           | Moderator |
+----------+                                        +-----------+
~~~
{: #fig-direct title="A device-resident agent connecting directly, endorsed by an Anchor off the path"}

## Consumer AI assistant operated end to end {#shape-collapsed}

A single company operates the platform, the only workload on it, and the user
relationship. Platform and tenant collapse into one party, and that party is
the natural user Anchor, though the user's agent may instead hold an
Endorsement from an Anchor the operator has nothing to do with. The operator
signs its traffic with Web Bot Auth, so the origin moderates the operator in
aggregate and, through the user's Credential, each user individually. The
tenant level disappears because there is one workload; the origin has no
reason to moderate it separately from the platform. {{fig-collapsed}} shows
this shape with the operator as Anchor; platform B in {{multiple-platforms}}
is another instance.

~~~aasvg
+-----------------------------------------+
| Operator O: platform, tenant, Anchor U  |
|                                         |
|  +----------+                           |
|  | Anchor U |                           |
|  +----+-----+                           |
|       | Endorsement                     |
|       v                                 |
|  +----------+       +--------------+    |            +-----------+
|  |  Remote  +------>| Intermediary +----+- MOLE(U) ->|   Origin  |
|  | agent U  |MOLE(U)+--------------+    |  +WBA(O)   | Moderator |
|  +----------+                           |            +-----------+
+-----------------------------------------+
~~~
{: #fig-collapsed title="A consumer AI assistant whose operator is platform, tenant, and Anchor at once"}

## Remote agent platform at workload granularity {#shape-tenant}

A platform hosts workloads from many tenants, each serving its own users. The
platform signs with Web Bot Auth and acts as Anchor for its tenants'
workloads, granting each workload k Endorsements per epoch. The workload
redeems them into a pool of Credentials, one per concurrent agent, and the
agents present from the pool, so the origin moderates each workload. The
platform does not know the tenants' users and does not endorse them. A noisy
workload is throttled on its own, which addresses the case where a remote
agent platform was blocked wholesale because unrelated users shared its
addresses. The tenant identifies its workload only if asked. {{fig-tenant}}
shows this shape, with the agents of one workload each presenting a
Credential from the workload's pool.

~~~aasvg
+------------------------------------------+
| Platform P                               |
|  +----------+                            |
|  | Anchor P |                            |
|  +----+-----+                            |
|       | k Endorsements (workload W)      |
|       v                                  |
|  +---------------------+                 |
|  |Workload W (tenant T)|                 |
|  | k Credentials, one  |                 |
|  | per agent           |                 |
|  |  +-------+-------+  |                 |
|  |  | Agent | Agent |  |                 |
|  |  +---+---+---+---+  |                 |
|  +------+-------+------+                 |
|         |       |                        |
|         v       v                        |
|     +--------------+                     |            +-----------+
|     | Intermediary +---------------------+- MOLE(W) ->|   Origin  |
|     +--------------+                     |  +WBA(P)   | Moderator |
+------------------------------------------+            +-----------+
~~~
{: #fig-tenant title="A remote agent platform endorsing a workload with a pool of Credentials; the origin moderates the workload"}

## Remote agent platform with stacked moderation {#shape-stacked}

This is the shape in {{shape-tenant}}, plus a user-level Credential. The
user's Anchor is separate from both the platform and the tenant: any party
with a relationship to the user can endorse the user's agent, and the
Endorsement reaches the agent without involving the hosting platform. The
tenant may happen to be that party, but nothing in the shape requires it. The
request carries the platform's signature, a presentation of the workload's
Credential, and a presentation of the user's Credential. {{fig-stacked}}
shows this shape with the user's Anchor outside the platform. It is the full
manifestation of the principles in {{principles}}.

~~~aasvg
+----------+
| Anchor U |
+----+-----+
     | Endorsement (user U)
     | +-----------------------------------------------------------+
     | | Platform P                                                |
     | |                                                           |
     | |  +----------+   Endorsement   +-------------------------+ |
     | |  | Anchor P +---------------->| Workload (tenant T)     | |
     | |  +----------+                 |                         | |
     | |                               |  +-----------+          | |
     +-+-------------------------------+->|   Agent   |          | |
       |                               |  | (user U)  |          | |
       |                               |  +-----+-----+          | |
       |                               +--------+----------------+ |
       |                                        | MOLE(W)+MOLE(U) |
       |                                        v                  |
       |                                 +--------------+          |
       |                                 | Intermediary |          |
       |                                 +------+-------+          |
       +----------------------------------------+------------------+
                                                | +WBA(P)
                                                v
                                        +--------------+
                                        |    Origin    |
                                        |  Moderator   |
                                        +--------------+
~~~
{: #fig-stacked title="A remote agent moderated at workload and user granularity behind an identified platform, with the user's Anchor elsewhere"}

## Enterprise ZTNA {#shape-ztna}

Employees' device-resident agents, including browsers and local assistants,
reach the internet through an enterprise ZTNA gateway. The gateway knows every
employee but has no reason to expose them to origins. It signs with Web Bot
Auth as the enterprise and is the natural Anchor for its employees' agents,
though an employee's agent may instead be endorsed by a separate Anchor. The
origin applies an aggregate limit to the enterprise and a per-employee limit
through MOLE. It cannot single out an executive's activity, which is the ZTNA
case of {{intro}}. Structurally this is {{shape-user-side}} with an
enterprise in the platform role, as {{fig-ztna}} shows.

~~~aasvg
                     +---------------------+
                     |  Enterprise ZTNA E  |
                     |  +--------------+   |
     +- Endorsement -+--+    Anchor    |   |
     |               |  +--------------+   |
     |               |                     |
     v               |                     |               +-----------+
+----------+         |  +--------------+   |               |   Origin  |
| Employee +-MOLE(U)-+->| ZTNA gateway +---+MOLE(U)+WBA(E)>|           |
|  device  |         |  +--------------+   |               | Moderator |
+----------+         +---------------------+               +-----------+
~~~
{: #fig-ztna title="Employee devices behind an enterprise ZTNA gateway that signs and endorses"}

## CDN-operated Moderator {#shape-cdn}

An origin delegates the Moderator role to a CDN or similar service that
fronts it, as allowed by the definition of origin in {{terminology}}. The CDN
issues the Challenges, runs Redeem & Issue, verifies presentations, and
issues Updates, then forwards accepted requests to the origin. Any of the
shapes above can sit in front of it. Two things change. The Moderator's
Anchor Set and policy are shared across the origins the CDN serves under that
configuration, which makes the anonymity set larger. And a Credential issued by
the CDN is valid at every origin the CDN moderates under the same context,
since the agent presents only to the Moderator that issued it. The CDN
therefore sees presentations across many origins, which is exactly what
MOLE's unlinkability is designed to make harmless. {{fig-cdn}} shows agents
of two shapes reaching two origins through one Moderator.

~~~aasvg
+-----------+                 +-----------------+
| Device    +-- MOLE(U) ----->|                 |        +----------+
| agent     |                 |  CDN            +------->| Origin A |
+-----------+                 |  (Moderator for |        +----------+
                              |   A and B)      |
+-----------+                 |  Anchor Set,    |        +----------+
| Remote    +-MOLE(W)+WBA(P)->|  policy, keys   +------->| Origin B |
| agent on  |                 |                 |        +----------+
| platform P|                 +-----------------+
+-----------+
~~~
{: #fig-cdn title="A CDN operating the Moderator for several origins"}

## Chained intermediaries {#shape-chain}

A device-resident agent behind an enterprise ZTNA gateway whose traffic then
leaves through a consumer VPN. A remote agent whose platform drives a browser
hosted on a second platform. A remote agent platform that reaches an origin
through the origin's own CDN. In each case the request crosses more than one
intermediary. Only an intermediary that terminates HTTP can sign: a forward
proxy or gateway can, a tunnel such as a consumer VPN cannot, since it
carries the TLS session untouched. The origin can therefore attribute the
request to every HTTP-terminating intermediary that signs, and sees a tunnel
only as a source address. Aggregate limits apply per signer. The MOLE
presentations are unaffected, because the Anchors that endorsed the workload
and the user are not tied to any intermediary on the path. Which signers on
a chain the aggregate limit should apply to is an open point noted below.
{{fig-chain}} shows the first example: the ZTNA gateway signs, the VPN
cannot, and the request reaches the origin carrying MOLE(U) and WBA(E) from
the VPN's address.

~~~aasvg
+----------+
|  Device  |
|  agent   |
+----+-----+
     | MOLE(U)
     v
+------------+
|Enterprise E|
| ZTNA gw    |
| signs WBA  |
+-----+------+
      | MOLE(U) + WBA(E)
      v
+------------+
|   VPN V    |
| (tunnel,   |
|  no WBA)   |
+-----+------+
      | MOLE(U) + WBA(E)
      |   from V's address
      v
+------------+
|   Origin   |
| Moderator  |
+------------+
~~~
{: #fig-chain title="A request crossing two intermediaries; only the one terminating HTTP can sign"}

## Bots {#shape-bot}

A crawler, an indexer, or a monitor has no user and does not use MOLE. On a
platform, its requests carry the platform's Web Bot Auth signature and are
limited as part of the platform's aggregate. Off a platform, its operator
signs with its own identity and the origin limits the operator's traffic in
aggregate. A bot operator that wants a finer limit than the aggregate can run
each crawl as a separate signing key, since identification, not anonymity, is
the bot's mode. {{fig-bot}} shows both cases.

~~~aasvg
+-----------------------------------+
| Platform P                        |
|  +---------+    +--------------+  |
|  |   Bot   +--->| Intermediary +--+----WBA(P)-----+
|  +---------+    +--------------+  |               |
+-----------------------------------+               |   +-----------+
                                                    +-->|   Origin  |
+-----------+                                       |   |           |
| Bot of    +-------------- WBA(O) ----------------+   | Moderator |
| operator O|                                           +-----------+
+-----------+
~~~
{: #fig-bot title="Bots identify with the platform's or their operator's Web Bot Auth signature"}

\[\[ Editor's note: Open points include:

* Chained intermediaries: when several HTTP-terminating intermediaries sign,
  which signers' aggregate limits the origin should apply.
* Intermediary signing: {{WBA-HTTPSIG}} covers agents signing their own
  requests. Signing forwarded requests on behalf of others, with a signature
  that covers the Authorization field carrying a presentation, may need a
  profile of that protocol.
* Choosing k: how a platform sizes a workload's Endorsement count per epoch,
  and whether the origin should see k or only the per-workload budget.
* A user with several device-resident agents, or with both a device-resident
  and a remote agent on the same platform, needs the per-user budget split
  between them, or one Endorsement per agent from the same Anchor.
* Whether a bot should ever be moderated with MOLE instead of identifying, as
  {{WBA-ABA}} proposes for bots whose operators cannot tolerate attribution.
  \]\]

# Moderation at the origin {#processing}

The shapes of {{shapes}} differ in who signs, who endorses, how many
intermediaries sit on the path, and whether the agent runs on the user's
device or on a platform. None of that is visible to the origin, and none of it
needs to be. The origin sees a request that carries zero or more
{{WBA-HTTPSIG}} signatures and zero or more MOLE presentations, each answering
a Challenge the origin itself issued on an earlier response. It moderates from
those alone. {{origin-view}} lists what the origin can and cannot see.

| The origin sees                                  | The origin does not see                               |
|--------------------------------------------------|-------------------------------------------------------|
| which platform or operator signed, if any        | whether the agent is device-resident, remote, or a bot |
| that a valid presentation moderates a workload   | which workload or tenant it is                        |
| that a valid presentation moderates a user       | which user it is, or which Anchor endorsed them       |
| that the Anchor is in the Anchor Set             | which Anchor in the set                               |
| the spend amount and top-up the presentation answers | the hidden balance                                |
| the outcome of its own policy for this request   | any earlier request from the same agent               |
{: #origin-view title="What an origin sees"}

This is what lets one implementation serve every shape. An origin does not
configure itself per platform or per topology. It configures a Moderator
policy, including an Anchor Set, one or more credential contexts, and the
amounts it charges and refunds. Everything else follows from what each request
carries.

## Processing a request {#processing-steps}

Every request is an exchange in which the client offers and the origin either
accepts or challenges. The client offers whatever it is willing to disclose
on this request. The origin accepts the offer if it is enough for what this
request needs, and otherwise challenges for more. What it challenges for
depends on the need, not on the topology behind the request. This is the
graduated disclosure of {{principles}} made operational.

### Client offer {#offers}

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
  origin asked for, at workload grain, user grain, or both. It discloses
  nothing about who, only that a budget exists and is being spent. It is
  offered only in answer to a presentation Challenge.

A CredentialRequest:
: A Redemption of an Endorsement from some Anchor in the Anchor Set, with a
  request for a Credential. It discloses membership of the set, not the
  Anchor. It is offered only in answer to a redemption Challenge.

Identity:
: Who the user is, through user authentication or authorization such as
  {{AAUTH}} or {{OAUTH-XAA}}; which workload this is, through a workload
  identity such as {{WIMSE-AIMS}} or a tenant identifying its workload; or
  which session this is. Identity is offered only when asked, and with the
  consent of the party it identifies, as {{principles}} requires.

### Offer acceptance {#accepting}

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

### Offer challenge {#challenging}

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
  workload, user, or both, and may offer several endorsement or credential
  types at once; the agent ignores types it does not support and answers
  one it does.

Identity:
: The origin needs to know who, because the resource is permissioned: an
  account-bound action, a paid API, a resource with per-principal access
  control. It challenges for identity, using the authentication or
  authorization scheme appropriate to the principal, and serves according to
  that identity once offered. MOLE does not satisfy this need, and the origin
  should not pretend it does by inferring identity from a presentation.

The two are not interchangeable in either direction. Identity is not a
substitute for MOLE on a public resource, and demanding it there is the coerced
disclosure that {{principles}} warns against. It also hands the origin a linkable
key it did not need. MOLE is not a substitute for identity on a permissioned
resource. An origin may need both, for instance a permissioned API that also
wants a per-user budget, and then asks for both. What it must not do is ask
for more than the request needs. The agent is free to decline any challenge.
What follows depends on the resource. A public resource keeps serving the
baseline, since that floor is what makes disclosure voluntary rather than
coerced. A permissioned resource has no public floor and may refuse. The
HTTP transport for MOLE {{MOLE-HTTP}} marks the difference: a Challenge on a
401 response means a presentation is required, a Challenge on a 200 response
means it is optional and would, for instance, raise a limit. Graduated
disclosure is the second form.

The challenge is interactive. The origin asks in response to a specific
request, and the agent answers on the request that follows. Nothing is
published in advance, and the origin chooses when, and of whom, to ask. An
origin that sees nothing of concern may never challenge. One under load may
challenge every request to a given path.

### Serving and charging {#serving}

Having accepted or challenged, the origin serves the request at the level
the accepted offer warrants and attaches an Update for each valid presentation
with the refund t it has chosen. If it wants the agent's next request
moderated, it also attaches the next presentation Challenge, since no
presentation is made without one. It records each presentation's result
atomically so a retry returns the same result and a copy of the Credential
is refused. {{decision}} summarises the exchange.

| Client offered                     | Origin needs      | Origin response                                         |
|------------------------------------|-------------------|---------------------------------------------------------|
| nothing, or a signature only       | nothing more      | serve baseline, within the aggregate if signed          |
| nothing, or a signature only       | fair share        | serve baseline; both MOLE Challenges at the wanted grain |
| nothing, or a signature only       | identity          | serve baseline or refuse; identity challenge            |
| CredentialRequest                  | any               | verify Redemption, issue Credential                     |
| valid presentation at wanted grain | fair share        | serve within that budget; Update with refund; next Challenge |
| valid presentation at one grain    | fair share, both  | serve within that budget; Challenge for the other grain |
| valid presentation                 | identity          | serve baseline; identity challenge                      |
| identity                           | identity          | serve per that identity's policy                        |
| invalid presentation               | fair share        | as nothing offered: baseline and both MOLE Challenges   |
{: #decision title="The offer and the response"}

A valid signature, in any row, additionally confines the request within the
signer's aggregate limit. It never decides whether to challenge or what to
challenge for. A device-resident agent behind a VPN and a remote agent on a
hosting platform that both offer MOLE(U) receive the same row, whether or not
a signature came with them.

## Request flow {#flow}

{{sequence}} shows the exchange of {{processing-steps}} for an origin that
needs a fair share and a client that starts by offering only a signature, as
a request moves from a remote agent through its platform's intermediary to an
origin that operates the Moderator. For a device-resident agent connecting directly,
remove the intermediary and the signature; the MOLE exchanges are unchanged.
The agent holds an Endorsement obtained earlier from its Anchor. The exchanges
follow {{MOLE-PROTOCOLS}}; the message names below are those of that document.

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

## Tuning challenges {#dynamics}

A Credential's hidden balance is what makes moderation dynamic. The chain of
Credentials that descends from one Redeem and Issue is, from the agent's point
of view, a session with the origin. The origin cannot link the presentations
in that chain to each other. It can nonetheless shape the session, because
the consequences of each request are written into the balance that the next
presentation must prove against.

[[OPEN ISSUE: there might be explicit sessions with delayed update]]

Three amounts give the origin its levers. The spend amount s is the price of
a request. The maximum top-up a is the most the origin offers to add back.
The refund t, between 0 and s + a, is chosen after the origin has seen the
request and its effect. The agent's new balance is c - s + t.

* Pricing by request. Different request classes carry different spend
  amounts: a cached page costs little, a search or an expensive API call
  costs more. The price is set by the Challenge the request answers, not by
  who sends it.
* Rewarding behaviour. A request that completes normally can be refunded in
  full, so a well-behaved agent keeps its balance. A request that trips an
  abuse signal, fails authorization, or arrives in a burst can be refunded
  partially or not at all. Over a session the balance comes to reflect the
  agent's behaviour, while the origin never learns which agent it is.
* Topping up. Setting a above zero lets an origin grow the balance of an
  agent whose behaviour it has come to trust, which bootstraps access from a
  cautious initial balance. This lets an origin accept Endorsements from
  Anchors it trusts less, since trust is built in the session rather than
  assumed at redemption.
* Refreshing. A presentation with s and a both zero spends nothing and
  returns a Credential with a fresh nullifier and the same balance. An origin
  can use this to rotate state without charging.
* Ending the session. Declining to refund ends the Credential chain and
  forfeits its remaining balance. Retiring a credential context expires every
  outstanding Credential issued under it. Both are revocation without
  identification.

Two constraints bound these levers. First, none of the four amounts, the
initial balance, s, a, or t, may vary per agent under a given policy; this is
a requirement of {{MOLE-PROTOCOLS}}. The refund is the sharpest case, since it
is chosen after the origin has seen the request and revealed in the Update. An
origin that tailored t to a suspected agent would be using the refund as an
identifier. Amounts are drawn from a small, policy-fixed set keyed on the
request class, the time window, or the load, and the Moderator's
configuration makes that set public. Second, repeated challenges with
varying s let an origin bound an agent's hidden balance. Agents limit how
often they present within a context, and present only one Credential from a
chain at a time.

In the stacked arrangement of {{fig-stacked}}, the workload's Credential and
the user's Credential carry separate balances. A misbehaving user drains their own
balance. The workload's balance reflects the behaviour of the workload as a
whole, and a tenant that fails to keep its users' behaviour uniform sees its
workload throttled before the platform's other tenants are affected.

# Security Considerations {#security}

The security of MOLE itself is treated in {{MOLE-ARCH}} and
{{MOLE-PROTOCOLS}}. This section covers what the composition with
intermediaries and {{WBA-HTTPSIG}} adds.

Presentations seen in transit:
: In every shape with an intermediary on the path, the intermediary sees the
  agent's presentation before the Moderator does. Under the policy-scoped
  profile of {{MOLE-PROTOCOLS}} a presentation is a one-use bearer
  authorization, and such an intermediary can attach it to a request of its
  own; the nullifier check ensures only one use is accepted, not which. A
  Moderator that may have intermediaries in front of its agents uses the
  request-bound profile, with a request context that distinguishes every
  request field that affects authorization, including the body. That is the
  default this document assumes.

Binding the signature to the presentation:
: Nothing in {{WBA-HTTPSIG}} or {{MOLE-PROTOCOLS}} ties a signature to the
  presentation in the same request. An intermediary that signs is expected to
  include the field carrying the presentation among the signed components,
  so that a presentation cannot be re-signed onto a different request
  without the signer being accountable for it. Without this, a stolen
  presentation paired with the thief's own signature is indistinguishable
  from an honest one, apart from the request-bound context above.

Dishonest or careless Anchors:
: An Anchor controls how many Endorsements exist. The aggregate limit on a
  signer does not constrain an Anchor that is a different party. The origin's
  protections are Anchor Set curation, a cautious initial balance and top-up
  policy for Anchors it trusts less, and the ability to drop an Anchor or
  retire a credential context. An origin that accepts an Anchor it cannot
  evaluate is trusting it with the scarcity of its own budgets.

Sybil budgets across Anchors:
: A user or workload with relationships at several Anchors in the same
  Anchor Set holds several budgets, one per Endorsement. An origin bounds
  this by the number of Anchors it accepts and by keeping initial balances
  small. It cannot detect it, since Anchor-hiding is the point.

Anchor and Moderator collusion:
: The unlinkability of a presentation to its redemption, and of
  presentations to each other, holds against a colluding Anchor and
  Moderator by construction in {{MOLE-ARCH}}. What collusion can still
  exploit is side channels: the timing between redemption and first
  presentation, the source address, and the signer. A platform that is both
  Anchor and signer sees its own users' presentations leave; it learns
  nothing from them it did not already know, but the Moderator learns the
  signer, which narrows the anonymity set to that platform's users.

Balance inference and amounts:
: Repeated Challenges with varying spend amounts let a Moderator bound a
  hidden balance, and any of the four amounts varied per agent becomes an
  identifier. Both are addressed in {{dynamics}} and in {{MOLE-PROTOCOLS}}.

Unsigned and unattributed requests:
: An absent signature is not an attack and must not be treated as one; a
  device-resident agent connecting directly has none to offer. An origin that
  refuses unsigned requests outright reintroduces the address-based policy
  this document sets out to replace.

# IANA Considerations {#iana}

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
