---
title: OAuth Client Metadata Lookup
abbrev: Client Metadata Lookup
category: std
docname: draft-gerber-oauth-client-metadata-lookup-latest
submissiontype: IETF
date: 2026-10-21
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
author:
 - fullname: Maxwell Gerber
   organization: Twilio
   email: mgerber@twilio.com
normative:
  RFC3986:
  RFC6749:
  RFC7591:
  RFC8414:
  RFC9110:
  RFC9111:
  RFC8259:
  RFC9700:
  I-D.ietf-oauth-client-id-metadata-document:
informative:
  RFC7592:
  RFC9449:
  I-D.ietf-oauth-v2-1:
  MCP-Auth:
    title: "Model Context Protocol: Authorization"
    target: https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization
    date: 2025-11-25
    author:
     - org: Model Context Protocol
  Watson-AS-Metadata:
    title: OAuth Authorization Server Metadata Client ID Parameter
    author:
     - fullname: Nicholas Watson
    target: https://github.com/njwatson32/as-metadata-client-id/blob/main/draft-watson-oauth-as-metadata-client-id.md
...

--- abstract

This document defines an OAuth endpoint for retrieving a public subset of a
client's effective configuration at an authorization server, including applied
defaults. It enables clients to check that their configuration is accepted by
an authorization server before an authorization attempt. It also enables
authorization servers to progressively roll out features to their client base.
While the endpoint chiefly serves Client ID Metadata Document clients,
it can also be used by clients registered through other means.

--- middle

# Introduction

OAuth Client ID Metadata Documents (CIMD)
{{I-D.ietf-oauth-client-id-metadata-document}} enable clients to publish metadata at a
URL used as their client identifier. The published document represents a client's
view of itself and its own capabilities. The published document does not tell a
client which values a given authorization server accepts or if the server applies
additional defaults. Incompatibility is only discovered when an authorization
attempt fails.

That failure is often invisible to the client. When the client identifier or
redirect URI is the problem, Section 4.1.2.1 of {{I-D.ietf-oauth-v2-1}} requires the AS not
to redirect back to the client. The user is left at an AS error page, and the
client never learns why the flow stopped.

This document defines an unauthenticated client metadata lookup operation. CIMD
clients are the principal motivation, but the operation also applies to clients
registered through other means, including statically registered public clients.

RFC 7592 Section 2.1 defines a similar Client Read Request at a Client
Configuration Endpoint {{RFC7592}}. That operation requires a registration
access token and can return credentials. The operation defined here is
separately discoverable, returns only public configuration, and does not require
the client to be created through DCR.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Client Identifier URL and Client ID Metadata Document have the meanings defined
in {{I-D.ietf-oauth-client-id-metadata-document}}.

Effective configuration is the client metadata the AS uses after resolving the
client's registration, applying defaults, and enforcing client configuration
policy.

# Use Cases

## Clients of Unfamiliar Authorization Servers

The Model Context Protocol (MCP) authorization specification {{MCP-Auth}} uses
CIMD as its preferred registration mechanism for clients with no prior
relationship to a server. An MCP client, such as a desktop agent or an IDE,
publishes one metadata document and presents it to whichever authorization
servers protect the MCP servers its users add. The client's developer cannot
test against each of those authorization servers in advance, and each
authorization server may apply its own policy to the same document.

With this endpoint, the client can look up whether its configuration is accepted
at an authorization server before sending the user to the authorization endpoint.
A rejected document becomes an error the client can show, log, or report, instead
of a runtime error presented to the user during the authorization flow.

## Progressive Feature Rollout

An AS can change a client's configuration as part of rolling out a new
requirement and expose the result through this endpoint, so clients can adapt
without first failing a user's request.

For example, an AS rolling out sender-constrained access tokens with DPoP
{{RFC9449}} can set the `dpop_bound_access_tokens` client metadata value,
defined in Section 5.2 of {{RFC9449}}, for one cohort of clients at a time.
When the value is `true`, the AS rejects token requests from the client that
lack a DPoP proof. No AS metadata parameter signals this requirement:
`dpop_signing_alg_values_supported` advertises support for DPoP, not whether it
is required. A client that finds `true` in its lookup response knows it must
use DPoP at this AS. Without the lookup, the first signal is a rejected token
request, after the user has already authenticated and consented.

