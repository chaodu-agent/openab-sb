# Use case — Muse browses from a home residential IP, not its built-in browser

An agent like Muse has a built-in browser. That browser runs in the agent's own environment, so
every site sees that environment's network address, and it holds none of your logins. For some
tasks that is the wrong browser: the site wants a residential address, or the task needs a
session you already have at home.

`openab-sb` is what lets Muse use the browser on a computer at home instead — a Mac mini, for
instance — even though the two cannot reach each other:

```
Muse (cloud agent)  ──HTTPS + Bearer──►  openab-sb  ◄──WSS (home dials out)──  macmini
                                         switchboard                            oab-instance-mcp
                                                                                + Playwright MCP
                                                                                + Chromium
```

This page is a reference for that one use case, not a requirement. It builds on
[`README.md`](../README.md), [`SOUTHBOUND-CONTRACT.md`](SOUTHBOUND-CONTRACT.md) and
[`adr/multi-computer.md`](adr/multi-computer.md).

## The problem

|  | Muse's built-in browser | The browser on the home computer |
|---|---|---|
| Network address seen by sites | Muse's environment | Your home **residential IP** |
| Cookies and logged-in sessions | none | the home browser profile's |
| Reachable from Muse | yes | **no** — no inbound port, behind NAT |

The row that blocks the use case is the last one. The home computer can only dial out, and Muse
cannot enter the home network.

## Where sb comes in

Both sides can dial out, so neither needs to listen:

- the home computer dials the switchboard (southbound, `/vm/attach`) and serves MCP on that
  socket;
- Muse calls the switchboard (northbound, `POST /mcp/<computer>`) like any remote MCP server;
- the switchboard relays between them, and its per-caller allowlist decides which tools Muse may
  use — here, the browser tools only.

