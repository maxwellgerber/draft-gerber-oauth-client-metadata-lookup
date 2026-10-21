---
title: OAuth Client Configuration Endpoint
abbrev: Client Configuration
category: std
docname: draft-gerber-oauth-client-registration-endpoint-latest
submissiontype: IETF
date: 2026-10-21
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
author:
 - fullname: Maxwell Gerber
   email: maxwell.gerber42@gmail.com
normative:
  RFC2119:
  RFC8174:
  RFC3986:
  RFC6749:
  RFC7591:
  RFC8414:
  RFC9111:
  RFC8259:
  RFC9700:
  I-D.ietf-oauth-client-id-metadata-document:
informative:
  RFC7592:
  OpenID.Registration:
    title: OpenID Connect Dynamic Client Registration 1.0 incorporating errata set 2
    target: https://openid.net/specs/openid-connect-registration-1_0.html
    date: 2023-12-15
    author:
     - fullname: Nat Sakimura
     - fullname: John Bradley
     - fullname: Michael B. Jones
  Watson-AS-Metadata:
    title: OAuth Authorization Server Metadata Client ID Parameter
    author:
     - fullname: Nicholas Watson
    target: https://github.com/njwatson32/as-metadata-client-id/blob/main/draft-watson-oauth-as-metadata-client-id.md
...

--- abstract

This document defines an OAuth endpoint for retrieving a public subset of a
client's effective configuration at an authorization server, including applied
defaults. It enables clients to check configuration before an authorization
attempt and supports progressive feature rollout. The endpoint applies to
Client ID Metadata Document clients and clients registered through other means.

--- middle

# Introduction

OAuth Client ID Metadata Documents (CIMD)
{{I-D.ietf-oauth-client-id-metadata-document}} let clients publish metadata at a
URL used as their client identifier. The published document does not tell a
client which values an authorization server (AS) accepts or which defaults it
applies. Discovering a mismatch through a user's authorization attempt can leave
that user at an error page.

This document defines an unauthenticated client configuration lookup operation.
CIMD is the principal motivation, but the operation also applies to clients
registered through Dynamic Client Registration (DCR) {{RFC7591}} or other means,
including statically registered public clients.

RFC 7592 Section 2.1 defines a similar Client Read Request at a Client
Configuration Endpoint {{RFC7592}}. That operation requires a registration
access token and can return credentials. The operation defined here is
separately discoverable, returns only public configuration, and does not require
prior DCR. It does not change RFC 7592 authentication or management operations.
An AS MAY host both operations at the same URL if their requirements remain
distinct.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Client Identifier URL and Client ID Metadata Document have the meanings defined
in {{I-D.ietf-oauth-client-id-metadata-document}}.

Effective configuration is the client metadata the AS uses after resolving the
client's registration, applying defaults, and enforcing client configuration
policy. It excludes user grants and transaction-specific authorization decisions.

# Use Cases

## Configuration Feedback

A client retrieves its configuration before involving an end user. A CIMD
client can compare the returned metadata with its published document; other
clients can inspect the configuration established through DCR or provisioning.

## Progressive Feature Rollout

An AS can enable a feature for selected clients and expose the resulting
configuration through this endpoint. For example,
`token_endpoint_auth_signing_alg`, defined by {{OpenID.Registration}}, reports
the configured signing algorithm for client authentication JWTs. Clients known
to support a new algorithm can receive that configuration while others retain
the previous value.

This endpoint reports configuration; it does not establish client capabilities.
Selection MUST respect the semantics of each metadata field and the client's
established capabilities. Clients must retrieve configuration to observe changes.

## Relationship to Client-Specific AS Metadata

{{Watson-AS-Metadata}} proposes a `client_id` parameter on AS metadata requests
so an AS can tailor its advertised capabilities to a client. The mechanisms
are complementary: AS metadata describes the AS using AS metadata fields;
this endpoint describes the client using client registration metadata fields.

For example, `token_endpoint_auth_signing_alg_values_supported` advertises AS
capabilities, while `token_endpoint_auth_signing_alg` reports the client's
configured algorithm. Deployments using both mechanisms MUST keep the two
views consistent. Neither mechanism requires the other.

# Endpoint Discovery {#discovery}

An AS supporting this specification MUST include the following member in its
AS metadata {{RFC8414}}:

client_configuration_endpoint:
: HTTPS URL of the public client configuration endpoint. The URL MUST NOT
  contain a fragment or a `client_id` query parameter.

This member is OPTIONAL for authorization servers generally. Its presence
advertises the operation defined here. Neither `registration_endpoint` nor
`client_id_metadata_document_supported` alone implies support.

# Configuration Request

The client sends an HTTP GET request to the advertised endpoint with exactly
one `client_id` query parameter. Its value is the client identifier defined in
RFC 6749 Section 2.2, encoded according to Appendix B of {{RFC6749}}. The
parameter is appended to any existing query component.

~~~ http-message
GET /client-configuration?client_id=native-app-42 HTTP/1.1
Host: as.example
Accept: application/json
~~~

