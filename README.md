<!-- regenerate: off (set to off if you edit this file) -->

# OAuth Identity Assertion Trust Framework

This is the working area for three related individual Internet-Drafts
about one question: when a Resource Authorization Server receives an
Identity Assertion JWT Authorization Grant (ID-JAG) for
`alice@acme.example`, should it trust the issuer for that subject?

Two parties publish policy, and they are easy to confuse:

- The **Resource Authorization Server** (the server that receives the
  grant) publishes a **Trust Policy**: the Trust Methods an issuer
  must satisfy, such as membership in a federation and authorization
  by the subject's domain. Its full name is "Identity Assertion Issuer
  Trust Policy", but no issuer publishes it.
- The **Subject Authority** (the owner of `acme.example`) publishes an
  **Issuer Authorization Policy**: the issuers it authorizes to assert
  identities in its domain. DAI defines it.

1. **OAuth Identity Assertion Trust Framework**: the Trust Policy
   document, Trust Method categories and registries, Subject
   Authority Determination, and the token-endpoint procedure, bound
   to ID-JAG.
   * [Editor's Copy](https://mcguinness.github.io/draft-mcguinness-oauth-id-assertion-framework/#go.draft-mcguinness-oauth-id-assertion-framework.html)
   * [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-id-assertion-framework)
   * [Individual Draft](https://datatracker.ietf.org/doc/html/draft-mcguinness-oauth-id-assertion-framework)

2. **OAuth Domain-Authorized Issuer Trust Method (DAI)**: the
   `domain_authorized_issuer` Trust Method and the Issuer
   Authorization Policy that a Subject Authority publishes through
   DNS and HTTPS.
   * [Editor's Copy](https://mcguinness.github.io/draft-mcguinness-oauth-id-assertion-framework/#go.draft-mcguinness-oauth-domain-authorized-issuer.html)
   * [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-domain-authorized-issuer)
   * [Individual Draft](https://datatracker.ietf.org/doc/html/draft-mcguinness-oauth-domain-authorized-issuer)

3. **OpenID Federation Trust Methods for Identity Assertion Grants
   (FED)**: `openid_federation`, which authenticates the issuer
   through an OpenID Federation trust chain, and
   `openid_federation_client`, which recognizes the client presenting
   the grant, with the `client_recognition` category and the
   `client_trust_methods` Trust Policy member.
   * [Editor's Copy](https://mcguinness.github.io/draft-mcguinness-oauth-id-assertion-framework/#go.draft-mcguinness-oauth-id-assertion-federation.html)
   * [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-id-assertion-federation)
   * [Individual Draft](https://datatracker.ietf.org/doc/html/draft-mcguinness-oauth-id-assertion-federation)

## How the documents relate

```
        OAuth Identity Assertion Trust Framework  (doc 1)
              (Trust Policy, Trust Methods,
               token-endpoint procedure)
                 |                          |
                 | extended by              | extended by
                 v                          v
  Domain-Authorized Issuer (doc 2)    OpenID Federation Trust
  (subject_namespace_authorization)   Methods (doc 3)
                                      (issuer_authentication,
                                       client_recognition)
```

A Resource Authorization Server that uses only DAI does not need FED.

## Contributing

See the
[guidelines for contributions](https://github.com/mcguinness/draft-mcguinness-oauth-id-assertion-framework/blob/main/CONTRIBUTING.md).

The contributing file also has tips on how to make contributions, if you
don't already know how to do that.

## Command Line Usage

Formatted text and HTML versions of the drafts can be built using `make`.

```sh
$ make
```

Command line usage requires that you have the necessary software installed.  See
[the instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md).
