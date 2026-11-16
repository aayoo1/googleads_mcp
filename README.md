# google-ads-mcp-build

Step-by-step build log for connecting the official Google Ads API MCP server to my Google Ads account, documented as I go, including the parts that did not work on the first try.

## Goal

Give Claude read access to Google Ads account data (campaigns, performance, accounts) through the official Google Ads API MCP server, and keep a build log of the whole process.

## Reference

Official docs: https://developers.google.com/google-ads/api/docs/developer-toolkit/mcp-server

## Build log

Each step gets its own file in docs/, added as it is completed.

[x] 01 Create Google Cloud project
[x] 02 Enable Google Ads API and create OAuth credentials
[x] 03 Install and run the MCP server locally
[x] 04 Bridge the server into Claude Desktop (mcp-remote, OAuth consent flow)
[x] 05 Fix client account queries (login-customer-id) and verify end to end

Note on the developer token: this build reused an existing Basic Access developer token already provisioned on the manager account used for this project rather than applying for a new one, so there is no separate write-up for that application step.

## Operational notes

Things learned after the initial build that are worth knowing when actually using the server day to day, not part of the setup sequence itself.

[x] 06 Expected behavior: access token expiry and why a single failed query does not mean the server is down

## Team deployment (Cloud Run)

[x] 07 Deploy the server to Google Cloud Run so the whole team can connect to one shared, always-on URL instead of each person running it locally. Covers billing setup, Secret Manager, Firestore for shared OAuth state, the Cloud Run deploy itself, and an OAuth client type gotcha (Desktop-type clients cannot be used behind a server-side OAuth proxy, a Web application client is required instead). The live service URL is intentionally not committed to this repo; it is shared with the team through a private channel.

## Result

Working end to end, both locally and as a shared Cloud Run deployment. The MCP server can query both the manager account and individual client accounts underneath it, verified with real queries against a sample client account's campaign data through both the local server and the deployed Cloud Run service, with matching results.

The local server is also reachable from a Cowork session through the remote-devices bridge, no separate setup needed.

## What is next

An expert-level skill file for analyzing this data (account audits, performance diagnostics, reporting conventions) and some read-only monitoring agent ideas (weekly digest, dormant account watch, budget pacing check, new campaign tripwire) are being explored as a next phase, not yet built.

## Status

Core build complete and verified, now also deployed for team-wide access. Started 2026-08-17.
