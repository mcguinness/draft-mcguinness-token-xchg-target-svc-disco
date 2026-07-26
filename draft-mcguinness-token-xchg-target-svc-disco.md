---
title: "OAuth 2.0 Token Exchange Target Service Discovery"
abbrev: "Token Exchange Target Service Discovery"
docName: "draft-mcguinness-token-xchg-target-svc-disco-latest"
category:  "std"
workgroup: "Web Authorization Protocol"
area: "Security"
ipr: "trust200902"
date: 2026-07-25
keyword:
  - "OAuth 2.0"
  - "Token Exchange"
  - "Identity Chaining"
  - "Resource Indicators"
  - "Cross-Domain"
  - "Authorization"
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "mcguinness/draft-mcguinness-token-xchg-target-svc-disco"
  latest: "https://mcguinness.github.io/draft-mcguinness-token-xchg-target-svc-disco/draft-mcguinness-token-xchg-target-svc-disco.html"

author:
 -
    fullname: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com
 -
    fullname: Aaron Parecki
    organization: Okta
    email: aaron@parecki.com
    uri: https://aaronparecki.com

normative:
  RFC6749:
  RFC7523:
  RFC8259:
  RFC8707:
  RFC8414:
  RFC8693:
  RFC9111:
  RFC9396:

informative:
  RFC6585:
  RFC6755:
  RFC9110:
  I-D.zehavi-oauth-rar-metadata:
  I-D.ietf-oauth-identity-chaining:
  I-D.oauth-identity-assertion-authz-grant:
    title: OAuth 2.0 Identity Assertion JWT Authorization Grant
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/
    author:
      - ins: A. Parecki
        name: Aaron Parecki
      - ins: K. McGuinness
        name: Karl McGuinness
      - ins: B. Campbell
        name: Brian Campbell
    date: 2026

--- abstract

This specification defines a method for OAuth 2.0 clients to discover the set of available Token Exchange Targets (such as audiences, resources, scopes, token types, and authorization details types) for a given subject token when performing OAuth 2.0 Token Exchange. The discovery endpoint accepts a subject token of any type the authorization server supports, identified by a token type URI, and returns values that are valid inputs to subsequent Token Exchange requests, supporting advanced use cases such as identity chaining and cross-domain delegation.

--- middle

# Introduction

OAuth 2.0 Token Exchange {{RFC8693}} enables a client to present a token issued in one security domain to an authorization server and securely trade it for a new token issued for another domain. The exchanged token is minted with the Target Service's token requirements, including its own audience, resource indicators, and scopes, so it becomes valid and enforceable for the desired downstream service. This enables controlled cross-domain access, removes the confused-deputy problem by ensuring tokens are explicitly targeted to the correct service, and avoids requiring direct trust between the original token issuer and the Target Service.

Authorization Servers may be capable of issuing tokens to multiple services for a given subject token and client, but the client must already know which values it may request. Today, this knowledge is typically provided through static configuration, proprietary APIs, or informal documentation, leading to brittle integrations and unnecessary Token Exchange failures, particularly when subjects are authorized to access only a subset of available services.

This specification defines the OAuth 2.0 Token Exchange Target Service Discovery Endpoint, a standardized mechanism that enables clients to dynamically discover the set of available Token Exchange Targets (such as audiences, resources, scopes, token types, and authorization details types) for a given subject token. The authorization server evaluates both the subject token and the client's permissions and returns only the values the client is authorized to request.

This extension is especially valuable in scenarios requiring identity chaining and cross-domain authorization:

* Progressive token exchange across service boundaries, such as multi-tier microservices architectures and delegated flows where tokens are exchanged with narrower or transformed permissions at each hop. The set of downstream services a client may target, and their required permissions, varies with the subject token's context and the client's authorization, so static configuration cannot anticipate these per-subject and per-client variations and clients attempt exchanges the subject is not authorized to perform.

* Cross-boundary deployments where downstream services exist in separate administrative, organizational, or cloud domains with independently managed authorization policies. Unlike progressive token exchange, which transforms permissions within a single domain, this scenario spans distinct trust boundaries whose policies and authorized targets change dynamically, so a client must discover them at runtime rather than rely on fixed configuration.

* Single Sign-On (SSO) to API flows, such as those enabled by the Identity Assertion Authorization Grant (ID-JAG) {{I-D.oauth-identity-assertion-authz-grant}}, where a client needs to seamlessly connect to cross-domain resources and act on behalf of the user to access APIs.

