# Bug Explanation

## What was the bug?

Inside `src/httpClient.ts`, the condition that decides whether to refresh the OAuth2
token was:

```ts
if (!this.oauth2Token || (this.oauth2Token instanceof OAuth2Token && this.oauth2Token.expired))
```

When `oauth2Token` was set to a plain object (e.g. `{ accessToken: "stale", expiresAt: 0 }`),
no refresh happened and no `Authorization` header was added to the request.

## Why did it happen?

The condition has two branches joined by `||`:

1. `!this.oauth2Token` — only catches `null` / `undefined` / falsy values.
   A plain object is truthy, so this branch is `false`.
2. `this.oauth2Token instanceof OAuth2Token && ...` — only applies when the token
   IS already an `OAuth2Token` instance. A plain object fails `instanceof`, so this
   branch is also `false`.

Result: the refresh is skipped entirely. The second guard (line 28) uses the same
`instanceof` check, so no header is written either.

## Why does the fix solve it?

The replacement condition is:

```ts
if (!(this.oauth2Token instanceof OAuth2Token) || this.oauth2Token.expired)
```

This refreshes unless the token is *already* a valid, non-expired `OAuth2Token`
instance. `null`, plain objects, and any other non-instance value all fail
`instanceof`, triggering a refresh and replacing the bad value with a real
`OAuth2Token`.

## Edge case the tests still don't cover

A token whose `expiresAt` is *exactly* equal to the current second (`now >= expiresAt`
returns `true`). This is a boundary condition: a token that expires at precisely
"now" is treated as expired, which is correct, but no test verifies this fence-post
behaviour. A clock skew scenario (server time ahead of client) could also cause a
valid token to appear expired and be unnecessarily refreshed.
