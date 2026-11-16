# 07 Deploy to Cloud Run for team access

## Why

The server was working great, but it only ran on my laptop through the mcp-remote bridge. That meant it stopped the moment I closed the terminal, and nobody else on the team could use it. This step moves the server to Google Cloud Run so it runs continuously and the whole team can connect to one shared URL, each authenticating with their own Google sign-in.

## What changed

Instead of a local process bridged into Claude Desktop, the server now runs as a container on Cloud Run, reachable at a stable HTTPS URL of the form:

```
https://<cloud-run-service-name>.<region>.run.app/mcp
```

(The real URL for this deployment is kept out of this repo and shared with the team through a private channel instead, not committed to git history.)

The service is public at the network layer (--allow-unauthenticated on Cloud Run), but that only means Cloud Run itself will route a request to the container. The server's own FastMCP OAuth proxy still requires every request to carry a valid access token from a completed Google sign-in, so unauthenticated visitors just get a 401 from the app, not access to any data. This is the same pattern the official google-ads-mcp repo documents for shared deployments.

## Steps taken

- **Link a billing account to the project.** Most GCP APIs will not enable without one. Done manually in the console (entering payment details is not something an assistant should do on someone else's behalf).
- - **Enable required APIs**: Cloud Run, Cloud Build, Artifact Registry, Secret Manager, Firestore.
  - - **Create an Artifact Registry repo** for the container image:
    - ```
gcloud artifacts repositories create <artifact-repo> --repository-format=docker --location=<region>
```
- **Build and push the image with Cloud Build** (no local Docker needed). The Dockerfile was changed from `uv pip install --system .` to `uv pip install --system .[firestore]` to include the Firestore storage extra, then:
  - ```
gcloud builds submit --tag <region>-docker.pkg.dev/PROJECT_ID/<artifact-repo>/google-ads-mcp:latest .
```
- **Create a Firestore database** (native mode) in the project. Cloud Run instances are ephemeral and can scale to multiple instances, so OAuth session state has to live somewhere shared instead of in the single local process's memory the way it did on the laptop. Firestore is that shared store, accessed via the default compute service account through Application Default Credentials (no VPC connector needed).
- - **Store secrets in Secret Manager** rather than plain environment variables:
  - `<developer-token-secret-name>`
  - `<oauth-client-secret-name>`

  - `<jwt-signing-key-secret-name>` (a persistent signing key so issued tokens stay valid across container restarts and multiple instances, unlike a key regenerated per boot)

  Granted the Cloud Run service account `roles/secretmanager.secretAccessor` and `roles/datastore.user` on each.

- - **Deploy to Cloud Run**, wiring the secrets in via `--set-secrets` and the non-sensitive config via `--set-env-vars` (project id, login customer id, storage type, host):
  - ```
gcloud run deploy google-ads-mcp \
--image <region>-docker.pkg.dev/PROJECT_ID/<artifact-repo>/google-ads-mcp:latest \
--platform managed --region <region> --allow-unauthenticated \
--set-env-vars="GOOGLE_PROJECT_ID=...,GOOGLE_ADS_LOGIN_CUSTOMER_ID=...,GOOGLE_ADS_MCP_OAUTH_CLIENT_ID=...,GOOGLE_ADS_MCP_STORAGE_TYPE=firestore,FASTMCP_HOST=0.0.0.0" \
--set-secrets="GOOGLE_ADS_DEVELOPER_TOKEN=<developer-token-secret-name>:latest,GOOGLE_ADS_MCP_OAUTH_CLIENT_SECRET=<oauth-client-secret-name>:latest,GOOGLE_ADS_MCP_JWT_SIGNING_KEY=<jwt-signing-key-secret-name>:latest"
```
- **Set `GOOGLE_ADS_MCP_BASE_URL`** to the real Cloud Run URL once it was assigned (this env var declares the server's own identity for OAuth discovery, and is not known until after the first deploy).

- - **Set a Cloud Run spend cap budget** (a few dollars) scoped to this project and the Cloud Run service, with the standard 50/80/100% notifications, as a safety net given the server is now always-on and reachable by the whole team rather than just one laptop.

## The OAuth client type problem (and the actual fix)

The first sign-in attempt from Claude Desktop against the Cloud Run URL failed with **"Access blocked: This app's request is invalid."** The cause: the OAuth client reused from local setup was registered as a **Desktop app** type. Google restricts Desktop-type clients to loopback redirects (localhost/127.0.0.1) only, by design, which is why that client type has no "Authorized redirect URIs" field in the console at all.

FastMCP's OAuth proxy, running server-side on Cloud Run, needs a real HTTPS redirect back to itself (`https://<service-url>/auth/callback`) to complete the handshake with Google. A Desktop client can never satisfy that, so Google rejected the request outright before an account could even be picked.

Fix: created a **second** OAuth client, type **Web application**, with `https://<cloud-run-service-name>.<region>.run.app/auth/callback` added as an Authorized redirect URI. Pushed the new client secret into the `<oauth-client-secret-name>` Secret Manager secret and updated the `GOOGLE_ADS_MCP_OAUTH_CLIENT_ID` env var to the new client's ID, then forced a new revision so Cloud Run picked both up:

```
printf '%s' 'NEW_SECRET' | gcloud secrets versions add <oauth-client-secret-name> --data-file=-
gcloud run services update google-ads-mcp --region <region> \
--update-env-vars=GOOGLE_ADS_MCP_OAUTH_CLIENT_ID=NEW_CLIENT_ID
```

The original Desktop client is untouched and still used for local/laptop setups; the Web application client is only used by the Cloud Run deployment.

## On the OAuth consent screen's broad permission wording

Signing in shows a warning that the app can "see, edit, create, and delete" Google Ads data. That is not a choice this project made: the Google Ads API only exposes a single OAuth scope (`https://www.googleapis.com/auth/adwords`), covering full read/write access; there is no narrower read-only scope Google offers for this API. Every tool that talks to the Google Ads API, including the local laptop setup, has to request the same scope.

What actually limits this server is its own code, not the scope: `google-ads-mcp` implements exactly three tools (`list_accessible_customers`, `get_resource_metadata`, `search`), all explicitly annotated read-only, with no create/update/delete/mutate operation implemented anywhere. The broad-looking consent screen does not reflect what this deployment can actually do.

## Verified end to end

Queried the Cloud Run URL directly (no bridge) after the OAuth client fix and confirmed it returns identical results to the local server: same accessible customer IDs, and the same campaign list and enabled/paused counts for a sample client account under the manager account.

## How a teammate connects

In their MCP client (e.g. Claude Desktop: Settings -> Connectors -> Add custom connector), add the shared server URL (get this from the team's private credential store, not from this repo):

```json
{
"mcpServers": {
"google-ads-mcp": {
"httpUrl": "https://<cloud-run-service-name>.<region>.run.app/mcp"
}
}
}
```

They will be prompted to sign in with their own Google account (needs Google Ads access to the manager account or a client account under it). No shared credentials, no local install, no separate server to keep running.

## Post-deployment security review and hardening

After this deployment was verified end to end, did a full review of the `google-ads-mcp` checkout and this build log looking for leaked secrets, cross-user data exposure, and general hardening gaps, partly in preparation for eventually making this repository public. Found no evidence of a breach: no credentials or secret values ever committed to git, no stray credential files on disk, and the code confirms each request uses the calling user's own Google OAuth token for every Ads API call (`ads_mcp/utils.py::_create_credentials`), not a shared service account, so teammates can only ever see accounts their own Google identity already has access to. The `search` tool only calls `search_stream` and no mutate operation is implemented anywhere, so it is genuinely read-only regardless of the broad `adwords` OAuth scope Google requires.

Applied two hardening fixes to the `google-ads-mcp` checkout used for builds.

**Fix one, added a `.dockerignore`.** There wasn't one, so `COPY . .` plus `gcloud builds submit .` was uploading the whole build context (including `.git` history and test fixtures) into Cloud Build and baking it into the image layer. No secrets were ever in the repo, so nothing had actually leaked, but it was unnecessary exposure surface. Now excludes `.git`, `.github`, `tests/`, dev tooling, and defensively excludes any `.env`/`*.pem`/`*.key`/`credentials.json`/`*service-account*.json` in case one is ever dropped in the project root.

**Fix two, the Dockerfile now runs as a non-root user.** Added a `useradd` step and `USER appuser` after the install step, plus `ENV HOME=/home/appuser`, so the process doesn't run as root inside the container. Cloud Run's own sandboxing does most of the real isolation, but this limits blast radius if the app or a dependency is ever compromised.

While fixing the Dockerfile, also caught that it had drifted from what this doc describes above: the install line had reverted to plain `uv pip install --system .` with no firestore extra, missing the `.[firestore]` change described in the Steps taken section. Restored it. This matters because without it, `create_client_storage()` raises at import time whenever `GOOGLE_ADS_MCP_STORAGE_TYPE=firestore` is set (which it is for this deployment); a rebuild from the drifted file would have crashed the server at startup rather than silently degrading security, but it would have broken the next redeploy.

Redeploy needed to pick these up:

```
gcloud builds submit --tag <region>-docker.pkg.dev/PROJECT_ID/<artifact-repo>/google-ads-mcp:latest .
gcloud run deploy google-ads-mcp --image <region>-docker.pkg.dev/PROJECT_ID/<artifact-repo>/google-ads-mcp:latest --region <region>
```

After redeploying, re-run the same end-to-end check as before (query the Cloud Run URL directly and confirm it still returns the same accessible customers and campaign counts) to confirm the non-root user and dockerignore changes didn't break anything at runtime.

**Not checked in this pass:** the live GCP project state, meaning current IAM bindings on the Secret Manager secrets, whether the spend-cap budget is still active, actual Cloud Run env/secret wiring, and Firestore access rules. This was a static code and config review, not a live infrastructure audit.