* Multi-tenant Target Services that share the same authorization server and/or resource across tenants, where the tenant is identified by a claim in the issued token. In these scenarios, the resource indicator alone cannot distinguish the tenants, so discovery enables a client to learn the distinct Target Services available per tenant and to present a tenant selection to the end user.

This specification provides the following benefits:

* **Dynamic Discovery**: Eliminates static configuration requirements by enabling clients to discover available Token Exchange Targets at runtime, reducing integration failures and improving developer experience.

* **Standardization**: Provides a standardized discovery mechanism, replacing proprietary APIs and improving interoperability across OAuth 2.0 implementations.

* **Real-Time Authorization**: Returns Token Exchange Targets based on real-time policy evaluation, client permissions, and subject token context, ensuring accurate and up-to-date authorization information. This enables per-subject and per-client results, which is essential when the set of available targets varies by user or client.

# Conventions and Definitions {#terminology}

{::boilerplate bcp14-tagged}

This document uses the following terms:

Target Service
: A downstream service that a client accesses using a token obtained through OAuth 2.0 Token Exchange {{RFC8693}}. A Target Service may be multi-tenant (see {{multi-tenant-target-services}}).

Token Exchange Target
: A combination of Token Exchange request parameters (an `audience` and, where applicable, `resource`, `scope`, requested token type(s), and available authorization details types) that the authorization server has authorized for a given subject token and requesting client. A Token Exchange Target is returned by the discovery endpoint as an element of the `supported_targets` array; it identifies how a client may obtain a token for a Target Service and does not assert that the service is reachable or operational.

# Token Exchange Target Service Discovery Endpoint

This specification defines a new endpoint for OAuth 2.0 authorization servers that enables clients to discover the set of available Token Exchange Targets (such as audiences, resources, scopes, token types, and authorization details types) for a given subject token when performing OAuth 2.0 Token Exchange {{RFC8693}}.

The endpoint is identified by the authorization server metadata parameter `token_exchange_target_service_discovery_endpoint`, as defined in {{authorization-server-metadata}}.

## Authorization Server Metadata {#authorization-server-metadata}

This specification defines the following authorization server metadata {{RFC8414}} parameter to enable clients to discover the token exchange target service discovery endpoint:

token_exchange_target_service_discovery_endpoint
: URL of the token exchange target service discovery endpoint. This URL MUST use the `https` scheme. The authorization server SHOULD publish this metadata value.

The following is a non-normative example of the metadata:

    {
      "issuer": "https://as.example.com",
      "token_exchange_target_service_discovery_endpoint": "https://as.example.com/target-discovery"
    }

## Endpoint Request

The client makes a request to the token exchange target service discovery endpoint by sending an HTTP POST request to the endpoint URL. The client MUST use TLS as specified in {{Section 1.6 of RFC6749}}.

The endpoint URL MUST be obtained from the authorization server's metadata document {{RFC8414}} using the `token_exchange_target_service_discovery_endpoint` parameter. The endpoint URL is an absolute URL.

The client sends the parameters using the `application/x-www-form-urlencoded` format per Appendix B of {{RFC6749}}. Character encoding MUST be UTF-8 as specified in Appendix B of {{RFC6749}}. If a parameter is included more than once in the request, the authorization server MUST return an error response with the error code `invalid_request` as described in {{error-response}}.

### Request Parameters

The following parameters are used in the request:

subject_token
: REQUIRED. The subject token used as input to discovery. The token type is indicated by the `subject_token_type` parameter.

subject_token_type
: REQUIRED. A string value containing a URI, as described in {{Section 3 of RFC8693}}, that indicates the type of the `subject_token` parameter. This identifier MUST be a valid URI, and SHOULD be registered in the "OAuth URI Registry" as established by {{RFC6755}}.

The client MAY include additional parameters as defined by extensions, and the authorization server MUST ignore parameters it does not understand. The parameters of the client authentication method in use (for example, `client_id`, `client_secret`, or `client_assertion` and `client_assertion_type`) are part of that method and are not treated as unknown parameters.

The authorization server identifies the requesting client in order to evaluate its permissions ({{authorization-policy-enforcement}}). Client authentication MAY be required by the authorization server. The means of client authentication are defined by the authorization server and MAY include any method supported by the authorization server, including those defined in {{Section 2.3 of RFC6749}} and extensions. If client authentication is required by the authorization server but not provided in the request, the authorization server MUST return an error response with the error code `invalid_client` as described in {{error-response}}. A public client that does not authenticate MAY identify itself with the `client_id` parameter; the authorization server determines the trust it places in such an unauthenticated identifier, and MAY base its results on the subject token alone when no client identity is established.

