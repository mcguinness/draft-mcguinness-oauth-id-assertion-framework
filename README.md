<!-- regenerate: on (set to off if you edit this file) -->

# OAuth Identity Assertion Trust Framework

This is the working area for two related individual Internet-Drafts
that together define an OAuth authority-delegation framework for
identity-assertion trust evaluation.

1. **OAuth Identity Assertion Trust Framework** — the Authority
   Delegation Model (vocabulary, trust-evaluation categories,
   combination rule, lookup states), plus the Trust Policy
   document, Trust Method machinery, Subject Authority
   Determination, and OAuth grant-profile bindings.
   * [Editor's Copy](https://mcguinness.github.io/draft-mcguinness-oauth-id-assertion-framework/#go.draft-mcguinness-oauth-id-assertion-framework.html)
   * [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-id-assertion-framework)
   * [Individual Draft](https://datatracker.ietf.org/doc/html/draft-mcguinness-oauth-id-assertion-framework)

2. **OAuth Domain-Authorized Issuer Trust Method (DAI)** — a
   Standards-Track profile of the framework. Defines the
   `domain_authorized_issuer` Trust Method and the Issuer
   Authorization Policy wire format it consumes, published via
   DNS+HTTPS by the namespace owner.
   * [Editor's Copy](https://mcguinness.github.io/draft-mcguinness-oauth-id-assertion-framework/#go.draft-mcguinness-oauth-domain-authorized-issuer.html)
   * [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-domain-authorized-issuer)
   * [Individual Draft](https://datatracker.ietf.org/doc/html/draft-mcguinness-oauth-domain-authorized-issuer)

## How the documents relate

```
        OAuth Identity Assertion Trust Framework  (doc 1)
              (Authority Delegation Model +
               Trust Policy + Trust Methods)
                          |
                          | extended with a Trust Method by
                          v
          OAuth Domain-Authorized Issuer Trust Method  (doc 2)
                (subject_namespace_authorization)
```

A Resource Authorization Server publishes a Trust Policy declaring
which Trust Methods it requires; each Trust Method realizes one
trust-evaluation category. DAI defines the
`domain_authorized_issuer` Trust Method that lets a namespace owner
publish (via DNS+HTTPS) the set of Assertion Issuers authorized for
its namespace.

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
