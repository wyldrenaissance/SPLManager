# SPLManager: from wireframe to working product

How the clickable UI in `app/` becomes the real management and install app for the renderer. Written 2026-11-05 as a plan to argue with, not a spec.

## 1. What we have

- One file, `app/index.html`. Every page is a render function over one object, `DATA`. Nothing talks to a network.
- `DATA` already has the right shape: spaces, zones, computers (with network, audio, processes), speaker maps, account, settings, integrations, events.
- The same UI is meant to run in two places: on a renderer Mac itself ("This computer" band) and remotely from any browser.

The one-object design is the asset. The integration job is to feed `DATA` from real sources and send user actions somewhere real, without redrawing the UI.

## 2. Target shape

Four pieces, three of which do not exist yet.

| Piece | Lives where | Owns |
|---|---|---|
| Renderer | each Mac (exists) | playback, REST server :7378, SPLSensor OSC :9002 |
| SPLM Agent | each Mac, new | that Mac's config, process control, logs, install steps, a WebSocket for live status |
| Site store | cloud Postgres (Supabase fits), new | spaces, zones, speaker maps, users, licenses, integrations |
| SPLM app | each Mac, new | a thin native shell (menu bar app) that bundles the agent and serves this UI |

The agent is the key addition. A web page cannot rename a Mac, install a daemon, enumerate Core Audio devices or ask for macOS permissions. A small local service can, and it also gives the UI one address per computer for everything, including things the renderer's own REST API will never do.

Remote access is the same UI loaded from any agent on the network (or hosted from the cloud later). The UI decides whether it is local by asking the agent "are you me?", which replaces today's `?remote` switch.

## 3. Who owns what

- **Computer-level config** (name, interface, audio device, channels, AES67, ports, process switches) is owned by the agent on that Mac and mirrored to the site store so remote views can show it.
- **Site-level objects** (spaces, zones, speaker maps, scenes, users, integrations) are owned by the site store. Every agent keeps a local copy so a site keeps working with the cloud unreachable; the copy is read-only until the cloud is back, except for playback.
- **Live status** (rendering, level, load, PTP, dropouts) never touches the database. Each agent pushes it over a WebSocket; the UI merges it into `DATA`.

This matches the rules already in the wireframe: Spaces own computers, zones only use computers in their Space, speaker maps reference computers by UUID and channel.

## 4. The API contract

Make the agent's API return the same shapes `DATA` holds today, so the integration is a swap, not a rewrite.

```
GET  /api/me                     -> { computerId | null }        is this the serving Mac
GET  /api/site                   -> { spaces, zones, speakermaps, integrations, defaults, account }
GET  /api/computers              -> { oc01: {...}, ... }         merged from every agent
WS   /api/events                 -> { type:'status'|'event'|'log', ... }

POST /api/zones/:id/play|stop    PUT /api/zones/:id { volume, content, boot, notes }
PUT  /api/computers/:id          { name, iface, device, channels, aes67, procs, ... }
POST /api/computers/:id/restart
PUT  /api/speakermaps/:id        POST /api/speakermaps (upload)
POST /api/install/step/:n        { ...fields }                   see section 5
```

In the UI, `DATA` becomes a store with two adapters: `mock` (today's object, kept forever as the demo and for design work, reachable at `?demo`) and `live` (fetch plus WebSocket). Every `data-*` click handler that mutates `DATA` today calls the adapter instead and re-renders when the reply arrives. Optimistic updates for sliders and toggles, confirmed by the next status push.

## 5. Install mode

The wireframe now has the installer at `#install`. What each step needs behind it on a real Mac:

| Step | Real work | Needs admin |
|---|---|---|
| 1 This computer | `scutil --set ComputerName`; list interfaces via SystemConfiguration; Energy = `pmset` sleep/autorestart; Start at login = `SMAppService.loginItem`; Local Network = trigger the TCC prompt once | name and energy: yes |
| 2 Install renderer | download the versioned bundle from the cloud, verify signature, place in `/Library/Application Support/SPL/`, write `/Library/LaunchDaemons/com.spl.renderer.plist`, `launchctl bootstrap` | yes |
| 3 Processes | write renderer config (REST on/off and port, SPLSensor on/off and port), restart, probe both ports | no |
| 4 Audio | enumerate Core Audio devices and output counts; for AES67 write stream name, interface, multicast IP, channel count into the renderer config; test tone through the renderer | no |
| 5 Wrap up | sign in or pick the site, register the computer UUID in a Space in the site store, apply site defaults (multicast, ports), hand off to Speakermaps | no |

Admin work goes through one signed privileged helper (`SMAppService.daemon`) that the shell app registers on first launch. The agent asks the helper; the web UI never sees privilege. This is the same pattern as Docker Desktop, Tailscale and most Mac tools with a daemon.

Rules the wireframe already follows and the real app should keep:
- Install mode opens by itself when the app runs on a Mac with no renderer registered. Skip for now is always visible and returns to the normal UI, with "Install renderer" left on the This computer band and in Settings.
- The installer can be rerun any time from the computer's page or Settings. Steps are prefilled from the current config, so rerunning is a repair, not a reset.
- On a non-Mac the installer is hidden. Remote installs are out of scope: someone is at the Mac.

Distribution: a notarized `.dmg` with the shell app. The app contains the agent, the helper and this UI. The renderer itself is downloaded in step 2 so renderer releases do not require a new app.

## 6. Order of work

1. **Contract first.** Write the JSON schema for `/api/site` and `/api/computers` from today's `DATA`. One afternoon. Everything else builds on it.
2. **Agent, read-only.** Serve the UI, answer `/api/me`, proxy renderer status over the WebSocket. The UI gets a `live` adapter and shows real status with every control still mocked. First moment of truth: one Mac, Dashboard and Computers lit up.
3. **Control.** Play, stop, volume, load scene, restart through the renderer's REST API. Zones tab becomes real.
4. **Config writes.** Computers tab writes through the agent. Site store appears here (Supabase: Postgres, auth, row-level security by site, file storage for speaker maps and resources). Speakermaps tab becomes real.
5. **Installer.** Shell app, privileged helper, steps 1 to 5 wired. This is the first thing a new customer touches, so it goes last where it can be tested against a finished app, not first.
6. **Accounts and remote.** Cloud sign-in, roles, local password fallback, cloud-hosted UI for off-site access through the agent's outbound connection.

Keep the mock adapter current throughout. It is the design surface, the demo, and the test fixture.

## 7. Decisions to make

- What the renderer's REST API already does (play, stop, volume, scene list, status?) and what the agent has to add. This decides how much of step 3 is new work.
- Shell app technology: Swift with WKWebView (smallest, best macOS integration, needed anyway for the helper) versus Tauri (Rust, cross-platform later). Recommendation: Swift.
- Site store: Supabase versus self-hosted Postgres. Recommendation: Supabase to start, for auth and storage out of the box.
- Offline rule: when the cloud is unreachable, can an installer still edit site objects? Proposal: no, playback only, to avoid merge conflicts between Macs.
- Speaker map format: today they are opaque uploaded files plus editable fields. Decide whether the renderer reads the JSON the UI edits, or the UI exports to the renderer's format.
