# Exercise 3

Note: could also be implemented with email instead of username

```plaintext
concept PersonalAccessToken [User, Scope]
purpose
  enable access to a user using a scope-limited secret

principle
  a user creates one or more tokens;
  each of those token is generated along with a secret string and a set of scopes that limit its permissions;
  the secret is only shown once at creation and must be saved by the user;
  clients can authenticate by presenting their username and token secret;
  the user may revoke a token, making it invalid.

state
  a set of Tokens with
    an owner User
    a secret String
    a set of Scope
    an active Flag

actions
  createToken (owner: User, scopes: set of Scope): (token: Token, secret: String)
    requires owner exists
    effects create a new active token for this owner with these given scopes
            generate a new, unique secret

  revokeToken (owner: User, token: Token)
    requires token exists, token.owner = owner, token.active = true
    effects set token.active := false

  authenticateWithToken (username: String, secret: String, scope: Scope): (user: User, token: Token)
    requires an active token exists with token.owner.username = username,
             token.secret = secret,
             and scope in token.scopes
    effects none (authentication successful)
```

## How they differ:

- Firstly, in personal access tokens, one user may have multiple tokens associated with them, whereas a user only has one password.
- Also, passwords grant full access to all resources, while tokens are scope-limited and restricted to certain actions
- Moreover, tokens can be revoked individually without impacting the user’s password, but a user must always have a password.
- Tokens are only shown once at creation and must be regenerated if lost, while passwords are usually user-chosen and can be reset or re-entered.
- Finally, tokens are intended mainly for APIs and programmatic access, whereas passwords are the general authentication method for account sign-in.

## Docs Improval

Github’s current statement, “Treat your access tokens like passwords,” is not entirely accurate. What they mean is that tokens should be protected with the same level of care, but this phrasing fails to explain the conceptual differences between the two. Instead, a clearer explanation would explicitly distinguish the two: passwords are single, full-access secrets used for interactive sign-in, while tokens are multiple, scope-limited, revocable credentials intended for programmatic access. This change on Github's website would make the documentation better for less technical/experiences audiences and make sure the difference is more obvious.