### Subject Token Processing {#subject-token-processing}

The authorization server MUST process the `subject_token` parameter according to the following rules:

1. The authorization server MUST validate that the `subject_token` and `subject_token_type` parameters are not empty strings. If either parameter is an empty string, the authorization server MUST return an error response with the error code `invalid_request` as described in {{error-response}}.

2. The authorization server MUST validate that the `subject_token_type` parameter is a valid URI. If the format is invalid (e.g., not a valid absolute URI or URN), the authorization server MUST return an error response with the error code `invalid_request` as described in {{error-response}}.

3. The authorization server MUST determine whether it supports the indicated token type. This determination is an implementation decision and MAY be based on the authorization server's capabilities, the `subject_token` value, the authenticated client, or any combination thereof. If the `subject_token_type` is not supported by the authorization server for the given context, the authorization server MUST return an error response with the error code `unsupported_token_type` as described in {{error-response}}.

4. The authorization server MUST validate the `subject_token` according to the rules for the indicated token type. The validation process MUST verify:
   * The token is properly formatted for the indicated token type
   * The token's signature (if applicable) is valid and can be verified using the appropriate cryptographic keys
   * The token was issued by a trusted issuer
   * The token has not expired
   * The token has not been revoked, where revocation status is applicable and available for the token type; a token known to be revoked MUST be rejected
   * The token is associated with the authenticated client, if client authentication is required

5. If the `subject_token` is invalid for any reason (e.g., malformed, expired, revoked, or does not match the `subject_token_type`), the authorization server MUST return an error response with the error code `invalid_request` as described in {{error-response}}.

6. The authorization server MUST evaluate the `subject_token` in conjunction with the requesting client's permissions (the authenticated client, when client authentication is used) to determine which Token Exchange Targets are available for discovery. The specific authorization policy evaluation mechanism is implementation-specific and MAY be based on scopes, claims, resource-based access control, or other authorization models.

### Request Example

The following is an example of a discovery request:

    POST /target-discovery HTTP/1.1
    Host: as.example.com
    Content-Type: application/x-www-form-urlencoded

    client_id=client-A
    &client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
    &client_assertion=eyJhbGciOi...
    &subject_token=SlAV32hkKG...ACCESSTOKEN...
    &subject_token_type=urn:ietf:params:oauth:token-type:access_token


## Endpoint Response

The authorization server validates the request and returns a response with the discovery results. The `Content-Type` header of the response MUST be set to `application/json`.

Because the response contains sensitive, per-subject and per-client authorization information, the authorization server SHOULD set `Cache-Control: no-store` by default. A deployment that accepts the associated risk MAY instead permit bounded private caching; in that case the cache directives MUST mark the response as private (for example, `Cache-Control: private`) and shared caches MUST NOT store it. The authorization server MAY include cache validators (such as `ETag` or `Last-Modified` headers) to enable conditional requests by the client, as specified in {{RFC9111}}.

### Successful Response

If the request is valid and authorized, the authorization server returns a JSON {{RFC8259}} object containing a `supported_targets` property: an array of the Token Exchange Targets (see {{terminology}}) available to the client. To perform a token exchange, the client selects one target and uses its values as the corresponding request parameters.

Empty and null values are not valid property values. The authorization server MUST NOT include an optional property whose value would be `null`, an empty string, an empty array, or an array containing an empty string; it MUST omit the property instead. A REQUIRED property MUST have a non-empty value.

Each target object contains the following properties:

audience
: REQUIRED. The value the client uses, verbatim, as the `audience` parameter in a subsequent token exchange request, identifying the Target Service as defined in {{Section 2.1 of RFC8693}}. Its syntax is authorization-server-defined unless constrained by a profile of this specification; the client MUST treat it as opaque and MUST NOT assume it is a URI. If the value is a URI that does not itself provide a usable location (for example, a URN), the authorization server SHOULD also return a `resource` property that provides one. A multi-tenant Target Service returns a distinct `audience` value per tenant; see {{multi-tenant-target-services}}.

tenant
: OPTIONAL. A machine-readable identifier for the tenant of a multi-tenant Target Service. The issued token represents this tenant using the claim defined by the token format and Target Service (for example, the `aud_tenant` claim when the Identity Assertion Authorization Grant {{I-D.oauth-identity-assertion-authz-grant}} is used); the specific claim name and encoding are outside the scope of this specification. This property is included only when the Target Service is multi-tenant and the authorization server knows the tenant identifier. It is descriptive: it lets the client correlate a Target Service with the tenant context of the resulting issued token, and is not used as a selector in the token exchange request (the `audience` value selects the tenant).

