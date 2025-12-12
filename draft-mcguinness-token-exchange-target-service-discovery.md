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
  RFC7159:
  RFC8707:
  RFC8414:
  RFC3986:
  RFC8693:

informative:

--- abstract

This specification defines a method for OAuth 2.0 clients to discover the set of available target services (audiences, resources, and scopes) for a given subject token when performing OAuth 2.0 Token Exchange. The discovery endpoint accepts any subject token type registered in the OAuth Token Type Registry and returns values that are valid inputs to subsequent Token Exchange requests, supporting advanced use cases such as identity chaining and cross-domain delegation.

--- middle

# Introduction

OAuth 2.0 Token Exchange {{RFC8693}} enables a client to present a token issued in one security domain to an Authorization Server and securely trade it for a new token issued for another domain. The exchanged token is minted with the target server’s token requirements, including its own audience, resource indicators, and scopes, so it becomes valid and enforceable for the desired downstream service. This enables controlled cross-domain access, removes the confused-deputy problem by ensuring tokens are explicitly targeted to the correct service, and avoids requiring direct trust between the original token issuer and the target service.

Authorization Servers may be capable of issuing tokens to multiple services for a given subject token and client, but the client must already know which values it may request. Today, this knowledge is typically provided through static configuration, proprietary APIs, or informal documentation, leading to brittle integrations and unnecessary Token Exchange failures, particularly when subjects are authorized to access only a subset of available services.

This specification defines the OAuth 2.0 Token Exchange Target Service Discovery Endpoint, a standardized mechanism that enables clients to dynamically discover the set of available Token Exchange targets (audiences, resources, and scopes) for a given subject token. The authorization server evaluates both the subject token and the client's permissions and returns only the values the client is authorized to request.

This extension is especially valuable in identity chaining and cross-domain authorization scenarios, such as:

* Multi-tier microservices architectures where tokens must be progressively exchanged with narrower permissions.
* Cross-organizational and federated systems where downstream services exist in separate administrative domains.
* Cross-cloud or hybrid deployments requiring strict control over which domains may issue or accept exchanged tokens.
* Delegated flows where derived tokens represent constrained or transformed identities across trust boundaries.

This specification provides several benefits:

* Eliminates the need for static configuration by allowing clients to dynamically discover available target services
* Reduces Token Exchange failures by enabling clients to learn valid targets before attempting a Token Exchange request
* Supports identity chaining by allowing the authorization server to provide the next allowable "hop" in a derived-identity chain
* Enhances cross-domain trust by preventing clients from requesting tokens for unauthorized or untrusted domains
* Improves interoperability by standardizing previously proprietary discovery mechanisms
* Improves developer and operator experience by enabling tools to automatically enumerate valid downstream targets
* Supports dynamic authorization by reflecting real-time policy evaluation, client permissions, and subject token context

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Token Exchange Target Service Discovery Endpoint

This specification defines a new endpoint for OAuth 2.0 Authorization Servers that enables clients to discover the set of available target services (audiences, resources, and scopes) for a given subject token when performing OAuth 2.0 Token Exchange {{RFC8693}}.

The endpoint is identified by the Authorization Server metadata parameter `token_exchange_target_service_discovery_endpoint`, as defined in Section 7.

## Endpoint Request

The client makes a request to the token exchange target service discovery endpoint by sending an HTTP POST request to the endpoint URL. The client MUST use TLS as specified in Section 1.6 of {{RFC6749}}.

The endpoint URL MUST be obtained from the Authorization Server's metadata document {{RFC8414}} using the `token_exchange_target_service_discovery_endpoint` parameter. The endpoint URL is an absolute URL as defined by {{RFC3986}}.

The client sends the parameters using the `application/x-www-form-urlencoded` format per Appendix B of {{RFC6749}}.

### Request Parameters

The following parameters are used in the request:

subject_token
: REQUIRED. The subject token used as input to discovery. The token type is indicated by the `subject_token_type` parameter.

subject_token_type
: REQUIRED. An identifier, as described in Section 5 of {{RFC8693}}, that indicates the type of the `subject_token` parameter. This identifier MUST be registered in the "OAuth Token Type Registry" as defined in Section 5.1 of {{RFC8693}}.

The client MAY include additional parameters as defined by the OAuth 2.0 specification or extensions. Client authentication MAY be required by the authorization server. The means of client authentication are defined by the authorization server and MAY include any method supported by the authorization server, including those defined in Section 2.3 of {{RFC6749}}.

### Request Example

The following is an example of a discovery request:

    POST /target-discovery HTTP/1.1
    Host: as.example.com
    Content-Type: application/x-www-form-urlencoded

    subject_token=SlAV32hkKG...ACCESSTOKEN...
    &subject_token_type=urn:ietf:params:oauth:token-type:access_token


