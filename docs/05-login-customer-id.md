# 05. Fix client account queries (login customer ID)

Goal: get queries working against individual client accounts, not just the manager account.

## The problem

After step 04, queries at the manager account level worked fine, for example listing the client accounts under the MCC. But as soon as a query targeted one client account's own data, like its campaigns, it failed with a permission error along these lines:

```
USER_PERMISSION_DENIED: User doesn't have permission to access customer.
Note: If you're accessing a client customer, the manager's customer id must be set in the 'login-customer-id' header.
```

## Why it happens

The Google account used to sign in does not have direct access to each client account. It has access to the manager account, and the manager account has access to the clients. The Google Ads API needs to be told which manager account you are acting through, and that is what the `login-customer-id` header does. Without it, the API checks your direct access to the client account, finds none, and refuses.

If you have been added directly to every client account yourself, you will not hit this. Most agency setups go through a manager account, so most people will.

## The fix

The server sends that header for you if you set the `GOOGLE_ADS_LOGIN_CUSTOMER_ID` environment variable to the manager account's customer ID.

1. Find the manager account's customer ID. It is the number shown at the top of the Google Ads interface when you are in the manager account, in the format 123-456-7890.
2. Add it to your `.env` file from step 03, **digits only, no dashes**. The version of the server this build used passes the value through as is, and the API rejects it with dashes.

   ```
   GOOGLE_ADS_LOGIN_CUSTOMER_ID=1234567890
   ```

3. Stop the server (Ctrl+C in its terminal), then load the `.env` file again and restart it with the same commands as in step 03.
4. Fully quit and reopen Claude Desktop (Cmd+Q).

You do not need to change claude_desktop_config.json. The header is added by the server, not by Claude Desktop or mcp-remote.

## Result

Client account queries worked straight away. Verified with a real query against a sample client account's campaign data, which returned its campaign list with the correct enabled and paused statuses. Manager level queries kept working as before.

## Note for newer server versions

Newer versions of the server also accept a `login_customer_id` per query, and strip dashes automatically. The environment variable still works as a default, so the fix above is the same.

Google's explanation of the header: https://developers.google.com/google-ads/api/docs/concepts/call-structure#cid