resource
: OPTIONAL. A single resource indicator value or an array of resource indicator values, as defined in {{Section 2 of RFC8707}}, available for this target. Each value MUST be an absolute URI and MUST NOT include a fragment component, per {{Section 2 of RFC8707}}. If present as an array, the array entries correspond to repeated `resource` parameters in the subsequent token exchange request.

scope
: OPTIONAL. A string value containing a space-delimited list of OAuth 2.0 scope values, as defined in {{Section 3.3 of RFC6749}}, that are available for this target. Each scope value MUST conform to the scope syntax defined in {{Section 3.3 of RFC6749}}. If present, the value MUST contain at least one scope value. The authorization server determines which scopes to return based on its authorization policy evaluation, which is implementation-specific. The scopes returned SHOULD be those that would be authorized in a subsequent token exchange request per {{Section 2.1 of RFC8693}}.

authorization_details_types
: OPTIONAL. An array of strings, each a Rich Authorization Requests (RAR) {{RFC9396}} `authorization_details` type identifier that is eligible for use with this target in a subsequent token exchange. The `authorization_details` request parameter {{RFC9396}} may be used in a token exchange request in addition to, or instead of, `scope`. Each identifier corresponds to an authorization details type supported by the authorization server; this specification does not define those types or their schemas. The schema and examples for a type are obtained from the authorization server's Authorization Details Types Metadata endpoint {{I-D.zehavi-oauth-rar-metadata}}, if published (see {{rich-authorization-requests}}). Listing a type indicates only that the type is available for this target; it does not authorize every schema-valid `authorization_details` object of that type, nor any particular combination of objects. The authorization server evaluates the specific `authorization_details` content (which may include fields such as actions, locations, accounts, or amounts) when the token exchange is performed. The fields permitted or required in an `authorization_details` object are governed by the type definition and its schema per {{RFC9396}}; the target's `resource` and `audience` do not remove or substitute for any field the type requires (for example, `locations`). If omitted, this specification makes no assertion about which authorization details types are available for the target.

supported_token_types
: OPTIONAL. An array of strings indicating the token types that may be requested for this target in a subsequent token exchange operation. Each string MUST be a valid absolute URI. A token type identifier MAY be any URI, as permitted by {{Section 3 of RFC8693}}, when that URI identifies the requested token type or token usage profile understood by the authorization server and client. For example, `urn:ietf:params:oauth:grant-type:jwt-bearer` identifies a JWT intended to be presented as a JWT bearer authorization grant as defined by {{RFC7523}}; this specification intentionally permits the RFC 7523 grant-type URI to be used in this role, which the generic `urn:ietf:params:oauth:token-type:jwt` identifier does not convey. If omitted, the client may use any token type supported by the authorization server.

display_name
: OPTIONAL. A human-readable name for the Target Service, suitable for display to an end user (for example, in a service picker). This value is intended for presentation only and MUST NOT be used as a token exchange parameter.

client_id
: OPTIONAL. The OAuth 2.0 client identifier that the client uses with the Target Service when presenting the issued token (see {{multi-tenant-target-services}}). This property supports multi-tenant Target Services that require a distinct client registration for each tenant. The client is expected to possess credentials for the indicated client identifier where client authentication applies. If omitted, the client uses a client identity it determines is appropriate by other means.

Extensions to this specification MAY define additional properties for the response object or for target objects. Clients MUST ignore any properties they do not understand.

A target is identified by its `audience` together with its `resource` set (the complete set of `resource` values, or their absence). This key MUST be unique within the `supported_targets` array: no two targets may share both the same `audience` and the same `resource` set (when comparing sets, element order does not matter, but the membership must match). Multiple targets with the same `audience` MAY therefore be returned only when they differ in their `resource` set. Because each key appears at most once, a target's `scope` is the aggregate set of scopes authorized for that key, rather than one of several alternative scope bundles. A multi-tenant Target Service is represented as one target per tenant, each with a distinct `audience` (see {{multi-tenant-target-services}}), so the `audience` distinguishes the tenants.

If no Token Exchange Targets are available for the given subject token and client, the authorization server returns a JSON object with an empty `supported_targets` array: `{"supported_targets": []}`.

Discovery results reflect authorization at the time of the request and are not a guarantee. The authorization server re-evaluates authorization when the subsequent token exchange is performed, and that exchange might still fail (for example, if policy changed, the subject token expired, or the target became unavailable in the interim). A client MUST handle errors from the token exchange request {{RFC8693}} rather than assuming a discovered target will succeed.

