# Draft: OAutorize

This is an Internet Draft on the automation of OAuth 2.0 authorization mechanisms into future web browsers.
Because of its OAuth authorization automation aspects, it is called O[Auto]rize (the missing 'h' is intended).

## Abstract

OAuth AUTOrize is a proposed OAuth-based Framework that targets three goals.
First, increasing web-based API security by leveraging transaction tokens for frontend-to-backend communication.
Second, automating security-related concepts like authorization flows, token exchanges and authorizing requests, so that client developers no more need to care about authorization anymore.
Third, finally protecting OAuth against token stealing via XSS attacks by implementing the framework into web browsers.

## Introduction

OAuth 2.0 in the year 2026 has many useful extensions to improve security.
However, it is hard for developers to learn all of them and implement them correctly by following all best practices.
Even with the use of AI, it is hard to maintain and secure large code bases.

Modern security concepts like transaction tokens further improve system security by utilizing transaction tokens in backend services.
Using transaction tokens on the client comes with the advantage that a compromized resource server cannot use a received access token to compromize data on other resource servers.
This improves security by implementing least-privilege and zero trust concepts, but heavily increases complexity of the token handling implementation at the client.

In MCP, the MCP client automatically handles authorization requests and bearer token insertion into authorization headers.
This draft applies the same concept to generic HTTP client implementation.
Client developers can use this HTTP client to implement their API requests via HTTP and do not have to care about OAuth-related authorization stuff.

Such a generic HTTP client which automatically handles OAuth-based authorization can also be implemented into the Fetch API and the XMLHttpRequest API of future web browsers as an extension of the existing FedCM standard.
As a result, web developers benefit from an authorization flow that is entirely implemented by the web browser via FedCM where users only need to choose their pre-authenticated user account in the browser.
Therefore, the browser can handle the token management in an environment isolated from the web application's JavaScript environment.
This solves the main issue of OAuth for web applications, that access tokens - by design - were vulnerable to extractions by cross-site scripting (XSS) because the had to be accessible by JavaScript to put them into the authorization header.
With OAuth AutoSecure, the browser puts bearer tokens into the authorization header and stores them inaccessible by the JavaScript API, as this is already done by HTTP-only cookies.

## Related Work

