# OAuth-AutoSecure-Draft

A draft how OAuth 2.0 can be implemented securely without developers need to care about OAuth.

## Abstract

OAuth AutoSecure is a proposed OAuth-based Framework that targets three goals.
First, increasing web-based API security by leveraging transaction tokens for frontend-to-backend communication.
Second, automating security-related concepts like authorization flows, token exchanges and authorizing requests, so that client developers no more need to care about authorization anymore.
Third, finally protecting OAuth against token stealing via XSS attacks by implementing the framework into web browsers.

## Introduction

OAuth 2.0 in the year 2026 has many useful extensions to improve security.
However, it is hard for developers to learn all of them and implement them correctly by following all best practices.
Even with the use of AI, it is hard to maintain and secure large code bases.

Modern security concepts like transaction tokens further improve system security by utilizing transaction tokens in backend services.
Using transaction tokens on the client comes with the advantage that a compromized resource server cannot use a received access token to compromize data on other resource servers.
This improves security but heavily increases complexity of the token handling implementation at the client.

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