### Response Example

The following is an example of a successful discovery response:

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "supported_targets": [
        {
          "audience": "https://api.example.com",
          "resource": [
            "https://api.example.com/orders",
            "https://api.example.com/inventory"
          ],
          "scope": "orders.read inventory.read",
          "authorization_details_types": ["order_management"],
          "supported_token_types": [
            "urn:ietf:params:oauth:token-type:access_token"
          ]
        },
        {
          "audience": "https://billing.provider.example",
          "scope": "customer.read customer.write",
          "supported_token_types": [
            "urn:ietf:params:oauth:grant-type:jwt-bearer"
          ]
        },
        {
          "audience": "https://backend-audit-service.example.com",
          "scope": "audit.read audit.write",
          "supported_token_types": [
            "urn:ietf:params:oauth:token-type:access_token"
          ]
        }
      ]
    }

## Multi-Tenant Target Services {#multi-tenant-target-services}

A Target Service may be multi-tenant, sharing the same authorization server and/or the same resource across two or more tenants, where the tenant is identified by a claim in the issued token. For example, an authorization server `as.saas.example` may host two tenants, `dev` and `staging`, that share the resource `https://api.saas.example`. Because the resource (and authorization server) is shared, the resource indicator alone cannot distinguish the tenants.

To support this case without changing the OAuth 2.0 Token Exchange {{RFC8693}} request contract, the authorization server represents each tenant of a multi-tenant Target Service as a separate target object with a distinct `audience` value. As described for the `audience` property, the client uses the discovered value verbatim as the `audience` parameter in the subsequent token exchange request, and the authorization server resolves it to the tenant-specific audience and tenant context in the issued token. The claim that conveys the tenant in the issued token is determined by the token format and Target Service (for example, the `aud_tenant` claim when the Identity Assertion Authorization Grant {{I-D.oauth-identity-assertion-authz-grant}} is used) and is outside the scope of this specification.

When a Target Service is multi-tenant, the authorization server SHOULD include the `tenant` property so that the client can correlate the Target Service with the tenant context of the resulting issued token, and SHOULD include the `display_name` property so that a client can present a tenant picker to an end user.

If a multi-tenant Target Service requires a distinct client registration for each tenant, the authorization server MAY include the `client_id` property in each target object to indicate the client identifier the client uses with that tenant when presenting the issued token. The issued token is not necessarily presented through token exchange: depending on the Target Service, it may be an authorization grant, such as an ID-JAG {{I-D.oauth-identity-assertion-authz-grant}}, that the client redeems at the Target Service's authorization server (where this `client_id` is used for client authentication), or an access token that the client presents directly to a resource server. If a shared client registration is used across tenants, the `client_id` property is omitted.

The following is a non-normative example of a discovery response for a multi-tenant Target Service with two tenants that share the same resource:

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "supported_targets": [
        {
          "audience": "urn:saas:tenant:dev",
          "tenant": "dev",
          "resource": "https://api.saas.example",
          "scope": "orders.read orders.write",
          "supported_token_types": [
            "urn:ietf:params:oauth:grant-type:jwt-bearer"
          ],
          "display_name": "SaaS Example Dev",
          "client_id": "client-dev"
        },
        {
          "audience": "urn:saas:tenant:staging",
          "tenant": "staging",
          "resource": "https://api.saas.example",
          "scope": "orders.read orders.write",
          "supported_token_types": [
            "urn:ietf:params:oauth:grant-type:jwt-bearer"
          ],
          "display_name": "SaaS Example Staging",
          "client_id": "client-staging"
        }
      ]
    }

The client presents the two tenants to the end user using the `display_name` values (for example, "SaaS Example Dev" and "SaaS Example Staging"). Once the user selects a tenant, the client performs the token exchange using the corresponding `audience` value (for example, `urn:saas:tenant:dev`). When the resulting issued token is later presented at the Target Service, the client uses the corresponding `client_id`, if present and where client authentication applies.

## Relationship to Rich Authorization Requests {#rich-authorization-requests}

Rich Authorization Requests {{RFC9396}} let a client request fine-grained authorization through the `authorization_details` parameter, in addition to or instead of `scope`. Because a token exchange is a token endpoint request, `authorization_details` may accompany it, and the `authorization_details_types` property lets discovery report which RAR types are eligible for use with a target.

This specification and the Authorization Details Types Metadata defined in {{I-D.zehavi-oauth-rar-metadata}} address complementary layers and do not overlap:

