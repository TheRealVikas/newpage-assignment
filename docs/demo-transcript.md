# Local MCP demonstration transcript

The following is a deterministic local demonstration, not a claim of a Claude or Gemini host connection.

1. Start API after `npm run setup`; start MCP with `MERIDIAN_ENV=prod MERIDIAN_CALLER_USER_ID=user_E7691888F28227066DD2FB86 npm run dev:mcp`.
2. `usage_summary({period:"last_7_days"})` returns Clinical Operations platform totals: allowed own-group read.
3. `usage_summary({group:"finance",period:"last_7_days"})` returns `{ "error":"permission_denied", "reason":"Requested group is not visible to this caller", "required_scope":"usage:read" }`; no Finance data is returned.
4. `set_capability({group:"clinical-operations",capability:"web-search",enabled:false})` as that lead returns a `capabilities:write` denial in prod.
5. Restart as Chloe Dubois (`user_533B58E6F66C98ACB833527A`): the same capability update succeeds. Disable `usage-connector`; subsequent summary is denied with required scope `usage-connector`; enable it again to recover.
