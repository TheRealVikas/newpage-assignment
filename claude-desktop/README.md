# Claude Desktop local configuration

Copy the server entry from `claude_desktop_config.example.json` into Claude Desktop's MCP configuration, replace `cwd`, run `npm run setup`, and restart Claude Desktop. This repository provides a local stdio server; no claim is made that Claude Desktop was executed in this development environment. Identity is fixed at server start by `MERIDIAN_CALLER_USER_ID` and is resolved against the fixture directory.