* The Authorization Details Types Metadata endpoint is static and authorization-server-wide: it describes the `authorization_details` types an authorization server supports and their JSON Schemas.
* This discovery endpoint is dynamic and per-subject: for a given subject token and client, it reports which of those types are eligible for a specific target, via the `authorization_details_types` property. Whether a given `authorization_details` object is granted still depends on its contents, which the authorization server evaluates when the token exchange is performed.

A client uses this endpoint to learn which authorization details types it may use for a target, and the Authorization Details Types Metadata endpoint to learn how to construct them. This layering mirrors the relationship between `scopes_supported` and `scope`: the metadata endpoint is the static catalog of what the authorization server supports, while this endpoint reports the subset eligible for a specific target. It is also proactive: reporting the eligible types before the token exchange complements the reactive remediation that {{I-D.zehavi-oauth-rar-metadata}} defines for signaling, after a request fails, what authorization was missing. An authorization server MAY publish both the `token_exchange_target_service_discovery_endpoint` and the Authorization Details Types Metadata endpoint (`authorization_details_types_metadata_endpoint`) in its metadata {{RFC8414}}.

## Error Response {#error-response}

If the request failed, the authorization server returns an error response as defined in {{Section 5.2 of RFC6749}}. The response MUST use the `application/json` media type, and the `Content-Type` header MUST be set to `application/json`. In addition to the error codes specified in {{Section 5.2 of RFC6749}}, the following error codes may be returned:

unsupported_token_type
: The authorization server does not support the subject token type indicated by the `subject_token_type` parameter.

The HTTP status code is determined as specified in {{Section 5.2 of RFC6749}}. In particular, when the error is `invalid_client` and the client attempted to authenticate via the `Authorization` request header field, the authorization server responds with HTTP 401 (Unauthorized) and includes a `WWW-Authenticate` response header field; otherwise it responds with HTTP 400 (Bad Request).

When the authorization server rate-limits a client (see {{information-disclosure}}), it MAY respond with HTTP 429 (Too Many Requests) {{RFC6585}} and SHOULD include a `Retry-After` header field {{RFC9110}}.

### Error Response Example

The following is an example of an error response:

    HTTP/1.1 400 Bad Request
    Content-Type: application/json

    {
      "error": "invalid_request",
      "error_description": "The subject token is invalid or expired"
    }

# Example

This example demonstrates a complete cross-domain identity chaining workflow described in {{I-D.ietf-oauth-identity-chaining}} with the addition of the token exchange target service discovery endpoint.

## Step 1: Discover Target Services

The client begins with a subject access token issued by Domain A and calls the target service discovery endpoint to learn which Token Exchange Targets are available.

### Discovery Request

    POST https://as.domainA.example/target-discovery HTTP/1.1
    Host: as.domainA.example
    Content-Type: application/x-www-form-urlencoded

    client_id=client-A
    &client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
    &client_assertion=eyJhbGciOi...
    &subject_token=SlAV32hkKG...ACCESSTOKEN...
    &subject_token_type=urn:ietf:params:oauth:token-type:access_token

### Discovery Response

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "supported_targets": [
        {
          "audience": "https://api.domainB.example",
          "resource": [
            "https://api.domainB.example/orders",
            "https://api.domainB.example/inventory"
          ],
          "scope": "orders.read inventory.read",
          "supported_token_types": [
            "urn:ietf:params:oauth:grant-type:jwt-bearer"
          ]
        }
      ]
    }

From this response, the client learns that it may request a token exchange for the audience `https://api.domainB.example` with the resources `https://api.domainB.example/orders` and `https://api.domainB.example/inventory` and the scopes `orders.read` and `inventory.read`. The `supported_token_types` value tells the client that, in the subsequent token exchange, it may request a JWT to be presented to the Target Service as a `jwt-bearer` authorization grant {{RFC7523}}.

## Step 2: Determine Token Types (Optional)

The discovery response's `supported_token_types` property indicates the token types the client may request for the target. In this example it listed `urn:ietf:params:oauth:grant-type:jwt-bearer`, so the client proceeds directly to the token exchange. When a discovery response omits `supported_token_types`, the client determines the requestable token types from the Target Service's documentation or other out-of-band configuration.

## Step 3: Perform Token Exchange

The client now performs a token exchange with Domain A's token endpoint, requesting a JWT for Domain B using the values discovered in the previous steps.

