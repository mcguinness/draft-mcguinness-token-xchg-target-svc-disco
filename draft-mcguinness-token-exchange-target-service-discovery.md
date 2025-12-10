---
title: "OAuth 2.0 Token Exchange Target Service Discovery"
abbrev: Token Exchange Target Service Discovery"
docName: "draft-mcguinness-token-exchange-target-service-discovery-latest"
category:  "std"
# workgroup: "Web Authorization Protocol"
# area: "Security"
ipr: "trust200902"
keyword:
  - "OAuth 2.0"
  - "Resource Indicators"
  - "Authorization Server"
  - "Token Exchange"
  - "Identity Chaining"
venue:
#  group: "Web Authorization Protocol"
#  type: "Working Group"
#  mail: "oauth@ietf.org"
#  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "mcguinness/draft-mcguinness-token-exchange-target-service-discovery"
  latest: "https://mcguinness.github.io/draft-mcguinness-token-exchange-target-service-discovery/draft-mcguinness-token-exchange-target-service-discovery.html"

author:
 -
    fullname: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com
 -
    fullname: Aaron Parecki
    organization: Okta
    email: aaron@parecki.com
    uri: https://www.okta.com

normative:
  RFC6749:
  RFC8707:
  RFC9728:
  RFC8414:
  RFC9207:
  RFC3986:
  RFC7519:

informative:

--- abstract

This specification defines a discovery endpoint that enables OAuth 2.0 clients to determine which audiences, resources, and scopes for a target service they are allowed to request with OAuth 2.0 Token Exchange (RFC 8693). The endpoint accepts any subject token type registered in the Token Exchange Token Type registry and returns values that are valid inputs to subsequent Token Exchange requests, supporting advanced use cases such as identity chaining and cross-domain delegation.

--- middle

# Introduction

OAuth 2.0 Token Exchange (RFC 8693) enables a client to exchange one security token for another to obtain a token targeted at a different audience, resource, or security domain. However, a client must already know which values it is permitted to request. Today, this knowledge is typically provided through static configuration, proprietary APIs, or informal documentation, leading to brittle integrations and unnecessary Token Exchange failures.

This specification defines the **OAuth 2.0 Token Exchange Target Service Discovery Endpoint**, a standardized mechanism that allows clients to dynamically discover the set of permitted Token Exchange targets (audiences, resources, and scopes) for a given subject token. The authorization server evaluates both the subject token and the client’s permissions and returns only the values the client is authorized to request.

This extension is especially valuable in **identity chaining** and **cross-domain authorization** scenarios, such as:

* Multi-tier microservices architectures where tokens must be progressively exchanged with narrower permissions.
* Cross-organizational and federated systems where downstream services exist in separate administrative domains.
* Cross-cloud or hybrid deployments requiring strict control over which domains may issue or accept exchanged tokens.
* Delegated flows where derived tokens represent constrained or transformed identities across trust boundaries.

Benefits include:

* **Eliminates static configuration** — clients no longer rely on manually configured audience or resource lists.
* **Reduces Token Exchange failures** — the client learns valid targets before attempting a Token Exchange request.
* **Supports identity chaining** — the AS can provide the next allowable “hop” in a derived-identity chain.
* **Enhances cross-domain trust** — prevents clients from requesting tokens for unauthorized or untrusted domains.
* **Improves interoperability** — standardizes previously proprietary discovery mechanisms.
* **Improves developer and operator experience** — tools can automatically enumerate valid downstream targets.
* **Supports dynamic authorization** — results reflect real-time policy evaluation, client permissions, and subject token context.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Token Exchange Target Service Discovery Endpoint

## Endpoint Definition

A new Authorization Server metadata parameter is defined:

```
token_exchange_target_service_discovery_endpoint
```

The value MUST be the absolute URL of the Target Service Discovery Endpoint.
Clients MUST obtain this value from Authorization Server Metadata (RFC 8414).
No specific path is mandated by this specification.

The endpoint:

* MUST accept HTTP POST
* MUST use `application/x-www-form-urlencoded`
* MAY require client authentication
* MAY apply authorization policy

## Request Parameters

This specification defines only the parameters required for Target Service Discovery:

| Parameter | Description | Required | Registration |
|----------|-------------|----------|--------------|
| `subject_token` | The subject token used as input to discovery. | Yes | Token Exchange Subject Token Registry |
| `subject_token_type` | The type of the subject token. | Yes | Token Exchange Token Type Registry |