## Endpoint Response

The authorization server validates the request and returns a response with the discovery results. The response MUST use the `application/json` media type as specified in {{RFC7159}}. The `Content-Type` header of the response MUST be set to `application/json`.

### Successful Response

If the request is valid and authorized, the authorization server returns a JSON array of available token exchange targets. Each element in the array represents a target service that the client is authorized to request in a subsequent token exchange operation.

Each target service object contains the following fields:

audience
: OPTIONAL. A string indicating an available audience value for token exchange, as defined in Section 2.1 of {{RFC8693}}.

resource
: OPTIONAL. A string or array of strings indicating available resource indicator values, as defined in {{RFC8707}}.

At least one of `audience` or `resource` MUST be present in each target service object. A target service object MAY specify both `audience` and `resource`.

scope
: OPTIONAL. A space-delimited list of OAuth 2.0 scope values, as defined in Section 3.3 of {{RFC6749}}, that are available for this target service.

supported_token_types
: OPTIONAL. An array of strings indicating the token types that may be requested for this target service in a subsequent token exchange operation. Each string MUST be a registered token type identifier from the "OAuth Token Type Registry" as defined in Section 5.1 of {{RFC8693}}. If omitted, the client may use any token type supported by the authorization server.

If no target services are available for the given subject token and client, the authorization server returns an empty array `[]`.

### Response Example

The following is an example of a successful discovery response:

    HTTP/1.1 200 OK
    Content-Type: application/json

    [
      {
        "audience": "https://api.example.com",
        "resource": ["https://api.example.com/orders", "https://api.example.com/inventory"],
        "scope": "orders.read inventory.read",
        "supported_token_types": [
          "urn:ietf:params:oauth:token-type:jwt-bearer"
        ]
      },
      {
        "audience": "https://api.example.com",
        "scope": "analytics.read",
        "supported_token_types": [
          "urn:ietf:params:oauth:token-type:jwt-bearer"
        ]
      },
      {
        "resource": ["https://api.example.com/reports"],
        "scope": "reports.read",
        "supported_token_types": [
          "urn:ietf:params:oauth:token-type:jwt-bearer"
        ]
      }
    ]

## Error Response

If the request failed, the authorization server returns an error response as defined in Section 5.2 of {{RFC6749}}. The response MUST use the `application/json` media type, and the `Content-Type` header MUST be set to `application/json`. The following error codes may be returned:

invalid_request
: The request is missing a required parameter, includes an invalid parameter value, includes a parameter more than once, or is otherwise malformed. This error code is also returned when the provided subject token is invalid, expired, revoked, or does not match the subject token type, or when another error condition occurred during token validation.

invalid_client
: Client authentication failed (e.g., unknown client, no client authentication included, or unsupported authentication method).

unauthorized_client
: The authenticated client is not authorized to use this discovery endpoint.

unsupported_token_type
: The authorization server does not support the subject token type indicated by the `subject_token_type` parameter.

server_error
: The authorization server encountered an unexpected condition that prevented it from fulfilling the request.

temporarily_unavailable
: The authorization server is currently unable to handle the request due to a temporary overloading or maintenance of the server.

### Error Response Example

The following is an example of an error response:

```
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "invalid_request",
  "error_description": "The subject token is invalid or expired"
}
```

# Example

This example demonstrates a complete cross-domain identity chaining workflow using the token exchange target service discovery endpoint.

## Step 1: Discover Target Services

The client begins with a subject access token issued by Domain A and calls the target service discovery endpoint to learn which target services are available.

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

    [
      {
        "audience": "https://api.domainB.example",
        "resource": ["https://api.domainB.example/orders", "https://api.domainB.example/inventory"],
        "scope": "orders.read inventory.read",
        "supported_token_types": [
          "urn:ietf:params:oauth:token-type:jwt-bearer"
        ]
      }
    ]

From this response, the client learns that it may request a token exchange for the audience `https://api.domainB.example` with the resources `https://api.domainB.example/orders` and `https://api.domainB.example/inventory` and the scopes `orders.read` and `inventory.read`. The client also learns that JWT bearer tokens are supported for this target service.

## Step 2: Discover Token Types (Optional)

If the discovery response does not include `supported_token_types`, or if the client needs to verify token type support, the client may query the Authorization Server Metadata of Domain B to determine which token types are supported for the target service.

### Metadata Request

    GET https://as.domainB.example/.well-known/oauth-authorization-server HTTP/1.1
    Host: as.domainB.example

### Metadata Response