Rollout through this endpoint suits fields whose values reflect AS policy, such
as requirements the AS enforces on the client. It is not a way for an AS to
change values that only describe the client, such as its redirect URIs or the
algorithms it signs with. `dpop_bound_access_tokens` is phrased as a client
declaration, but it obliges the AS to reject non-DPoP token requests, and
Section 5.2 of {{RFC9449}} anticipates AS-managed client configuration. An AS
that sets the value for a client reports it here. Whether AS policy can set a
given field is governed by that field's definition, not by this
specification. Clients learn of changes only by repeating the lookup; this
specification defines no change notification.

## Relationship to Client-Specific AS Metadata

{{Watson-AS-Metadata}} proposes a `client_id` parameter on AS metadata requests
so an AS can tailor its advertised capabilities to a client. The mechanisms
are complementary: AS metadata describes the AS using AS metadata fields;
this endpoint describes the client using client registration metadata fields.

For example, `dpop_signing_alg_values_supported` (Section 5.1 of {{RFC9449}})
is AS metadata, stating which algorithms the AS accepts for DPoP proofs.
`dpop_bound_access_tokens` (Section 5.2 of {{RFC9449}}) is client metadata,
stating whether the AS requires DPoP of one client. Client-specific AS metadata
can tailor the first, but has no field to carry the second. Deployments using
both mechanisms MUST keep the two views consistent. Neither mechanism requires
the other.

# Client Metadata Lookup Endpoint {#endpoint}

The client metadata lookup endpoint is used by a client to retrieve its
effective configuration at an authorization server. The client calls the
endpoint directly, typically before redirecting an end user to the
authorization endpoint.

The client obtains the URL of the endpoint from the
`client_metadata_lookup_endpoint` member of the authorization server's metadata
({{discovery}}).

The endpoint URL MUST use the `https` scheme and MUST NOT include a fragment
component or query component.

The authorization server MUST support the use of the HTTP GET method
({{Section 9.3.1 of RFC9110}}) at this endpoint.

The endpoint MUST accept requests without client authentication. A client
identifier is not proof of the requester's identity, and the response MUST NOT
depend on an ambient user session or cookies.

Because clients access this endpoint directly, including from browsers, the
authorization server SHOULD support Cross-Origin Resource Sharing (CORS) at
this endpoint, consistent with Section 2.6 of {{RFC9700}}.

# Endpoint Discovery {#discovery}

An AS supporting this specification MUST include the following member in its
AS metadata {{RFC8414}}:

client_metadata_lookup_endpoint:
: URL of the client metadata lookup endpoint, as described in {{endpoint}}.

This member is OPTIONAL for authorization servers generally. Its presence
advertises the operation defined here. Neither `registration_endpoint` nor
`client_id_metadata_document_supported` alone implies support.

# Lookup Request {#request}

The client sends an HTTP GET request to the advertised endpoint with exactly
one `client_id` query parameter. Its value is the client identifier defined in
RFC 6749 Section 2.2, encoded according to Appendix B of {{RFC6749}}. For a
CIMD client, the value is its Client Identifier URL:

~~~ http-message
GET /lookup?client_id=https%3A%2F%2Fapp.example%2Fclient HTTP/1.1
Host: as.example
Accept: application/json
~~~

The AS MUST resolve the client's configuration using the same registration
rules and policy it uses at other endpoints. For CIMD clients, CIMD validation,
retrieval, and caching requirements apply. Other clients use their existing
registration.

# Lookup Response

A successful response MUST use HTTP 200 and `application/json` {{RFC8259}}.
The body is an object containing `client_id` and a public subset of effective
client metadata as top-level members. Metadata names and types follow
{{RFC7591}} and applicable extensions. The returned `client_id` MUST match the
request using simple string comparison {{RFC3986}}.

For supported fields safe to disclose, the AS MUST return effective values,
including defaults it applies. This specification does not change those defaults.
Unsupported or non-public members MUST be omitted. An omitted member does not
mean an empty value or acceptance of a submitted value. Clients MUST NOT infer
an effective value by applying registration-time defaults to an omitted member.

