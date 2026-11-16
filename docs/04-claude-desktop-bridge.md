# 04. Bridge the server into Claude Desktop

Goal: get Claude Desktop talking to the MCP server that is now running locally on http://localhost:8080/mcp from step 03.

## Why this needs a bridge

Claude Desktop's config file (claude_desktop_config.json) only supports stdio shaped entries under mcpServers, meaning a command it can launch as a subprocess. It does not support a raw HTTP url entry. The Google Ads MCP server runs as an HTTP server (FastMCP over Uvicorn), so something has to sit in between and translate stdio to HTTP.

That something is mcp-remote, an npm package that acts as a stdio to HTTP bridge. Claude Desktop launches mcp-remote as a subprocess, and mcp-remote forwards everything to the locally running HTTP server.

## Getting Node available

No admin access on this Mac, same constraint as step 03, so Node was installed inside the same conda environment used for Python.

Command used:

conda install -n mcp nodejs -y

Confirmed with node --version and npx --version once the environment was activated.

## The config entry

Edited claude_desktop_config.json directly (found at ~/Library/Application Support/Claude/claude_desktop_config.json on Mac) and added an mcpServers entry that points command at the absolute path to npx inside the conda environment, with args running mcp-remote against the local server URL.

This is the final working version, including the two fixes described further down. Replace `<you>` with your Mac username (check with `whoami`). If the file already has other servers in it, add `google-ads-mcp` inside the existing `mcpServers` block rather than pasting a second one.

```json
{
  "mcpServers": {
    "google-ads-mcp": {
      "command": "/Users/<you>/miniforge3/envs/mcp/bin/npx",
      "args": ["-y", "mcp-remote", "http://localhost:8080/mcp"],
      "env": {
        "PATH": "/Users/<you>/miniforge3/envs/mcp/bin:/usr/local/bin:/usr/bin:/bin"
      }
    }
  }
}
```

Notes on the entry:

- `command` must be the full path to npx inside the conda environment, not just `npx`.
- The `env` PATH line is what lets npx find node. Without it you get the first error below.
- Use `localhost`, not `127.0.0.1`. That is the second error below.
- No credentials go in this file. The server already has them from the `.env` file in step 03, and the server from step 03 must be running for this to connect.

Important: any edit to this file only takes effect after a full quit of Claude Desktop (Cmd+Q) and reopen. Closing the window is not enough.

## What did not work on the first try

First attempt: Claude Desktop reported Server disconnected. The log file at ~/Library/Logs/Claude/mcp-server-google-ads-mcp.log showed env: node: No such file or directory. Even though the command field used an absolute path to npx, Claude Desktop's subprocess PATH did not include the conda environment's bin directory, so npx could not find node internally. Fixed by adding an explicit env override inside the mcpServers config entry, setting PATH to prepend the conda environment's bin directory (the miniforge3/envs/mcp/bin path from step 03).

Second attempt, after the PATH fix: a new error appeared, Protected resource http://localhost:8080/mcp does not match expected http://127.0.0.1:8080/mcp (or origin). This came from following general MCP debugging advice online that recommended using 127.0.0.1 instead of localhost. That advice was wrong for this specific server, since the server itself declares its OAuth resource identity as localhost, not 127.0.0.1. Fixed by changing the mcp-remote target URL in the config back to http://localhost:8080/mcp so it matched what the server expected.

## The OAuth consent flow, and its own set of problems

Once the connection itself was stable, Claude Desktop triggers a two step consent flow on first use: a local consent screen served by the MCP server's own OAuth proxy (Allow Access), followed by Google's own account sign in and consent screen.

This flow failed in a few different ways before it finally succeeded:

This site can't be reached, localhost refused to connect, right after approving the local consent screen. The server log showed Long poll timeout reached appearing twice, followed by the client itself cancelling the connection before the Google consent screen could be completed. This was a timing race on the client side, not a config bug. Fixed by retrying and clicking through both consent screens, the local Allow Access and then Google's sign in, back to back as quickly as possible.

Authorization session mismatch, please try authenticating again. This was stale cached auth state left over from the earlier failed attempts. mcp-remote caches OAuth session state in ~/.mcp-auth with no built in clear command. Fixed with rm -rf ~/.mcp-auth, then a full quit and reopen of Claude Desktop before retrying.

CSRF double submit check failed, possible cross site consent forgery. This came directly from the server's own consent.py and was, again, a stale cookie or session artifact from the repeated failed attempts. Fixed the same way, clearing ~/.mcp-auth and doing a full quit and reopen, this time also using a private browser window for the retry to avoid any leftover localhost cookies interfering.

## Result

After clearing the cached auth state and completing both consent screens quickly in one pass, the browser showed Authorization successful, you may close this window and return to the CLI. Claude Desktop connected successfully. A first query at the manager account level (list of client accounts under the MCC) returned real data.

Client account level queries (an individual client account's own campaigns) still failed at this point with a different error. That is covered in the next doc, [05](05-login-customer-id.md).
