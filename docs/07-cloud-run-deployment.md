# 07. Deploy to Cloud Run for team access

## Why

The server was working great, but it only ran on my laptop through the mcp-remote bridge. That meant it stopped the moment I closed the terminal, and nobody else on the team could use it. This step moves the server to Google Cloud Run so it runs continuously and the whole team can connect to one shared URL, each signing in with their own Google account.

## What you end up with

Instead of a local process bridged into Claude Desktop, the server runs as a container on Cloud Run, reachable at a stable HTTPS URL of the form:

```
https://<cloud-run-service-name>.<region>.run.app/mcp
```

The real URL for this deployment is kept out of this repo and shared with the team through a private channel.

The service is public at the network layer (`--allow-unauthenticated` on Cloud Run), but that only means Cloud Run will route a request to the container. The server's own FastMCP OAuth proxy still requires every request to carry a valid access token from a completed Google sign in, so unauthenticated visitors just get a 401 from the app, not access to any data. This is the same pattern the official google-ads-mcp repo documents for shared deployments.

## Before you start

- Steps 01 to 05 done and working locally.
- The [gcloud CLI](https://cloud.google.com/sdk/docs/install) installed and signed in (`gcloud auth login`).
- Pick a region close to your team, for example `europe-west1`, and use it everywhere `<region>` appears below.

Placeholders used below: `<project-id>` is your Google Cloud project ID from step 01. `<region>`, `<artifact-repo>` and the secret names are names you choose. Pick them once and reuse them.

## Steps

### 1. Link a billing account

Most of the APIs below will not enable without one. Done manually in the console under Billing.

### 2. Set the project and enable the APIs

```bash
gcloud config set project <project-id>
gcloud services enable run.googleapis.com cloudbuild.googleapis.com \
  artifactregistry.googleapis.com secretmanager.googleapis.com firestore.googleapis.com
```

### 3. Get the server code and prepare the Dockerfile

Cloud Run needs a container image, so this step uses Google's own repo and its Dockerfile, pinned to the same version as the local setup:

```bash
git clone https://github.com/googleads/google-ads-mcp.git
cd google-ads-mcp
git checkout 2a336aac68b12758fc65bcb984d63fcc1dc14a72
```

Replace the contents of `Dockerfile` with this. It adds the Firestore extra the deployment needs, and runs the server as a non root user (see the security section at the end for why):

```dockerfile
FROM python:3.11-slim
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/
WORKDIR /app
COPY . .
RUN uv pip install --system ".[firestore]"
RUN useradd --create-home appuser
USER appuser
ENV HOME=/home/appuser
EXPOSE 8080
CMD ["google-ads-mcp"]
```

Then create a `.dockerignore` file in the same folder, so the build does not upload git history, tests or any stray credential file:

```
.git
.github
tests/
.venv/
__pycache__/
.env
.env.*
*.pem
*.key
credentials.json
*service-account*.json
```

If you use the latest server version instead of the pinned commit, also copy [`examples/tools_config.readonly.yaml`](../examples/tools_config.readonly.yaml) from this repo into the folder as `tools_config.yaml` to keep the deployment read only. Note that the latest Dockerfile has extra lines of its own, so edit it rather than replacing it.

### 4. Build and push the image

Create a repo in Artifact Registry, then build with Cloud Build (no local Docker needed). Run this from inside the `google-ads-mcp` folder:

```bash
gcloud artifacts repositories create <artifact-repo> --repository-format=docker --location=<region>
gcloud builds submit --tag <region>-docker.pkg.dev/<project-id>/<artifact-repo>/google-ads-mcp:latest .
```

### 5. Create a Firestore database

Cloud Run instances are temporary and can scale to several at once, so OAuth session state has to live somewhere shared instead of in one process's memory like it did on the laptop. Firestore is that shared store, accessed through the default compute service account, so no VPC connector is needed.

```bash
gcloud firestore databases create --location=<region>
```

Firestore does not delete expired sessions on its own with this server, so for a long running deployment it is worth clearing old entries now and then.

### 6. Store the secrets in Secret Manager

Secrets go into Secret Manager rather than plain environment variables. `read -rs` lets you paste each value without it showing on screen or landing in your shell history.

```bash
read -rs DEV_TOKEN
printf '%s' "$DEV_TOKEN" | gcloud secrets create <developer-token-secret-name> --data-file=-

openssl rand -base64 32 | gcloud secrets create <jwt-signing-key-secret-name> --data-file=-
```

The JWT signing key is a persistent key so issued tokens stay valid across container restarts and multiple instances, instead of a new key on every boot.

The OAuth client secret comes in step 9, because the client it belongs to needs the service URL first.

### 7. Give the Cloud Run service account access

```bash
PROJECT_NUMBER=$(gcloud projects describe <project-id> --format='value(projectNumber)')
SA="${PROJECT_NUMBER}-compute@developer.gserviceaccount.com"

for s in <developer-token-secret-name> <jwt-signing-key-secret-name>; do
  gcloud secrets add-iam-policy-binding $s --member="serviceAccount:$SA" --role=roles/secretmanager.secretAccessor
done

gcloud projects add-iam-policy-binding <project-id> --member="serviceAccount:$SA" --role=roles/datastore.user
```

### 8. First deploy and get the URL

The first deploy is only there to get the service URL assigned. It will not accept sign ins yet.

```bash
gcloud run deploy google-ads-mcp \
  --image <region>-docker.pkg.dev/<project-id>/<artifact-repo>/google-ads-mcp:latest \
  --platform managed --region <region> --allow-unauthenticated \
  --set-env-vars="GOOGLE_PROJECT_ID=<project-id>,GOOGLE_ADS_LOGIN_CUSTOMER_ID=<manager-customer-id>,GOOGLE_ADS_MCP_STORAGE_TYPE=firestore,FASTMCP_HOST=0.0.0.0" \
  --set-secrets="GOOGLE_ADS_DEVELOPER_TOKEN=<developer-token-secret-name>:latest,GOOGLE_ADS_MCP_JWT_SIGNING_KEY=<jwt-signing-key-secret-name>:latest"

gcloud run services describe google-ads-mcp --region <region> --format='value(status.url)'
```

The last command prints the service URL. Keep it, the next two steps need it.

### 9. Create a Web application OAuth client

In the Google Cloud console, go to Google Auth Platform, Clients, Create client:

- Application type: **Web application** (not Desktop app, see "The OAuth client type problem" below for why)
- Authorized redirect URI: `<service-url>/auth/callback`, using the URL from step 8

Store its secret, give the service account access to it, and wire both values into the service, together with the base URL:

```bash
read -rs WEB_SECRET
printf '%s' "$WEB_SECRET" | gcloud secrets create <oauth-client-secret-name> --data-file=-
gcloud secrets add-iam-policy-binding <oauth-client-secret-name> --member="serviceAccount:$SA" --role=roles/secretmanager.secretAccessor

gcloud run services update google-ads-mcp --region <region> \
  --update-env-vars="GOOGLE_ADS_MCP_OAUTH_CLIENT_ID=<web-client-id>,GOOGLE_ADS_MCP_BASE_URL=<service-url>" \
  --update-secrets="GOOGLE_ADS_MCP_OAUTH_CLIENT_SECRET=<oauth-client-secret-name>:latest"
```

`GOOGLE_ADS_MCP_BASE_URL` tells the server its own public address for OAuth discovery, which is why it can only be set after the first deploy.

### 10. Set a budget alert

In the console under Billing, Budgets and alerts, create a small budget (a few dollars) scoped to this project, with the standard 50, 80 and 100% notifications. It is a safety net now that the server is always on and reachable by the whole team rather than just one laptop.

## The OAuth client type problem (and the actual fix)

My first sign in attempt from Claude Desktop against the Cloud Run URL failed with **"Access blocked: This app's request is invalid."** The cause: I had reused the OAuth client from the local setup, which was registered as a **Desktop app** type. Google restricts Desktop type clients to loopback redirects (localhost or 127.0.0.1) only, by design, which is why that client type has no "Authorized redirect URIs" field in the console at all.

FastMCP's OAuth proxy, running on Cloud Run, needs a real HTTPS redirect back to itself (`<service-url>/auth/callback`) to complete the handshake with Google. A Desktop client can never satisfy that, so Google rejected the request before an account could even be picked.

The fix was the second, Web application client from step 9. The original Desktop client is untouched and still used for local setups. The Web application client is only used by the Cloud Run deployment.

## On the OAuth consent screen's broad permission wording

Signing in shows a warning that the app can "see, edit, create, and delete" Google Ads data. That is not a choice this project made. The Google Ads API only has a single OAuth scope (`https://www.googleapis.com/auth/adwords`), covering full read and write access, and Google offers no narrower read only scope for it. Every tool that talks to the Google Ads API, including the local setup, has to request the same scope.

What actually limits this server is its own code, not the scope. At the pinned version, `google-ads-mcp` has exactly three tools (`list_accessible_customers`, `get_resource_metadata`, `search`), all marked read only, with no create, update or delete operation anywhere. Newer versions add write tools, which is why steps 03 and 07 either pin the version or use the read only tools config.

## Verify it end to end

Queried the Cloud Run URL directly (no bridge) and confirmed it returns identical results to the local server: same accessible customer IDs, and the same campaign list and enabled and paused counts for a sample client account under the manager account.

## How a teammate connects

Share the service URL through a private channel, not in a repo. In Claude Desktop, each teammate goes to Settings, Connectors, Add custom connector, and pastes:

```
<service-url>/mcp
```

Other MCP clients take the same URL in their own config format.

They will be prompted to sign in with their own Google account, which needs Google Ads access to the manager account or a client account under it. No shared credentials, no local install, no separate server to keep running.

## Post deployment security review and hardening

After this deployment was verified end to end, I did a full review of the `google-ads-mcp` clone used for builds and this build log, looking for leaked secrets, cross user data exposure, and general hardening gaps, partly in preparation for making this repository public. Found no evidence of a breach: no credentials or secret values ever committed to git, no stray credential files on disk, and the code confirms each request uses the calling user's own Google OAuth token for every Ads API call (`ads_mcp/utils.py::_create_credentials`), not a shared service account. So teammates can only ever see accounts their own Google identity already has access to. At the pinned version the `search` tool only calls `search_stream` and no write operation exists, so it is genuinely read only regardless of the broad `adwords` scope.

Two hardening fixes came out of the review, both already included in step 3 above:

**Fix one, a `.dockerignore`.** There wasn't one at first, so `COPY . .` plus `gcloud builds submit .` was uploading the whole build folder (including `.git` history and test fixtures) into Cloud Build and baking it into the image. Nothing secret was ever in it, so nothing leaked, but it was unnecessary exposure. The `.dockerignore` excludes all of that, and defensively excludes any `.env`, key or credentials file in case one is ever dropped in the folder.

**Fix two, a non root user.** The original Dockerfile ran the server as root inside the container. Adding `useradd`, `USER appuser` and `ENV HOME=/home/appuser` changes that. Cloud Run's own sandboxing does most of the real isolation, but this limits the damage if the app or a dependency is ever compromised.

While fixing the Dockerfile I also caught that it had drifted back to plain `uv pip install --system .` without the Firestore extra. Without it, the server crashes at startup whenever `GOOGLE_ADS_MCP_STORAGE_TYPE=firestore` is set, so the next redeploy would have broken. The Dockerfile in step 3 has the extra, so check it is still there before any rebuild.

To redeploy after changing the Dockerfile:

```bash
gcloud builds submit --tag <region>-docker.pkg.dev/<project-id>/<artifact-repo>/google-ads-mcp:latest .
gcloud run deploy google-ads-mcp --image <region>-docker.pkg.dev/<project-id>/<artifact-repo>/google-ads-mcp:latest --region <region>
```

Then re-run the end to end check above to confirm nothing broke.

**Not checked in this pass:** the live Google Cloud project itself, meaning current IAM bindings on the secrets, whether the budget alert is still active, the actual Cloud Run environment and secret wiring, and Firestore access rules. This was a code and config review, not a live infrastructure audit.