A tool call is then executed on the home computer, so the browser it drives leaves from the home
IP and carries that profile's cookies and sessions. That is the benefit, and also the risk — see
[Trust and risk](#trust-and-risk).

## Roles

Muse appears in two different roles in this repository. Do not mix them up:

| Role | What it is | Direction | Where it is configured |
|---|---|---|---|
| **Computer** (`muse-vm`) | A Muse Secure VM that is *controlled*, running `sb_daemon.py` | dials `/vm/attach` | `[[computer]]` |
| **Caller** (this page) | Muse the agent that *controls* a computer | calls `POST /mcp/<computer>` | `[[client]]` |

This page is about Muse as a **caller** and the Mac mini as the **computer**.

## Components

> Each cloud agent below is a separate caller with its own `[[client]]` token and allowlist. Muse is the worked example; **Grok Bot**, **OpenAI Dots** and **Claude Duet** are drawn the same way, but this page has not checked that each can add a remote MCP server with a custom `Authorization` header.

```mermaid
flowchart LR
    subgraph cloud["Cloud"]
        muse["Muse agent<br/>remote MCP client<br/>Bearer client token"]
        grok["Grok Bot<br/>remote MCP client<br/>own Bearer token"]
        dots["OpenAI Dots<br/>remote MCP client<br/>own Bearer token"]
        duet["Claude Duet<br/>remote MCP client<br/>own Bearer token"]
    end

    subgraph edge["Switchboard host (you run it)"]
        front["TLS front<br/>tailscale serve (PoC)<br/>or a public tunnel"]
        sb["openab-sb<br/>POST /mcp/macmini<br/>allowlist: browser_* only"]
        front --> sb
    end

    subgraph home["Home — outbound only"]
        subgraph mac["macmini (logged-in Aqua session)"]
            imcp["oab-instance-mcp<br/>--switchboard … --switchboard-profile desktop<br/>--upstream browser=…"]
            pw["Playwright MCP :8794<br/>loopback"]
            chromium["Chromium (headed)<br/>persistent profile"]
            imcp -- "loopback MCP" --> pw --> chromium
        end
    end

    muse -- "HTTPS, Bearer" --> front
    grok -- "HTTPS, Bearer" --> front
    dots -- "HTTPS, Bearer" --> front
    duet -- "HTTPS, Bearer" --> front
    imcp == "WSS /vm/attach<br/>dialled by macmini<br/>Bearer computer secret" ==> front
```

| Piece | Runs on | Job |
|---|---|---|
| Muse, Grok Bot, OpenAI Dots, Claude Duet | cloud | MCP clients, one `[[client]]` each. Needs to add a remote MCP server by URL with an `Authorization` header |
| TLS front | switchboard host | Terminates TLS. The PoC uses `tailscale serve` (tailnet only), so Muse must reach the tailnet; a public tunnel such as `cloudflared` is the alternative when it cannot |
| `openab-sb` | switchboard host | Authenticates, picks the computer from the path, applies the per-caller tool allowlist, relays |
| `oab-instance-mcp` | macmini | Dials the switchboard, serves tools on that socket under one profile |
| Playwright MCP | macmini | Loopback MCP server driving the browser (`poc/pw-mcp` in instance-mcp) |
| Chromium | macmini | The browser, with the persistent profile |

## Call flow

1. `oab-instance-mcp` dials `wss://<switchboard>/vm/attach` with the computer secret. The
   switchboard picks the slot from the secret and sends `initialize`.
2. Muse calls `POST https://<switchboard>/mcp/macmini` with its client token.
3. The switchboard resolves, in order, the token, then the computer, then the tool. A tool
   outside the allowlist is absent from `tools/list` and refused on `tools/call`.
4. Allowed calls are relayed down the computer's socket. `oab-instance-mcp` filters again by
   its own profile and forwards `browser_*` calls to the Playwright MCP upstream on loopback.
5. Chromium acts from the home network and replies travel back the same way.

While the Mac is offline, Muse gets the tool error "the VM is not connected" and still finds
the switchboard (`/healthz` is 200).

## Setup

### 1. Switchboard

```sh
openab-sb gen-secret     # once for the computer  → secret (macmini) + verifier
openab-sb gen-secret     # once for the caller    → token  (Muse)    + verifier
```

`openab-sb.toml`:

```toml
[[computer]]
name = "macmini"
secret_sha256 = "sha256:…"            # verifier of the macmini secret

[[client]]
name = "muse"
token_sha256 = "sha256:…"             # verifier of Muse's token
default_computer = "macmini"
computers = { macmini = [
  "sys_info",
  "browser_navigate", "browser_navigate_back", "browser_snapshot", "browser_find",
  "browser_click", "browser_type", "browser_fill_form", "browser_press_key",
  "browser_hover", "browser_select_option", "browser_wait_for", "browser_tabs",
  "browser_take_screenshot", "browser_console_messages", "browser_resize",
] }
max_inflight = 2
```

The list above is the instance-mcp `desktop` browser set
(`ToolProfile.desktopBrowserTools`) plus `sys_info`. Trim it further if the task allows.
Deliberately **absent**: `mouse`, `key`, `osascript`, `exec*`, `screenshot` of the whole
desktop. Then `openab-sb check -c openab-sb.toml` and `openab-sb serve -c openab-sb.toml`.
Put TLS in front — see [Exposing the switchboard](#exposing-the-switchboard).

### 2. Mac mini

Install the Playwright MCP upstream (`poc/pw-mcp/install.sh` in instance-mcp), then give
`oab-instance-mcp` the switchboard URL, the computer secret and the upstream:

```sh
oab-instance-mcp --token-file ~/.config/oab-instance-mcp/token \
  --upstream browser=http://127.0.0.1:8794/mcp \
  --switchboard wss://<switchboard>/vm/attach \
  --switchboard-secret-file ~/.config/oab-instance-mcp/switchboard.secret \
  --switchboard-profile desktop
```

Why `desktop`: `observe` serves `sys_info` and `screenshot` only, with no browser tools.
`owner` adds shell and arbitrary-JavaScript browser tools. `desktop` is the smallest profile
that carries the 15 browser tools. It *also* carries `mouse`, `key` and `osascript` on the
computer, which is why the switchboard allowlist above is what keeps Muse to the browser.

With the packaged installer, put the URL, secret and profile in
`~/.config/oab-instance-mcp/switchboard.{url,secret,profile}` and re-run the installer.

### 3. Muse

Add a remote MCP server with:

- URL `https://<switchboard>/mcp/macmini`
- header `Authorization: Bearer <Muse token>`

Muse must support Streamable HTTP MCP and a custom header. Whether and how a given Muse
deployment allows that is outside this repository.

### 4. Verify

```sh
# 1. computer attached and ready
curl -s -H "Authorization: Bearer $MUSE_TOKEN" https://<switchboard>/readyz/macmini -o /dev/null -w '%{http_code}\n'   # 200

# 2. what Muse can see: expect sys_info + the browser_* names, and no mouse/key/exec
curl -s -H "Authorization: Bearer $MUSE_TOKEN" -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' https://<switchboard>/mcp/macmini

# 3. a forbidden tool must read as unknown
#    tools/call name=mouse  → unknown tool
```

Then have Muse call `browser_navigate` to an IP-echo page and compare with your home
address, and open a site the profile is logged in to.

## Exposing the switchboard

`openab-sb` binds loopback and needs TLS in front.

- Muse **inside** your tailnet (**the PoC**): `tailscale serve --bg --https=8445 http://127.0.0.1:8796`, giving `https://<host>.<tailnet>.ts.net:8445/mcp/<computer>`. Tailnet only, no Funnel.
- Muse **outside** your tailnet: a public tunnel such as `cloudflared` (not used in the PoC). The bearer tokens are then the only gate, so keep the generated 256-bit
  tokens, rotate with `SIGHUP`, and keep the allowlist narrow.
- If Muse only reaches approved addresses, that address is the switchboard's front.

## Trust and risk

- **The browser profile is lent whole.** Muse can act as any account logged in to that
  Chromium profile. Use a **dedicated profile** logged in only to what the task needs, not
  your daily browser.
- **The allowlist is policy, not isolation.** It narrows what Muse may *call*; it does not
  narrow the browser. The computer is still an unrestricted desktop user, and `mouse`/`key`/
  `osascript` are hidden from Muse by the allowlist only. The computer's profile remains the
  ceiling and the caller gets the intersection.
- **Prompt injection.** Page text and screenshots go into Muse's context. A hostile page can
  try to steer an agent that holds your sessions. Treat browser output as untrusted.
- **Network reach.** The browser reaches whatever the Mac reaches, including localhost and the
  home LAN. Put the Mac on a network segment you are willing to expose, or use a dedicated
  machine.
- **The switchboard holds both sides' credentials.** Run it only on a host you trust. It
  stores only `sha256:` verifiers.
- **Public exposure.** With a public tunnel there is no tailnet identity check, only tokens.

## Limits of this reference

- **macOS only today for `--switchboard`.** In instance-mcp `main`, `--switchboard` exists in
  the Swift daemon. The Linux Rust hands node (`poc/reverse-attach-linux`) has no switchboard
  mode, so a Raspberry Pi or mini PC would need a daemon that follows
  [`SOUTHBOUND-CONTRACT.md`](SOUTHBOUND-CONTRACT.md) (see `examples/muse-daemon/sb_daemon.py`)
  and its own browser tooling.
- **No browser-only profile yet.** `desktop` still includes GUI tools; instance-mcp tracks a
  narrower browser tier in its #45. Until then the switchboard allowlist is the control.
- **One socket per computer.** A second daemon with the same secret evicts the first (`4002`).
- **Not verified here:** this page was assembled from the repository documents and source,
  not from an end-to-end run with Muse. Run the checks in [Verify](#4-verify) before relying
  on it.
- **Muse's built-in browser is described from the use case, not measured.** The comparison in
  [The problem](#the-problem) states the motivation; this page did not test what address or
  sessions Muse's own browser presents.
