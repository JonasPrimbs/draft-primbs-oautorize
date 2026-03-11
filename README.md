# Draft: OAuth AUTOrize

A draft how OAuth 2.0 can be implemented securely without developers need to care about OAuth by automatically request authorization (AUTOrize) based on the REST API.

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

TODO

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
