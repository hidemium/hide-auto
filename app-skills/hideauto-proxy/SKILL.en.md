---
name: "hideauto-proxy"
description: "Proxy on the Open & Connect node (manual proxy or HideProxy rotation: country/state/city fields) and connecting the user's HideProxy account (hideproxy_manage)."
---

# hideauto-proxy

## Proxy on `openConnect`

For the three built-in modes (`defaultProfile`, `newProfile`, `existedProfile`): set `openConnectProxyEnabled: true` and pick `openConnectProxyProvider`:

- `manual` → `openConnectProxyType` = `http` / `socks` and `openConnectProxyServer` as `host:port`.
- `hideproxy` (default) → rotating proxy from the HideProxy account connected in Settings → HideProxy account. If no account is connected the node fails at run time (see setup below).
  - `openConnectHideproxyCountry` is OPTIONAL and UPPERCASE (e.g. `"US"`). Empty = random exit country, and then `openConnectHideproxyState` / `openConnectHideproxyCity` MUST be empty too (a state without a country is dropped).
  - `openConnectHideproxyProxyId` picks which rotating proxy of the account to use — leave it EMPTY for auto (the one with most data left) unless the user names one.
  - Optionally narrow with `openConnectHideproxyState`, then `openConnectHideproxyCity` (a city needs a state). Values are machine codes in lower case (e.g. `"california"`); if unsure of a code, set the country only.
  - Do NOT set `openConnectProxyType` / `Server` / `Username` / `Password` when the provider is `hideproxy`.

## Connecting the HideProxy account

Settings section `hideproxy-integration`.

1. Start with `hideproxy_manage status` and follow its `nextStep` + `guide`, ONE step at a time: tell the user what to click, wait for them, then check `status` again.
2. Put every link in your reply as a markdown link and mention the "Open HideProxy page" button under the tool step (it signs them in when possible).
3. Only give HideProxy URLs exactly as `hideproxy_manage link` returns them — never type or guess one.
4. The API key is a secret: `credentials_manage` `request` target `hideproxy_api_key` — never ask the user to type it into the chat.
