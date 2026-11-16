# 03. Install and run the MCP server locally

Goal: get the Google Ads MCP server running locally as a first proof of concept, using the developer token and OAuth client from the previous steps.

## Environment setup

The system Python on this Mac (3.9.6) was too old, the server needs Python 3.10 or newer, and there was no admin access to install a newer system Python the normal way. Installed Miniforge instead, a Conda distribution that installs entirely into the home directory and needs no sudo. Created a dedicated environment with Python 3.11, activated it, and installed pipx inside that environment.

Commands used:

curl -L -o Miniforge3.sh https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-MacOSX-arm64.sh
bash Miniforge3.sh -b -p $HOME/miniforge3
conda init bash  (if your terminal defaults to zsh instead of bash, use conda init zsh instead; check with echo $SHELL)
conda create -n mcp python=3.11 -y
conda activate mcp
pip install pipx

## Running the server

With the mcp environment active, set four environment variables using the values from steps 01 and 02: GOOGLE_PROJECT_ID, GOOGLE_ADS_DEVELOPER_TOKEN, GOOGLE_ADS_MCP_OAUTH_CLIENT_ID, GOOGLE_ADS_MCP_OAUTH_CLIENT_SECRET. Then ran:

pipx run --spec git+https://github.com/googleads/google-ads-mcp.git google-ads-mcp

## Result

The server started successfully as a FastMCP app, listening on http://0.0.0.0:8080/mcp using streamable-http transport. No credential errors, which confirms the developer token, project ID, and OAuth client are all correctly wired up. This proves the setup works end to end, though right now it is only reachable from this machine, not yet exposed for use from anywhere else.

## What did not work on the first try

pipx run initially failed with: Package google-ads-mcp requires a different Python: 3.9.6 not in >=3.10. That was because the default pipx install used the Mac system Python. There was also no admin access to fix this the normal way, a Homebrew install attempt failed with a sudo permission error. Miniforge solved both problems since it manages its own Python versions entirely inside the home directory.