### Token Exchange Request

    POST https://as.domainA.example/token HTTP/1.1
    Host: as.domainA.example
    Content-Type: application/x-www-form-urlencoded

    grant_type=urn:ietf:params:oauth:grant-type:token-exchange
    &client_id=client-A
    &client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
    &client_assertion=eyJhbGciOi...
    &subject_token=SlAV32hkKG...ACCESSTOKEN...
    &subject_token_type=urn:ietf:params:oauth:token-type:access_token
    &requested_token_type=urn:ietf:params:oauth:grant-type:jwt-bearer
    &audience=https://api.domainB.example
    &resource=https://api.domainB.example/orders
    &resource=https://api.domainB.example/inventory
    &scope=orders.read inventory.read

### Token Exchange Response

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "access_token": "eyJraWQiOi...DOMAINB.JWT...",
      "issued_token_type": "urn:ietf:params:oauth:grant-type:jwt-bearer",
      "token_type": "N_A",
      "expires_in": 3600,
      "scope": "orders.read inventory.read"
    }

The client now holds a Domain-B-scoped JWT token that can be used to access the Target Service, derived from Domain A's access token through the token exchange process.

# Security Considerations

## Subject Token Validation

The authorization server MUST validate the subject token as specified in {{subject-token-processing}}. In particular, it MUST verify that the token is well-formed for the indicated type, is correctly signed by a trusted issuer, has not expired, has not been revoked (where revocation status is applicable and available), and, where client authentication is required, is associated with the authenticated client. A token that fails validation MUST be rejected as described in {{subject-token-processing}}.

## Client Authentication

The authorization server SHOULD require client authentication for the discovery endpoint to prevent unauthorized access to authorization information. The authorization server MUST support at least one of the client authentication methods defined in {{Section 2.3 of RFC6749}}. If client authentication is required but not provided, the authorization server MUST return an error response with the error code `invalid_client` as described in {{error-response}}.

## Authorization Policy Enforcement {#authorization-policy-enforcement}

The authorization server MUST evaluate both the subject token and the client's permissions when determining which Token Exchange Targets to return. The server MUST only return targets that the client is authorized to request in a subsequent token exchange operation. The specific authorization policy evaluation mechanism is implementation-specific and MAY be based on scopes, claims, resource-based access control, attribute-based access control, or other authorization models supported by the authorization server.

## Information Disclosure {#information-disclosure}

The discovery endpoint reveals information about which Target Services are available for a given subject token and client. This information could be used by an attacker to enumerate authorization relationships. To mitigate this risk:

* The authorization server SHOULD require client authentication
* The authorization server SHOULD apply rate limiting to prevent enumeration attacks
* The authorization server MAY return different results based on the authenticated client to limit information disclosure
* The authorization server SHOULD log access to the discovery endpoint for security monitoring

## Multi-Tenant Information Disclosure

For multi-tenant Target Services, the `tenant`, `display_name`, and `client_id` properties reveal the existence of specific tenants and, where present, the per-tenant client registration topology of the authorization server. This information could be used by an attacker to enumerate tenants or client registrations. The authorization server MUST only return target objects, including their tenant and client registration properties, that the authenticated client is authorized to discover, applying the same authorization policy enforcement and information disclosure mitigations described in this section. The authorization server SHOULD NOT include a `client_id` value that the authenticated client is not authorized to use.

## Token Confidentiality

The subject token is transmitted in the request. The authorization server MUST require the use of TLS as specified in {{Section 1.6 of RFC6749}} to protect the token in transit. Because the request body carries a live credential, the authorization server MUST NOT record the `subject_token` (or other equally sensitive request content) in logs, audit records, or error reports, and SHOULD avoid emitting it where it could be retained by intermediaries.

## Error Handling

The authorization server MUST NOT provide detailed error messages that could aid an attacker in understanding the authorization server's internal state or policies. Error responses SHOULD be generic and not reveal specific information about why a request failed beyond what is necessary for the client to correct the request.

# Privacy Considerations

The discovery endpoint returns information about authorization relationships between subjects, clients, and Target Services. This information may be considered privacy-sensitive, as it reveals:

* Which Target Services a subject is authorized to access
* The scope of permissions available for each Target Service
* The existence of authorization relationships
* For multi-tenant Target Services, the specific tenants a subject is associated with or authorized to access, as conveyed by the `tenant`, `display_name`, and `client_id` properties

To protect privacy:

* The authorization server SHOULD only return information that the authenticated client is authorized to know
* The authorization server SHOULD apply the principle of least privilege when determining which Token Exchange Targets to return
* The authorization server SHOULD only disclose tenant-identifying properties (`tenant`, `display_name`, and `client_id`) for the tenants that both the subject and the authenticated client are authorized to access
* The authorization server SHOULD log access to the discovery endpoint in accordance with applicable privacy regulations
* The authorization server MAY provide mechanisms for subjects to control or limit the information returned by the discovery endpoint

