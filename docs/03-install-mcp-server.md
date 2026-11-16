# 03. Install and run the MCP server locally

Goal: get the Google Ads MCP server running locally as a first proof of concept, using the developer token and OAuth client from the previous steps.

## Environment setup

The system Python on this Mac (3.9.6) was too old, the server needs Python 3.10 or newer, and there was no admin access to install a newer system Python the normal way. Installed Miniforge instead, a Conda distribution that installs entirely into the home directory and needs no sudo. Created a dedicated environment with Python 3.11, activated it, and installed pipx inside that environment.

Commands used (Apple Silicon Mac, for Intel use the `MacOSX-x86_64` installer instead):

```bash
curl -L -o Miniforge3.sh https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-MacOSX-arm64.sh
bash Miniforge3.sh -b -p $HOME/miniforge3
$HOME/miniforge3/bin/conda init zsh
```

`conda init zsh` is for the default Mac shell. Check yours with `echo $SHELL`, and use `conda init bash` if it says bash. Then close and reopen the terminal and run:

```bash
conda create -n mcp python=3.11 -y
conda activate mcp
pip install pipx
```

If you already have Python 3.10 or newer, you can skip Miniforge and install pipx the normal way.

## Credentials

The server reads its credentials from environment variables. To keep them out of your shell history and out of git, put them in a local `.env` file. Copy [`examples/env.example`](../examples/env.example) to `.env` in the folder you will run the server from, and fill in:

- `GOOGLE_PROJECT_ID`: the project ID from step 01
- `GOOGLE_ADS_DEVELOPER_TOKEN`: your developer token (see the README)
- `GOOGLE_ADS_MCP_OAUTH_CLIENT_ID` and `GOOGLE_ADS_MCP_OAUTH_CLIENT_SECRET`: the Desktop app client from step 02
- `GOOGLE_ADS_LOGIN_CUSTOMER_ID`: your manager account ID, digits only. You can leave this out for now, step 05 shows why you will need it.

Never commit the filled in `.env`.

## Running the server

With the mcp environment active, load the `.env` file and start the server, pinned to the version this build used:

```bash
set -a; source .env; set +a
pipx run --spec git+https://github.com/googleads/google-ads-mcp.git@2a336aac68b12758fc65bcb984d63fcc1dc14a72 google-ads-mcp
```

If you want the latest version instead, drop the `@2a336aa...` part. Newer versions include write tools, so to keep the server read only, copy [`examples/tools_config.readonly.yaml`](../examples/tools_config.readonly.yaml) into the same folder as `tools_config.yaml` before starting it.

## Result

The server started successfully as a FastMCP app, listening on http://0.0.0.0:8080/mcp using streamable-http transport. No credential errors, which confirms the developer token, project ID, and OAuth client are all correctly wired up. This proves the setup works, though right now it is only reachable from this machine. Leave this terminal open, the server stops when you close it.

## What did not work on the first try

pipx run initially failed with: Package google-ads-mcp requires a different Python: 3.9.6 not in >=3.10. That was because the default pipx install used the Mac system Python. There was also no admin access to fix this the normal way, a Homebrew install attempt failed with a sudo permission error. Miniforge solved both problems since it manages its own Python versions entirely inside the home directory.