- [Deferred Token Response for OAuth Authorization](https://github.com/gniero/oidc-dtr-resources): May be implemented as a new non-blocking mediation mode for FedCM
- [MCP Authorization](https://modelcontextprotocol.io/specification/draft/basic/authorization): Concept for automation of authorization in MCP
- [Transaction Tokens](https://datatracker.ietf.org/doc/draft-ietf-oauth-transaction-tokens/): Can be used to improve security for clients that interact with multiple resource servers
- [OAuth 2.0 Protected Resource Metadata](https://datatracker.ietf.org/doc/rfc9728/): Automated discovery of the resource server's access token demands
- [OAuth 2.0 Authorization Server Metadata](https://datatracker.ietf.org/doc/rfc8414/): Automated discovery of the authorization server
- [OAuth Client ID Metadata](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/): Automated discovery of the Client
- [FedCM](https://github.com/w3c-fedid/FedCM): Browser API for managing the resource owner's identity at multiple authorization servers

## Concept

### Classic OAuth Resource Request

This is a classic OAuth 2.0 example where the resource owner authorizes the client which uses the access token issued by the authorization server to access protected resources from the resource server.

1. The resource owner opens the web application in their web browser.

```bash
open https://client.example.com/
```

2. The web applications uses FedCM to trigger the authorization code flow.

```js
const credential = await navigator.credentials.get({
  identity: {
    providers: [{
      configURL: 'https://as.example.com/fedcm.json',
      clientId: 'https://client.example.com/',
      fields: ['profile', 'email'], // Requested default scopes
      authorization: true // Indicates, that this is an OAuth authorization request
    }]
  }
});
```

3. During the authorization code flow, the web browser loads the client metadata from `{base-url}/.well-known/client-metadata`:

```json
{
  "description": "An example application", // Description of the client displayed at authorization server
  "authorization": [ // List of intended authorization servers to use
    {
      "authorizationServer": "https://as.example.com/", // Base URL of authorization server
      "clientId": "https://client.example.com/", // The client's client ID
      "resources": [ // List of resource servers intended to use
        "https://rs1.example.com/*",
        "https://rs.example.com/rs2/*",
      ]
    }
  ]
}
```

4. The web browser remembers this client metadata for all FedCM credentials. They can be accessed as follows:

```js
print(credential.clientDescription.authorization[0].authorizationServer); // Returns "https://as.example.com/"
print(credential.clientDescription.authorization[0].clientId); // Returns "https://client.example.com/"
print(credential.clientDescription.authorization[0].resources); // Returns ["https://rs1.example.com/*","https://rs.example.com/rs2/*"]
```

5. In the authorization request, the authorization server identifies the client by its client ID.
This could be a normal client ID that is pre-registered by client developers at the authorization server.
However, if the authorization server supports the [OAuth 2.0 Client ID Metadata Document draft](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/), the authorization server browses the client metadata document to dynamically register the client.

6. The web application sends a resource request using the Fetch API:

```js
const response = await fetch('https://rs1.example.com/userdata', {
  credential: credential  // The credential from FedCM
});
```

7. In background, the web browser checks whether any of the credential's resources match the requested URL.
If not, this is a normal GET request.
Otherwise, the browser attaches the best matching bearer token available to the request in the authorization header:

```http
GET /userdata HTTP/1.1
Host: rs1.example.com
Authorization: Bearer {access-token}
```

8. If the access token misses required scopes, the resource server answers with the following HTTP response:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="as.example.com",
                  error="insufficient_scope",
                  error_description="The request requires higher privileges."
                  scope="rs1_read"
```

9. Instead of resolving the asynchronouse fetch call with this error, the user agent uses a dialog to inform the resource owner that the request has failed and demands more permissions.
If the user clicks "accept", FedCM initiates another authorization flow with the extended scope (`profile email rs1_read`).
In the FedCM popup containing the authorization server's authorization screen, the resource owner grants permissions to the client.
Afterwards, the Fetch API repeats the request with the updated scope and resolves the asynchronous fetch call with the final response.

10. The currently granted scope can be introspected via FedCM API as follows:

```js
print(credential.fields); // Returns ["profile", "email", "rs1_read"]
```

### Resource Request with Token Exchange

This is an advanced example how a resource owner the client access at the authorization server which issues an access token to the client.
The client exchanges this access token for a transaction token at the resource server's transaction token service before the client uses this transaction token to access protected resources.
This is a useful mechanism to mitigate leaking access tokens accepted by multiple resource servers to a compromized resource sever.

The expected authorization flow is explained in Figure 1.

```
+--------+     (1)     +-------------------+
|        | ----------> |   Authorization   |
|        | <---------- |      Server       |
|        |     (2)     +-------------------+
|        |
|        |     (3)     +-------------------+
| Client | ----------> | Transaction Token |
|        | <---------- |     Service       |
|        |     (4)     +-------------------+
|        |
|        |     (5)     +-------------------+
|        | ----------> |     Resource      |
|        | <---------- |      Server       |
|        |     (6)     +-------------------+
+--------+
```
Fig. 1: Authorization Flow with transaction token.

1. The client sends an authorization request to the authorization server.
2. The authorization server issues an access token to the client.
3. The client sends a token exchange request to the transaction token service using the access token to prove authorization.
4. The transaction token service issues a transaction token to the client.
5. The client sends a resource request to the resource server using the transaction token to prove authorization.
6. The resource server issues protected resources to the client.

#### Discovery

To automate such a flow, the client must learn from resource server and transaction token service metadata, which tokens are required for the authorization request.
The discovery process works as follows:

1. The client introspects the protected resource metadata on `https://rs.example.com/.well-known/oauth-protected-resource/userdata`:

```json
{
  "resource": "https://rs.example.com",
  "authorization_servers": [
    "https://tts.example.com" // Token from transaction token service required
  ],
  "token_types_supported": [
    "urn:ietf:params:oauth:token-type:txn_token" // Requires transaction tokens
  ],
  "bearer_methods_supported": [
    "header"
  ],
  "scopes_supported": [
    "data_read",
    "data_write"
  ],
  "resource_documentation": "https://resource.example.com/resource_documentation.html"
}
```

2. The client introspects the transaction token service's metadata on `https://tts.example.com/.well-known/oauth-authorization-server`:

```json
{
  "issuer": "https://tts.example.com",
  "authorization_servers": [
    "https://as.example.com" // Token from authorization server is accepted
  ],
  "token_endpoint": "https://tts.example.com/token",
  "grant_types_supported": [
    "urn:ietf:params:oauth:grant-type:token-exchange" // Required for transaction token services
  ],
  "token_types_supported": [
    "urn:ietf:params:oauth:token-type:txn_token" // Can issue transaction tokens
  ],
  "token_endpoint_auth_methods_supported": [
    "client_secret_basic",
    "private_key_jwt"
  ],
  "token_endpoint_auth_signing_alg_values_supported": [
    "RS256",
    "ES256"
  ],
  "userinfo_endpoint": "https://tts.example.com/userinfo",
  "jwks_uri": "https://tts.example.com/jwks.json",
  "registration_endpoint": "https://tts.example.com/register",
  "scopes_supported": [
    "data_read",
    "data_write"
  ],
  "service_documentation":
    "http://tts.example.com/service_documentation.html"
}
```

3. The client introspects the authorization server's metadata on `https://as.example.com/.well-known/oauth-authorization-server`:

```json
{
  "issuer": "https://as.example.com",
  "authorization_endpoint": "https://as.example.com/authorize", // Endpoint for authorization request
  "token_endpoint": "https://as.example.com/token", // Endpoint for token request
  "token_endpoint_auth_methods_supported": [
    "client_secret_basic",
    "private_key_jwt"
  ],
  "token_endpoint_auth_signing_alg_values_supported": [
    "RS256",
    "ES256"
  ],
  "token_types_supported": [
    "urn:ietf:params:oauth:token-type:access_token" // can issue access tokens
  ],
  "userinfo_endpoint": "https://as.example.com/userinfo",
  "jwks_uri": "https://as.example.com/jwks.json",
  "registration_endpoint": "https://as.example.com/register",
  "scopes_supported": [
    "openid",
    "profile",
    "email",
    "offline_access",
    "data_read",
    "data_write"
  ],
  "response_types_supported": [
    "code"
  ],
  "service_documentation": "http://as.example.com/service_documentation.html"
}
```

#### Obtain Authorization

To obtain authorization, the client must follow these steps after obtaining an access token from the authorization server `https://as.example.com`:

1. Request a transaction token from `https://tts.example.com/token` with the access token from `https://as.example.com`:

```http
POST /token HTTP/1.1
Host: tts.example.com
Authorization: Basic Base64("https%3A//client.example.com:")
Content-Type: application/x-ww-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange&
resource=https://rs.example.com/userdata& // Resource identifier -> Resolves audience
subject_token=accVkjcJyb4BWCxGsndESCJQbdFMogUC5PbRDqceLTC& // Access token from https://as.example.com
subject_token_type=urn:ietf:params:oauth:token-type:access_token& // Request with an access token
requested_token_type=urn:ietf:params:oauth:token-type:txn_token // Request a transaction token
```

2. The transaction token service issues a transaction token:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-cache, no-store

{
  "access_token":"eyJhbGciOiJFUzI1NiIsImtpZCI6IjllciJ9.eyJhdWQiOiJo
    dHRwczovL2JhY2tlbmQuZXhhbXBsZS5jb20iLCJpc3MiOiJodHRwczovL2FzLmV
    4YW1wbGUuY29tIiwiZXhwIjoxNDQxOTE3NTkzLCJpYXQiOjE0NDE5MTc1MzMsIn
    N1YiI6ImJkY0BleGFtcGxlLmNvbSIsInNjb3BlIjoiYXBpIn0.40y3ZgQedw6rx
    f59WlwHDD9jryFOr0_Wh3CGozQBihNBhnXEQgU85AI9x3KmsPottVMLPIWvmDCM
    y5-kdXjwhw", // The transaction token
  "issued_token_type": "urn:ietf:params:oauth:token-type:txn_token",
  "token_type":"Bearer",
  "expires_in":60
}
```

#### Resource Request

To request the protected resource, the client performs the following request:

```http
GET /userdata HTTP/1.1
Host: rs.example.com
Authorization: Bearer eyJhbGciOiJFUzI1NiIsImtpZCI6IjllciJ9.eyJhdWQiOiJodHRwczovL2JhY2tlbmQuZXhhbXBsZS5jb20iLCJpc3MiOiJodHRwczovL2FzLmV4YW1wbGUuY29tIiwiZXhwIjoxNDQxOTE3NTkzLCJpYXQiOjE0NDE5MTc1MzMsInN1YiI6ImJkY0BleGFtcGxlLmNvbSIsInNjb3BlIjoiYXBpIn0.40y3ZgQedw6rxf59WlwHDD9jryFOr0_Wh3CGozQBihNBhnXEQgU85AI9x3KmsPottVMLPIWvmDCMy5-kdXjwhw


```

## Edge Cases

### Multiple Chained Authorization Servers

The following protected resource metadata lists one transaction token service and one authorization server.

```json
{
  "resource": "https://rs.example.com",
  "authorization_servers": [
    "https://tts.example.com", // Transaction token service
    "https://as.example.com" // Authorization server
  ],
  ...
}
```

The client needs an access token from the authorization server to exchange it for an access token at the transaction token service.
Therefore, the shortest way to obtain authorization is requesting an access token from the authorization server directly.
Since the client sends the request with a valid `credential` object from the authorization server, the client will prefer the access token issued by `https://as.example.com`.
If the access token has no sufficient scope, the client will try to extend the scope.
If extending the scope fails, or the client has no matching audience, the client will fall back to the alternative authorization server, `https://tts.example.com`.

## Security Considerations

### XSS Protection

Since the access token and refresh token is neither accessible through the FedCM, nor the Fetch API, they are not accessible directly with via JavaScript.
Therefore, attackers with a successful XSS attack are never able to access any of these tokens directly.

The client metadata contains an list of allowed resource endpoints in `authorization.[0].resources`.
By strictly checking this allow-list, the web browser prevents sending an authorized request containing a valid access token to a malicious endpoint controlled by the attacker.

### CSRF Protection

To use the Fetch API for authorized resource requests, the `credential` option of the FedCM API must be set.
Since the `credential` object of FedCM API is individual per browser tab and origin, a `credential` object is not accessible from another tab or origin within the same browser session.
This mitigates cross-site-request-forgery (CSRF) attacks.