# IANA Considerations

## OAuth Authorization Server Metadata Registry

This specification registers the following value in the IANA "OAuth Authorization Server Metadata" registry established by {{RFC8414}}.

Metadata Name: `token_exchange_target_service_discovery_endpoint`

Metadata Description: URL of the token exchange target service discovery endpoint

Change Controller: IETF

Specification Document(s): \[\[ This document \]\]

## OAuth Extensions Error Registry

This specification registers the following error in the IANA "OAuth Extensions Error Registry" established by {{RFC6749}}, adding a usage location for the token exchange target service discovery endpoint. The error name `unsupported_token_type` is also registered for other usage locations (for example, the token revocation endpoint); this registration adds a distinct usage location and does not change the existing entries.

Error Name: `unsupported_token_type`

Error Usage Location: Token exchange target service discovery endpoint response

Related Protocol Extension: OAuth 2.0 Token Exchange Target Service Discovery

Change Controller: IETF

Specification Document(s): \[\[ This document \]\]

--- back

# Acknowledgments
{:numbered="false"}

The authors would like to thank the following individuals who contributed ideas, feedback, and wording that helped shape this specification: Max Gerber

# Document History
{:numbered="false"}

-03

* Added optional discovery of Rich Authorization Requests (RFC 9396) authorization details: the optional `authorization_details_types` target property, which lists the authorization details types eligible for a target (listing a type does not authorize a specific `authorization_details` object), and a Relationship to Rich Authorization Requests section that aligns this endpoint (dynamic, per-subject) with the Authorization Details Types Metadata endpoint (static, authorization-server-wide) of draft-zehavi-oauth-rar-metadata. RFC 9396 is referenced normatively.

-02

* Added optional support for multi-tenant Target Services that share an authorization server and/or resource across tenants, via the optional `tenant`, `display_name`, and `client_id` properties and a new Multi-Tenant Target Services section; a distinct `audience` per tenant selects the tenant without changing the OAuth 2.0 Token Exchange request contract.
* Added a terminology section defining "Target Service" (the downstream service) and "Token Exchange Target" (a pre-authorized combination of Token Exchange request parameters); named each response element a "target object"; and clarified that discovery returns authorized Token Exchange Targets, not an assertion that a service is reachable.
* Defined the `audience` property as the exact, opaque value the client uses verbatim in Token Exchange, with authorization-server-defined syntax unless profiled, and specified that the `(audience, resource set)` pair uniquely identifies a target whose `scope` is the aggregate authorized for that pair.
* Clarified requesting a JWT for use as an RFC 7523 `jwt-bearer` authorization grant: examples and `supported_token_types` use `urn:ietf:params:oauth:grant-type:jwt-bearer` as the token type value (replacing the invalid `urn:ietf:params:oauth:token-type:jwt-bearer`), a token type identifier must identify the requested token type or usage, and RFC 7523 is now normative.
* Aligned the `resource` property with RFC 8707 (absolute URI, no fragment, array entries mapping to repeated `resource` parameters).
* Consolidated empty-value handling into a single rule: optional properties with an empty string, empty array, or null value are omitted, and a present `scope` contains at least one value.
* Clarified the client identity model (client authentication parameters are not unknown parameters; public-client identification) and that the requesting client's permissions are evaluated.
* Switched the caching reference to RFC 9111 (normative): no-store by default, with bounded private caching as an opt-in and no shared-cache storage.
* Defined error behavior: registered an `unsupported_token_type` usage location in the OAuth Extensions Error Registry, mapped `invalid_client` to HTTP 401 per RFC 6749, and added HTTP 429 with `Retry-After` (RFC 6585, RFC 9110) for rate limiting.
* Strengthened security and privacy guidance: the subject token must not be logged, the subject-token revocation check applies where revocation status is available, and multi-tenant tenant/client information is disclosed only to authorized clients.
* Moved the Authorization Server Metadata parameter into the endpoint section so the endpoint URL is discovered before it is used.
* Editorial: replaced the obsolete RFC 7159 reference with RFC 8259, corrected the `abbrev` value, set IANA change controllers to IETF, added client authentication to the examples, narrowed the abstract, and made terminology and capitalization consistent.

-01

* Changed discovery response to an object for extensibility with supported_targets param
* Updated references

-00

* Initial revision