The endpoint MUST support requests without authentication. A client identifier
is not proof of the requester's identity. The response MUST NOT depend on an
ambient user session or cookies.

The AS MUST resolve the client's configuration using the same registration
rules and policy it uses at other endpoints. For CIMD clients, CIMD validation,
retrieval, and caching requirements apply. Other clients use their existing
registration. An AS MUST NOT infer that an identifier is a CIMD URL solely from
its `https` scheme or fetch it in place of an existing registration.

Lookup MUST NOT create a durable registration, issue credentials, record
consent, or grant access. Normal metadata caching and abuse controls are
permitted; lookup MUST NOT force a refresh that bypasses CIMD caching policy.

# Configuration Response

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
Other sensitive metadata MUST also be omitted; omission does not require
rejecting the entire lookup. Every returned value MUST accurately reflect the
client's effective configuration.

For the statically registered public client in the request above:

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=300
Access-Control-Allow-Origin: *

{
  "client_id": "native-app-42",
  "client_name": "Example Native App",
  "redirect_uris": ["https://app.example/callback"],
  "grant_types": ["authorization_code"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none"
}
~~~

For CIMD clients, the request uses the Client Identifier URL as the encoded
`client_id`, and the response contains that exact URL as its `client_id`.

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
`invalid_client` responses. Metadata diagnostics MAY be returned only when safe
for an unauthenticated requester; otherwise the AS MUST return `invalid_client`
without explanatory detail. Errors MUST NOT expose internal network details or
upstream response bodies.

For example, when a publicly hosted CIMD cannot be retrieved and the AS can
safely disclose that fact:

~~~ http-message
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store
Access-Control-Allow-Origin: *

{
  "error": "invalid_client_metadata",
  "error_description": "Metadata document could not be retrieved."
}
~~~

Rate limits and temporary AS failures MAY use HTTP 429 and 503 respectively.
Clients MUST handle HTTP errors without JSON bodies. An AS failure does not
necessarily indicate a defect in the client's metadata.

# Caching

Following the CIMD caching model, successful configuration responses MAY be
cached. The AS SHOULD provide HTTP cache headers {{RFC9111}} reflecting source
freshness and configuration policy; it MUST NOT blindly copy a source CIMD's
cache headers. Clients SHOULD respect these headers but MAY apply their own
bounds on cache lifetime. `max-age=300` above is illustrative, not a default.

Clients MUST NOT cache errors or invalid or malformed responses. The AS MUST
send `Cache-Control: no-store` on errors. Cache keys MUST distinguish the AS
endpoint and complete `client_id` query value. Source CIMD caching and caching
of this endpoint's responses are separate operations.

# Cross-Origin Resource Sharing

Consistent with the guidance for directly accessed OAuth endpoints in
RFC 9700 Section 2.6 {{RFC9700}}, this endpoint SHOULD support Cross-Origin
Resource Sharing (CORS) for success and error responses.

For this public operation, the AS MAY send `Access-Control-Allow-Origin: *`.
Browser clients SHOULD omit credentials. The AS MUST NOT enable credentialed
cross-origin access or use ambient credentials to return richer configuration.
If an AS dynamically selects an allowed origin, it MUST send `Vary: Origin`.
CORS is not authentication and does not constrain non-browser callers.

# Security and Privacy Considerations

The security considerations of {{RFC9700}} apply. For CIMD clients, the CIMD
security and privacy requirements also apply, including restrictions on URL
fetching, redirects, special-use destinations, and response sizes.

This endpoint adds an unauthenticated path to metadata retrieval. The AS SHOULD
bound fetch time, document size, dependent fetches, and request volume to limit
SSRF and denial-of-service risks. Attackers can submit unique URLs to avoid
cache hits and induce traffic to third-party hosts.

Public source metadata does not make an AS's private overrides or association
with a client public. The AS MUST assess returned fields for disclosure,
including ordinary metadata such as scopes and redirect URIs. Responses MUST
NOT reveal user grants, prior client use, or first- or last-use timestamps.
Acceptance results can reveal policy or rollout membership; they are not access
credentials. Clients MUST NOT downgrade security merely because lookup fails.

CIMD fetches can reveal the AS's network identity and fetch timing to the
metadata host. Query strings can expose client identifiers in logs. Operators
SHOULD minimize unnecessary fetches and retention of identifying log data.

# IANA Considerations

This document requests the following registration in the OAuth Authorization
Server Metadata registry established by {{RFC8414}}:

Metadata Name:
: `client_configuration_endpoint`

Metadata Description:
: URL of the public client configuration endpoint.

Change Controller:
: IETF

Specification Document:
: {{discovery}} of this document.

This document also requests that IANA add "client configuration endpoint" as
a usage location for `invalid_request`, `invalid_client`, and
`invalid_client_metadata` in the OAuth Extensions Error Registry, with this
document as the related protocol extension and {{errors}} as the reference.
Existing usages and references are retained. The registration of `client_id`
in the OAuth Parameters registry is extended to include client configuration
requests, with the Configuration Request section as an additional reference.

--- back

# Acknowledgments
{:numbered="false"}

TODO
