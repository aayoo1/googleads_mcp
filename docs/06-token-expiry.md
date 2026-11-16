# 06. Access token expiry, and why one failed query does not mean the server is down

Goal: explain a failure that looks serious the first time you see it, but is expected behaviour.

## What you will see

After the server has been running for a while, typically an hour or more, a query can suddenly fail with an authentication or token error, even though nothing was changed and it was working a moment ago.

## Why it happens

When you sign in, Google issues a short lived access token, usually valid for about an hour, plus a longer lived refresh token. The access token is what actually gets sent with each Google Ads API call. Once it expires, the next call can fail until a fresh token has been obtained.

In practice the first query after expiry is the one most likely to fail. The token refresh kicks in, and the next attempt goes through.

## What to do

1. **Retry the same query once.** Most of the time it just works on the second try.
2. If it fails again, **fully quit and reopen Claude Desktop** (Cmd+Q). This makes mcp-remote reconnect and, if needed, prompts you to sign in again.
3. If you get errors like "Authorization session mismatch" or a sign in loop, clear the cached auth state and sign in fresh, same as in step 04:

   ```bash
   rm -rf ~/.mcp-auth
   ```

   Then fully quit and reopen Claude Desktop.
4. Only if all of that fails, check the terminal where the server is running. If the process has stopped, restart it as in step 03.

## The takeaway

A single failed query is not a sign the server is down or broken. Check whether the server terminal is still running before changing anything. If it is, the problem is almost always just an expired token, and a retry fixes it.

On the Cloud Run deployment (step 07), the server is always running, so steps 1 to 3 are all you will ever need.