The AS MUST NOT return secrets, private keys, tokens, user data, usage history,
or internal administrative information. In particular, `client_secret`,
`client_secret_expires_at`, and `registration_access_token` MUST NOT appear.
Other sensitive metadata MUST also be omitted.

Suppose the CIMD client in the request above publishes a minimal document at
its Client Identifier URL:

~~~ json
{
  "client_id": "https://app.example/client",
  "client_name": "Example App",
  "redirect_uris": ["https://app.example/callback"]
}
~~~

The lookup response fills in the values the AS applies to the omitted members:

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=300
Access-Control-Allow-Origin: *

{
  "client_id": "https://app.example/client",
  "client_name": "Example App",
  "redirect_uris": ["https://app.example/callback"],
  "grant_types": ["authorization_code"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none"
}
~~~

The `grant_types` and `response_types` values match the {{RFC7591}} defaults.
`token_endpoint_auth_method` does not: RFC 7591 defaults it to
`client_secret_basic`, a shared-secret method CIMD clients cannot use, so this
AS treats the client as public and applies `none`. A client that applied RFC
7591 defaults to its own document would get this wrong, which is why clients
MUST NOT infer effective values from omitted members.

Success describes current configuration, not endorsement, user consent, or a
promise of token issuance. The AS MUST independently validate subsequent
requests. Configuration can change between lookup and protocol use.

# Error Response {#errors}

Errors MUST use the JSON representation from RFC 6749 Section 5.2, with required
`error` and optional `error_description` and `error_uri` members, including that
section's syntax restrictions {{RFC6749}}. Responses MUST use HTTP 400 and
`application/json` unless another HTTP status is specified below.

The following error codes apply at this endpoint:

invalid_request:
: The request is malformed, including a missing or repeated `client_id`.

invalid_client:
: Configuration is unavailable because the client is unknown, disallowed, or
  cannot safely be acknowledged. This reuses the RFC 6749 error name for this
  endpoint; it does not indicate a failed authentication attempt.

invalid_client_metadata:
: Client metadata could not be retrieved or accepted, including an unreachable
  CIMD URL, malformed document, identifier mismatch, or unacceptable value.
  This extends the use of the error name defined by {{RFC7591}} to this endpoint.

Unknown, disallowed, and non-disclosable clients MUST receive indistinguishable
`invalid_client` responses.

A Client ID Metadata Document is public and its Client Identifier URL is
self-asserted, so explaining why the AS rejected the document discloses little
the requester could not already see. When the AS rejects a CIMD client's
document, it SHOULD return `invalid_client_metadata` with an `error_description`
identifying the failing member or check. For other clients, metadata
diagnostics MAY be returned only when safe for an unauthenticated requester;
otherwise the AS MUST return `invalid_client` without explanatory detail.

For example, when a CIMD client's document declares a shared-secret
authentication method, which {{I-D.ietf-oauth-client-id-metadata-document}}
does not permit:

~~~ http-message
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store
Access-Control-Allow-Origin: *

{
  "error": "invalid_client_metadata",
  "error_description":
    "token_endpoint_auth_method client_secret_basic not permitted"
}
~~~

# Caching

Following the CIMD caching model, successful lookup responses MAY be cached.
The AS SHOULD provide HTTP cache headers {{RFC9111}} reflecting source
freshness and configuration policy; it MUST NOT blindly copy a source CIMD's
cache headers.

Clients MUST NOT cache errors or invalid or malformed responses. The AS MUST
send `Cache-Control: no-store` on errors.

# Security Considerations {#security}

TODO!

# Privacy Considerations {#privacy}

TODO!

# IANA Considerations

This document requests the following registration in the OAuth Authorization
Server Metadata registry established by {{RFC8414}}:

Metadata Name:
: `client_metadata_lookup_endpoint`

Metadata Description:
: URL of the client metadata lookup endpoint.

Change Controller:
: IETF

Specification Document:
: {{discovery}} of this document.

--- back

# Acknowledgments
{:numbered="false"}

TODO