HTTP/1.1 200 OK
Content-Type: application/json

    {
      "issuer": "https://as.domainB.example",
      "token_endpoint": "https://as.domainB.example/token",
      "requested_token_types_supported": [
        "urn:ietf:params:oauth:token-type:jwt-bearer"
      ]
    }

This confirms that Domain B supports JWT bearer tokens as a requested token type.

## Step 3: Perform Token Exchange

The client now performs a token exchange with Domain A's token endpoint, requesting a JWT for Domain B using the values discovered in the previous steps.

### Token Exchange Request

    POST https://as.domainA.example/token HTTP/1.1
    Host: as.domainA.example
    Content-Type: application/x-www-form-urlencoded

    grant_type=urn:ietf:params:oauth:grant-type:token-exchange
    &subject_token=SlAV32hkKG...ACCESSTOKEN...
    &subject_token_type=urn:ietf:params:oauth:token-type:access_token
    &requested_token_type=urn:ietf:params:oauth:token-type:jwt-bearer
    &audience=https://api.domainB.example
    &resource=https://api.domainB.example/orders
    &resource=https://api.domainB.example/inventory
    &scope=orders.read inventory.read

### Token Exchange Response

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "access_token": "eyJraWQiOi...DOMAINB.JWT...",
      "issued_token_type": "urn:ietf:params:oauth:token-type:jwt-bearer",
      "token_type": "N_A",
      "expires_in": 3600,
      "scope": "orders.read inventory.read"
    }

The client now holds a Domain B-scoped JWT token that can be used to access the target service, derived from Domain A's access token through the token exchange process.

# Authorization Server Metadata

This specification defines the following authorization server metadata {{RFC8414}} parameter to enable clients to discover the token exchange target service discovery endpoint:

token_exchange_target_service_discovery_endpoint
: URL of the token exchange target service discovery endpoint. This URL MUST use the `https` scheme. The authorization server SHOULD publish this metadata value.

The following is a non-normative example of the metadata:

    {
      "issuer": "https://as.example.com",
      "token_exchange_target_service_discovery_endpoint": "https://as.example.com/target-discovery"
    }

# Security Considerations

## Subject Token Validation

The authorization server MUST validate the subject token provided in the request. The validation process MUST verify that:

* The subject token is valid and not expired
* The subject token type matches the `subject_token_type` parameter
* The subject token is associated with the authenticated client (if client authentication is required)
* The subject token has not been revoked

If any validation fails, the authorization server MUST return an `invalid_request` error as described in Section 2.2.

## Client Authentication

The authorization server SHOULD require client authentication for the discovery endpoint to prevent unauthorized access to authorization information. The authorization server MUST support at least one of the client authentication methods defined in Section 2.3 of {{RFC6749}}.

## Authorization Policy Enforcement

The authorization server MUST evaluate both the subject token and the client's permissions when determining which target services to return. The server MUST only return target services that the client is authorized to request in a subsequent token exchange operation.

## Information Disclosure

The discovery endpoint reveals information about which target services are available for a given subject token and client. This information could be used by an attacker to enumerate authorization relationships. To mitigate this risk:

* The authorization server SHOULD require client authentication
* The authorization server SHOULD apply rate limiting to prevent enumeration attacks
* The authorization server MAY return different results based on the authenticated client to limit information disclosure
* The authorization server SHOULD log access to the discovery endpoint for security monitoring

## Token Confidentiality

The subject token is transmitted in the request. The authorization server MUST require the use of TLS as specified in Section 1.6 of {{RFC6749}} to protect the token in transit.

## Error Handling

The authorization server MUST NOT provide detailed error messages that could aid an attacker in understanding the authorization server's internal state or policies. Error responses SHOULD be generic and not reveal specific information about why a request failed beyond what is necessary for the client to correct the request.

# Privacy Considerations

The discovery endpoint returns information about authorization relationships between subjects, clients, and target services. This information may be considered privacy-sensitive, as it reveals:

* Which target services a subject is authorized to access
* The scope of permissions available for each target service
* The existence of authorization relationships

To protect privacy:

* The authorization server SHOULD only return information that the authenticated client is authorized to know
* The authorization server SHOULD apply the principle of least privilege when determining which target services to return
* The authorization server SHOULD log access to the discovery endpoint in accordance with applicable privacy regulations
* The authorization server MAY provide mechanisms for subjects to control or limit the information returned by the discovery endpoint

# IANA Considerations

## OAuth Authorization Server Metadata Registry

This specification registers the following value in the IANA "OAuth Authorization Server Metadata" registry {{RFC8414}} established by {{RFC8414}}.

Metadata Name: `token_exchange_target_service_discovery_endpoint`

Metadata Description: URL of the token exchange target service discovery endpoint

Change Controller: IESG

Specification Document(s): [[ this document ]]

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.

