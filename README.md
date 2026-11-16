# google-ads-mcp-build

Step by step build log for connecting the official Google Ads API MCP server to Claude, first on my own machine and then as a shared Cloud Run deployment for a whole team. Documented as I went, including the parts that did not work on the first try and how each one was fixed.

## Goal

Give Claude read access to Google Ads account data (campaigns, performance, accounts) through the official Google Ads API MCP server, and keep a build log anyone can follow.

## Who this is for

Anyone who manages Google Ads accounts, usually through a manager account (MCC), and wants to query them from Claude or another MCP client. You do not need to be a developer, but you will use the terminal and the Google Cloud console.

## Before you start

You will need:

- **A Google Ads manager account** with access to the accounts you want to query.
- **A Google Ads API developer token** (see below).
- **A Google Cloud account.** Steps 01 to 06 are free. Step 07 (Cloud Run) needs a billing account, but costs a few dollars a month at most for light team use.
- **A Google Workspace account** if you want the "Internal" OAuth consent screen used in step 02. On a personal Gmail account, use "External" and add yourself as a test user instead.
- **macOS on Apple Silicon** for the local commands as written. Other systems work too, but you will need the matching installer in step 03.
- **Claude Desktop** (or another MCP client).

### Getting a developer token

The developer token is what lets your Google Cloud project call the Google Ads API at all. It belongs to a manager account.

1. Sign in to your Google Ads **manager** account.
2. Go to Admin, then API Center, and fill in the application.
3. New tokens start at **Test Account Access**, which only works with Google Ads test accounts. To query real accounts, apply for **Basic Access** from the same page. Google reviews this manually, so allow some days.

This build reused a Basic Access token that already existed on the manager account, so there is no separate write up for the application itself. Google's guide: https://developers.google.com/google-ads/api/docs/api-policy/developer-token

## Version used

This build used the official server at commit [`2a336aa`](https://github.com/googleads/google-ads-mcp/commit/2a336aac68b12758fc65bcb984d63fcc1dc14a72) (August 2026). At that version the server only has three read only tools.

Newer versions of the server also include **write tools** (creating assets, uploading videos, managing labels), all switched on by default. They ask for confirmation before changing anything, but if you want read only access, either pin to the commit above or use the read only tools config in [`examples/tools_config.readonly.yaml`](examples/tools_config.readonly.yaml). Steps 03 and 07 show both.

## Build log

Each step has its own file in docs/. Follow them in order.

1. [Create a Google Cloud project](docs/01-google-cloud-project.md)
2. [Enable the Google Ads API and create OAuth credentials](docs/02-oauth-credentials.md)
3. [Install and run the MCP server locally](docs/03-install-mcp-server.md)
4. [Bridge the server into Claude Desktop (mcp-remote, OAuth consent flow)](docs/04-claude-desktop-bridge.md)
5. [Fix client account queries (login customer ID)](docs/05-login-customer-id.md)

## Operational notes

Things learned after the initial build that are worth knowing when using the server day to day.

6. [Access token expiry, and why one failed query does not mean the server is down](docs/06-token-expiry.md)

## Team deployment (Cloud Run)

7. [Deploy the server to Google Cloud Run](docs/07-cloud-run-deployment.md) so the whole team can connect to one shared, always on URL instead of each person running it locally. Covers billing, Secret Manager, Firestore for shared OAuth state, the deploy itself, and an OAuth client gotcha: Desktop type clients cannot be used behind a server side OAuth proxy, you need a Web application client.

## Result

Working end to end, both locally and as a shared Cloud Run deployment. The MCP server can query both the manager account and individual client accounts underneath it, verified with real queries against a sample client account's campaign data through both the local server and the Cloud Run service, with matching results.

## Security notes

- No credentials are stored in this repo. Client IDs, secrets and tokens live in the Google Cloud console, Secret Manager, or a local `.env` file that `.gitignore` keeps out of git.
- Each person signs in with their own Google account, so they only ever see the Google Ads accounts they already have access to.
- The live Cloud Run URL for this deployment is not in this repo.

## What is next

An expert level skill file for analysing this data (account audits, performance diagnostics, reporting conventions) and some read only monitoring agent ideas (weekly digest, dormant account watch, budget pacing check, new campaign tripwire) are being explored as a next phase, not yet built.

## Status

Core build complete and verified, also deployed for team wide access. Started 2026-08-17.

## Licence

MIT, see [LICENSE](LICENSE). The Google Ads MCP server itself is Google's, under its own licence.