Client authentication is RECOMMENDED but NOT REQUIRED.  Client Authentication parameters are defined in their respective OAuth profiles.


## Successful Response

The response is a JSON array of permitted Token Exchange targets.

### Response Parameter Table

| Field | Description | Required | Registration |
|-------|-------------|----------|--------------|
| `audience` | A permitted audience for Token Exchange. | Yes | RFC 8693 |
| `resource` | A permitted resource indicator (string or array). | No | RFC 8707 |
| `scope` | Permitted OAuth scopes. | No | RFC 6749 |

# Cross-Domain Identity Chaining Example

This example demonstrates the complete cross-domain identity-chaining workflow:

1. **Discover permitted audiences/resources/scopes** for a given *subject access token*.
2. **Discover downstream token types** for the target audience using OAuth Authorization Server Metadata.
3. **Perform Token Exchange** using a *requested token type* returned by discovery.

## Step 1 — Discover Target Audiences, Resources, and Scopes

The client begins with a subject **access token** issued by Domain A and calls the Target Service Discovery Endpoint.

### Request

```http
POST https://as.domainA.example/target-discovery
Content-Type: application/x-www-form-urlencoded

client_id=client-A
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=eyJhbGciOi...
&subject_token=SlAV32hkKG...ACCESSTOKEN...
&subject_token_type=urn:ietf:params:oauth:token-type:access_token
```

### Response

```json
[
  {
    "audience": "https://api.domainB.example",
    "resource": ["orders", "inventory"],
    "scope": "orders.read inventory.read",
    "requested_token_type": "urn:ietf:params:oauth:token-type:jwt"
  }
]
```

From this response the client learns:

* The permitted downstream audience: `https://api.domainB.example`
* Permitted scopes/resources
* Recommended downstream token type: `urn:ietf:params:oauth:token-type:jwt`

## Step 2 — Discover Allowed Token Types for the Audience’s Authorization Server

The client queries the Authorization Server Metadata of Domain B to ensure the requested token type is supported.

### Domain B AS Metadata

```
GET https://as.domainB.example/.well-known/oauth-authorization-server
```

### Example Metadata

```json
{
  "issuer": "https://as.domainB.example",
  "identity_chaining_requested_token_types_supported": [
    "urn:ietf:params:oauth:token-type:jwt",
    "urn:ietf:params:oauth:token-type:access_token"
  ]
}
```

This confirms that Domain B **supports JWT as a downstream token type**.

## Step 3 — Perform Token Exchange Using the Discovered Token Type

The client now performs Token Exchange with Domain A’s token endpoint, requesting a JWT for Domain B.

### Token Exchange Request

```http
POST https://as.domainA.example/token
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token=SlAV32hkKG...ACCESSTOKEN...
&subject_token_type=urn:ietf:params:oauth:token-type:access_token
&requested_token_type=urn:ietf:params:oauth:token-type:jwt
&audience=https://api.domainB.example
&scope=orders.read inventory.read
```

### Token Exchange Response

```json
{
  "access_token": "eyJraWQiOi...DOMAINB.JWT...",
  "issued_token_type": "urn:ietf:params:oauth:token-type:jwt",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "orders.read inventory.read"
}
```

The client now holds a **Domain B–scoped JWT**, derived from Domain A’s access token.

# Error Responses

Any registered OAuth 2.0 error may be returned.  Some examples include:

| Error | When Returned | Registration |
|-------|---------------|--------------|
| `invalid_request` | Missing or malformed parameters. | RFC 6749 |
| `invalid_client` | Client authentication failure. | RFC 6749 |
| `unauthorized_client` | Client not permitted to perform discovery. | RFC 6749 |
| `invalid_grant` | Subject token invalid, expired, or failed validation. | RFC 6749 / RFC 8693 |
| `unsupported_grant_type` | Server interprets request as unsupported structure. | RFC 6749 |
| `invalid_scope` | Client requests unauthorized scopes. | RFC 6749 |

# Metadata Discovery

Metadata field:

| Name | Type | Description |
|------|------|-------------|
| `token_exchange_target_service_discovery_endpoint` | URL | Token Exchange Target Service Discovery Endpoint |

# Security Considerations

Token validation, policy enforcement, enumeration protections, and safe error handling.

# Privacy Considerations

Discovery results may reveal authorization relationships; servers SHOULD minimize exposure.

# IANA Considerations

Register `token_exchange_target_service_discovery_endpoint`.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.

