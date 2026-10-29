# NetScope — Technical Design Document

> Revision 2. Addresses every HIGH/MEDIUM finding and all NITs from
> `design-review.json` (review 1). See §17 Review Responses for the per-finding
> disposition.

## Overview

NetScope is a greenfield Electron desktop application for Windows that gives users real-time visibility into every device on their local network. It is inspired by Fing's device-list + device-detail UX pattern but carries an entirely original visual identity and adds features Fing lacks on desktop: dark/light theme toggle, new-device alerts, custom device labels, scheduled background scans with Windows toast notifications, a lightweight internet speed test, and a security checklist for high-risk open ports.

The application is built with Electron + Node.js. The renderer is plain HTML/CSS/JS with no heavy framework. Persistence uses better-sqlite3. Network discovery uses Windows-native commands (`arp -a`, `ping -n 1`) plus Node's `net` module for port scanning. The architecture is split so that the networking and persistence modules are plain CommonJS `require()`-able Node modules; they can be exercised from a headless smoke-test script without ever launching the Electron GUI. Because `better-sqlite3` is a native addon, a rebuild step reconciles the Node ABI (smoke test) and the Electron ABI (`npm start`).

Technology stack (locked once approved): Electron 29.1.0 + Node.js (Electron's bundled runtime), plain HTML/CSS/JS renderer, better-sqlite3 9.4.3, node-notifier 10.0.1, @electron/rebuild 3.6.0 (dev). No bundler, no React/Vue, no extra networking library.

Build order: empty Electron window → networking layer → UI shell → persistence + IPC wiring → nice-to-haves (scheduling, speed test, security checklist, export).

---

## 1. Folder Structure

```
netscope/
├── package.json
├── main.js                  ← Electron main process entry point
├── preload.js               ← contextBridge / IPC shim
│
├── src/
│   ├── net/
│   │   ├── discovery.js     ← gateway/IP/subnet detection + ARP + ping sweep
│   │   ├── portscanner.js   ← TCP port scan with concurrency, banner grab
│   │   ├── speedtest.js     ← internet speed test (Cloudflare endpoint)
│   │   └── oui.js           ← OUI CSV loader + vendor lookup
│   │
│   ├── db/
│   │   ├── database.js      ← better-sqlite3 init, migrations, WAL mode
│   │   └── queries.js       ← all SQL read/write helpers (no raw SQL outside)
│   │
│   ├── scheduler.js         ← setInterval-based background scan timer
│   ├── notifier.js          ← Windows toast via node-notifier
│   └── exporter.js          ← CSV generation (device-list or per-scan mode)
│
├── renderer/
│   ├── index.html           ← single-page shell
│   ├── app.js               ← renderer-side JS (IPC calls, DOM updates)
│   ├── styles/
│   │   ├── base.css         ← CSS custom properties (palette, typography)
│   │   ├── layout.css       ← card grid, sidebar, panels
│   │   └── theme.css        ← dark/light overrides via [data-theme] attribute
│   └── assets/
│       └── logo.svg         ← NetScope logo (original, see §7)
│
├── data/
│   └── oui.csv              ← bundled IEEE OUI file (fetched on first run if absent)
│
├── db/
│   └── netscope.db          ← SQLite file (created at runtime, gitignored)
│
└── smoke-test.js            ← headless test: imports discovery.js + queries.js, no GUI
```

`main.js` sets `app.setAppUserModelId('com.netscope.app')` before any window is created so Windows toast notifications are attributed correctly (see §10, §27).

**Key constraint**: `src/net/*.js` and `src/db/*.js` must use only Node built-ins, `better-sqlite3`, and each other — no Electron imports — so `smoke-test.js` can `require()` them directly under the Node ABI. The Electron main process depends on these modules and is the only place Electron APIs are imported.

---

## 2. IPC Contract

All IPC uses Electron's `ipcRenderer.invoke` (renderer → main, Promise-based) and `ipcMain.handle` on the main side. The preload exposes a `window.netscope` API object via `contextBridge`. Push events from main to renderer use `webContents.send` / `ipcRenderer.on`. Every `invoke` that rejects resolves to an `IPCError` object (see §2.3); the renderer wraps calls in `try/catch` and reads `err.error`.

### 2.1 Renderer → Main (invoke channels)

| Channel | Request payload | Response payload |
|---|---|---|
| `scan:start` | `{ portPreset: 'top20'\|'top100'\|'custom', portRange?: [number,number], confirm?: boolean }` | `{ scanId: number }` — scan row ID; progress arrives via push events. Rejects with `SCAN_IN_PROGRESS` if a scan is active, `SUBNET_TOO_LARGE` if the detected subnet exceeds the hard cap, or `SCAN_DECLINED` if the user rejects the large-subnet confirmation dialog. |
| `scan:cancel` | `{}` | `{ ok: boolean }` — `ok:false` (not an error) when no scan is running. After a successful cancel, main emits `scan:complete` with `cancelled:true` and emits no further `scan:progress` events. |
| `scan:history` | `{ limit?: number }` | `{ scans: Scan[] }` |
| `device:list` | `{}` | `{ devices: Device[] }` — latest known state per device (see §3.3) |
| `device:detail` | `{ mac: string }` | `{ device: Device, history: ScanResult[] }` |
| `device:rename` | `{ mac: string, label: string }` | `{ ok: boolean }` |
| `device:delete` | `{ mac: string }` | `{ ok: boolean }` — deletes only the `devices` row; `scan_results` history is retained (see §3) |
| `device:acknowledge` | `{ mac: string }` | `{ ok: boolean }` — user-driven; sets `is_known=1` so the device no longer alerts as new |
| `export:csv` | `{ scanId?: number }` | `{ path: string }` — absolute path written. `scanId` omitted → export device list (Mode A); `scanId` provided → export that scan's `scan_results` (Mode B). See §12. |
| `settings:get` | `{}` | `Settings` object (reassembled with defaults for missing keys) |
| `settings:set` | `Partial<Settings>` | `{ ok: boolean }` — upserts only the provided keys |
| `speedtest:run` | `{}` | `{ downloadMbps: number, latencyMs: number }` — rejects `SPEEDTEST_FAILED` (see §16) |
| `network:info` | `{}` | `{ gateway: string\|null, localIp: string\|null, cidr: string\|null, iface: string\|null, hostCount: number, prefix: number }` |

### 2.2 Main → Renderer (send / on channels)

| Channel | Payload | When |
|---|---|---|
| `scan:progress` | `{ phase: 'arp'\|'ping'\|'ports'\|'done', done: number, total: number, currentIp?: string }` | During active scan; not emitted after cancel |
| `scan:device-found` | `{ device: Device, isNew: boolean }` | Each time a device is discovered or updated |
| `scan:complete` | `{ scanId: number, newDevices: Device[], durationMs: number, cancelled: boolean }` | Scan finished (normally or after cancel) |
| `alert:new-device` | `{ device: Device }` | An unacknowledged MAC enters the network |
| `notification:scheduled` | `{ scanId: number }` | Background scan started automatically |

### 2.3 Type Shapes

```typescript
// IPCError — every invoke that rejects resolves to this (thrown in main,
// caught by the handler and returned as a rejected promise).
{ error: string; detail?: string }

// Device (as sent over IPC — plain serializable object)
{
  mac: string,            // AA:BB:CC:DD:EE:FF (uppercase, canonical)
  ip: string,
  hostname: string|null,
  vendor: string|null,
  label: string|null,     // user-supplied rename; null = use vendor/hostname
  openPorts: number[],    // from devices.open_ports (latest scan), JSON in DB
  firstSeen: number,      // Unix ms
  lastSeen: number,       // Unix ms
  isNew: boolean,         // scan: is_known===0 at scan time; list: is_known===0
  isOnline: boolean       // from devices.is_online
}

// ScanResult
{
  id: number,
  scanId: number,
  mac: string,
  ip: string,
  hostname: string|null,
  openPorts: number[],    // JSON-serialized in DB
  bannerInfo: object|null,// JSON { port: banner_string }; null if no banner
  respondedAt: number     // Unix ms
}

// Scan (as sent over IPC)
{
  id: number,
  startedAt: number,       // Unix ms
  finishedAt: number|null,// null while running
  trigger: 'manual'|'scheduled',
  gateway: string|null,
  localIp: string|null,
  cidr: string|null,
  deviceCount: number|null, // null while running
  newDeviceCount: number,
  durationMs: number|null   // finishedAt - startedAt; null if running
}

// Settings — first-run defaults (returned by settings:get when table empty)
{
  theme: 'dark'|'light',                  // default 'dark'
  scanIntervalMinutes: number,            // default 0 (disabled)
  portPreset: 'top20'|'top100'|'custom',  // default 'top20'
  portRange: [number, number],            // default [1, 1024]
  scheduledScansEnabled: boolean,         // default false
  desktopNotificationsEnabled: boolean    // default true
}
```

---

## 3. SQLite Schema

Database file: `db/netscope.db` (absolute path resolved relative to `app.getPath('userData')` at runtime, falls back to `./db/netscope.db` in dev mode).

Initialized in `src/db/database.js` using `PRAGMA journal_mode=WAL` and `PRAGMA foreign_keys=ON`.

```sql
-- ─── Devices ──────────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS devices (
  mac          TEXT PRIMARY KEY,          -- AA:BB:CC:DD:EE:FF
  ip           TEXT NOT NULL,             -- last-seen IP
  hostname     TEXT,
  vendor       TEXT,
  label        TEXT,                      -- user rename; NULL = use vendor/hostname
  open_ports   TEXT NOT NULL DEFAULT '[]', -- JSON array, latest scan's ports
  first_seen   INTEGER NOT NULL,          -- Unix ms
  last_seen    INTEGER NOT NULL,          -- Unix ms
  is_online    INTEGER NOT NULL DEFAULT 1,-- 1 = seen in latest scan, 0 = offline
  is_known     INTEGER NOT NULL DEFAULT 0 -- 0 = new/never acknowledged, 1 = known
);

-- ─── Scans ────────────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS scans (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  started_at   INTEGER NOT NULL,                 -- Unix ms
  finished_at  INTEGER,                          -- NULL while running
  trigger      TEXT NOT NULL DEFAULT 'manual',   -- 'manual'|'scheduled'
  cancelled    INTEGER NOT NULL DEFAULT 0,       -- 1 if cancelled by user
  gateway      TEXT,
  local_ip     TEXT,
  cidr         TEXT,
  device_count INTEGER,
  new_device_count INTEGER DEFAULT 0
);

-- ─── Scan Results ─────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS scan_results (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  scan_id      INTEGER NOT NULL REFERENCES scans(id) ON DELETE CASCADE,
  mac          TEXT NOT NULL,                    -- no FK to devices: a device may be
  ip           TEXT NOT NULL,                    --   deleted while history is retained
  hostname     TEXT,
  open_ports   TEXT NOT NULL DEFAULT '[]',        -- JSON array e.g. [22,80,443]
  banner_info  TEXT,                              -- JSON object { port: banner_string }
  responded_at INTEGER NOT NULL                   -- Unix ms
);
CREATE INDEX IF NOT EXISTS idx_scan_results_mac    ON scan_results(mac);
CREATE INDEX IF NOT EXISTS idx_scan_results_scan   ON scan_results(scan_id);

-- ─── Settings (key-value, one row per Settings field) ─────────────────────────
CREATE TABLE IF NOT EXISTS settings (
  key   TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
```

Note on `scan_results.mac`: it intentionally has **no foreign key** to `devices`. `device:delete` removes only the `devices` row; the device's scan history is retained so a re-discovered device keeps its prior history. A future "purge history" action would be separate scope.

### 3.1 first-seen / last-seen / open_ports / is_online logic

Owned by `src/db/queries.js`. `upsertDevice(device)` runs per discovered device during a scan:

```sql
INSERT INTO devices (mac, ip, hostname, vendor, label, open_ports, first_seen, last_seen, is_online, is_known)
  VALUES (?, ?, ?, ?, NULL, ?, ?, ?, 1, 0)
ON CONFLICT(mac) DO UPDATE SET
  ip        = excluded.ip,
  hostname  = excluded.hostname,
  vendor    = excluded.vendor,
  open_ports= excluded.open_ports,   -- latest scan's ports, JSON
  last_seen = excluded.last_seen,
  is_online = 1
  -- first_seen is never overwritten
  -- label is never overwritten (user may have renamed it)
  -- is_known is never reset to 0 once set to 1
```

At `scan:complete`, inside a single transaction, devices not seen this scan are marked offline (handles the empty-seen-set edge case):

```js
// queries.js
function markOfflineExcept(db, seenMacs) {
  if (seenMacs.length === 0) {
    db.prepare('UPDATE devices SET is_online = 0').run();
  } else {
    const placeholders = seenMacs.map(() => '?').join(',');
    db.prepare(`UPDATE devices SET is_online = 0 WHERE mac NOT IN (${placeholders})`).run(...seenMacs);
  }
}
```

### 3.2 New-vs-known device diff

At the start of each scan, `queries.getKnownMacs(db)` returns the set of all MACs with `is_known = 1`. For each device found during the scan:

```
isNew = !knownMacs.has(mac)        // simple, single rule: not yet acknowledged
```

- If `isNew`, emit `scan:device-found { device, isNew:true }` **and** `alert:new-device { device }`.
- `isNew` is consistent on the device-list view: a device shows the "new" badge while `is_known === 0`.

Acknowledgement is **user-driven, never automatic**. The UI calls `device:acknowledge { mac }` when the user dismisses/clicks a new-device card; `queries.acknowledgeDevice(db, mac)` sets `is_known = 1`. `is_known` is **not** flipped at `scan:complete`, so an unacknowledged device re-alerts on every subsequent scan until the user acknowledges it. This preserves the new-device-alert improvement: the first scan ever flags every found device as new until acknowledged, and only genuinely-unacknowledged devices keep alerting.

### 3.3 device:list query

```sql
SELECT mac, ip, hostname, vendor, label, open_ports, first_seen, last_seen, is_online, is_known
FROM devices
ORDER BY last_seen DESC;
```

`queries.js` parses `open_ports` (JSON → `number[]`), maps `is_online`/`is_known` (1→true, 0→false), and sets `isNew = (is_known === 0)` before returning `Device[]` over IPC. Because `open_ports` and `is_online` are now columns on `devices`, `device:list` is a single-table read with no join.

### 3.4 Settings storage mapping

Each `Settings` field is one row (`key` = field name, `value` = string):

- Scalars stored as their string form (`scanIntervalMinutes: '30'`, `portPreset: 'top20'`, `theme: 'dark'`).
- Booleans as `'true'` / `'false'`.
- `portRange` as JSON: `'[1,1024]'`.

`settings:set` upserts only the keys present in the `Partial<Settings>` payload:

```sql
INSERT INTO settings (key, value) VALUES (?, ?)
ON CONFLICT(key) DO UPDATE SET value = excluded.value;
```

`settings:get` reads all rows, parses each value by field type, applies the first-run defaults (§2.3) for any missing keys, and returns a full `Settings` object.

---

## 4. UI Wireframe

### 4.1 Application Shell

```
┌─────────────────────────────────────────────────────────────────────────┐
│  [◉ NetScope]   Network: 192.168.1.0/24 · GW 192.168.1.1  [🌙] [⚙]    │  ← titlebar
├──────────────┬──────────────────────────────────────────────────────────┤
│  SIDEBAR     │  MAIN PANEL                                               │
│              │                                                           │
│  Devices  ←  │  ← device-list or device-detail renders here             │
│  Scans       │                                                           │
│  Speed Test  │                                                           │
│  Settings    │                                                           │
│              │                                                           │
│  [▶ Scan]    │                                                           │
│  [↓ Export]  │                                                           │
└──────────────┴──────────────────────────────────────────────────────────┘
```

### 4.2 Device-List View

Each device is a card in a responsive CSS grid (2–4 columns depending on window width). All spec-required fields appear on the card:

```
┌──────────────────────────────────┐
│  [vendor icon / initial]         │
│  192.168.1.42                    │
│  AA:BB:CC:DD:EE:FF               │
│  Vendor: TP-Link Technologies    │
│  "My Router"   ← label if set    │
│  Hostname: router.local          │
│  Ports: 22 80 443   [⚠ HIGH RISK]│  ← ⚠ only if a port ∈ HIGH_RISK_PORTS
│  First seen: 2024-01-15          │
│  Last seen: 2 min ago   [● online]│
│                          [Rename]│
└──────────────────────────────────┘
```

New devices get a highlighted border (accent color) and a pulsing dot. The high-risk ⚠ badge uses a single shared constant `HIGH_RISK_PORTS = {22, 23, 3389, 445}` (defined once in `renderer/app.js`, see §11); ports outside this set never get the badge. FTP (21) and other informational ports are shown in the security checklist only (§11), never as a high-risk badge.

### 4.3 Device-Detail View

Clicking a card navigates (DOM swap, no router) to a full-panel detail view:

```
← Back

  ◉ 192.168.1.42 — "My Router" [Edit label]
  MAC: AA:BB:CC:DD:EE:FF · Vendor: TP-Link Technologies
  Hostname: router.local
  First seen: 2024-01-15 · Last seen: just now   [● online / offline]

  ┌─ Open Ports ──────────────────────────────┐
  │  Port 22  SSH    [⚠ HIGH RISK]            │
  │  Port 80  HTTP   ✓                        │
  │  Port 443 HTTPS  ✓                        │
  │  Banner:  OpenSSH 8.2p1 Ubuntu...         │
  └───────────────────────────────────────────┘

  ┌─ Security Checklist ──────────────────────┐
  │  ⚠ SSH (22) open — ensure key-based auth  │  ← HIGH_RISK_PORTS entries
  │  ℹ FTP (21) — plaintext creds; use SFTP  │  ← INFORMATIONAL_PORTS entries
  │  ℹ Default password reminder — change     │
  │     default router credentials            │
  └───────────────────────────────────────────┘

  ┌─ Scan History ────────────────────────────┐
  │  2024-01-20 14:32  Ports: 22,80,443       │
  │  2024-01-19 09:15  Ports: 80,443          │  ← diff: 22 appeared
  └───────────────────────────────────────────┘
```

### 4.4 Theme Toggle

`[🌙]` button in the titlebar toggles `document.documentElement.dataset.theme` between `'dark'` (default) and `'light'`. The chosen value is persisted via `settings:set { theme }`. On app start, the setting is read before first render to avoid flash.

### 4.5 Re-scan Button & Live Progress

`[▶ Scan]` in the sidebar:
- Becomes `[⏹ Cancel]` while a scan runs (calls `scan:cancel`).
- If a scan is already active, `scan:start` rejects with `SCAN_IN_PROGRESS`; the UI shows the Cancel button instead.
- Below it, a progress bar + text label: `"Pinging 192.168.1.42 (37/254)"`.
- Cards in the device list update in real time as `scan:device-found` arrives; the online indicator flips live because `upsertDevice` sets `is_online = 1` as devices are found.

---

## 5. Windows Discovery Flow

Implemented in `src/net/discovery.js`. All steps are async and use `child_process.exec` (or `spawn`) — **never `execSync`**, which would block IPC handling during parsing. The module exports `getNetworkInfo()` (pure parse, returns `{ gateway, localIp, cidr, iface, prefix, hostCount, hostCountCapped }`) and `discoverNetwork(scanId, opts)` which drives the scan and emits progress.

### Step 0 — Subnet-size guard (runs before any sweep)

```js
const MAX_HOSTS    = 1024;   // refuse larger subnets outright
const CONFIRM_HOSTS = 256;   // prompt the user above this (covers /24 silently)

function hostCount(prefixLen) { return Math.pow(2, 32 - prefixLen) - 2; }
```

`getNetworkInfo()` computes `hostCount(prefix)`. The `scan:start` handler (§8) enforces:
- `hostCount > MAX_HOSTS` → reject with `{ error: 'SUBNET_TOO_LARGE', detail }` (fatal, scan never starts).
- `hostCount > CONFIRM_HOSTS` → for a **manual** scan, show a confirmation dialog (`dialog.showMessageBox` in main): "This network has N hosts. Scan anyway?"; if declined → reject `{ error: 'SCAN_DECLINED' }`. **Scheduled** scans skip the prompt (no user present): they proceed if under `MAX_HOSTS` and skip+log if over.

### Step 1 — Auto-detect network info

```js
// Uses: child_process.exec  (async only; never execSync)
// Command: ipconfig /all
```

Adapter selection rule (in order):
1. Parse each adapter section. Consider only sections whose **"Default Gateway" is non-empty**.
2. Among those, skip adapters whose Description matches known virtual NICs: `/VirtualBox|Hyper-V|VMware|Docker|WSL|TAP|Tunnel|Loopback/i`.
3. If multiple remain, prefer an adapter whose IPv4 is in `192.168.x` or `10.x` over one in `172.16–31.x` (common home LANs).
4. If none have a gateway, fall back to the first non-loopback IPv4 and surface a UI warning (`network:info` returns `gateway:null`) so the user knows gateway detection failed.

For the chosen adapter, extract:
- `localIp` → "IPv4 Address" field (strip " (Preferred)" suffix).
- `subnetMask` → "Subnet Mask" field.
- `gateway` → "Default Gateway" field.
- `iface` → adapter name from the section header.

Derive CIDR via `maskToCidr(subnetMask)` (count set bits in the 4-octet mask — pure utility, no library): `cidr = localIp + "/" + prefix`.

### Step 2 — ARP table read (once, after the sweep)

The ARP table is read **once** after the ping sweep completes (Step 3), not per host. Parse `arp -a` lines matching `^\s+(\d+\.\d+\.\d+\.\d+)\s+([\da-fA-F-]+)\s+dynamic` (dynamic entries only). Normalize MAC to `AA:BB:CC:DD:EE:FF` uppercase colon-separated. Build `Map<ip, mac>` for matching against responding IPs.

### Step 3 — ICMP ping sweep (fills gaps in ARP table)

**Step 3a**: For each host IP in the subnet (derived from localIp + mask), run `ping -n 1 -w 500 <ip>`. Concurrency: **32 parallel pings** (a counter+queue semaphore in `discovery.js`). Collect the set of responding IPs. Emit progress `{ phase:'ping', done, total, currentIp }`.

**Step 3b**: Read `arp -a` once (Step 2) and build `Map<ip, mac>`.

**Step 3c**: For each responding IP, look up its MAC in the map (null if Windows did not populate ARP for it). Emit/update the device candidate.

This batching avoids up to 254 per-host `arp` exec calls; a single `arp -a` read after the sweep covers all hosts at once.

### Step 4 — Hostname resolution

For each discovered IP, `dns.promises.reverse(ip)` — timeout via `Promise.race` with a 2-second reject. Failure is non-fatal; `hostname` stays `null`.

### Step 5 — OUI vendor lookup

`src/net/oui.js` loads `data/oui.csv` into a `Map<string, string>`. The **canonical key form is colon-free** (`AABBCC`) on both sides:

```js
// loader: from CSV row -> key "AABBCC"
const key = row.Assignment.toUpperCase();          // IEEE Assignment column is colon-free

// lookup: from MAC -> key "AABBCC"
function ouiKey(mac /* "AA:BB:CC:DD:EE:FF" */) {
  return mac.replaceAll(':', '').slice(0, 6).toUpperCase();
}
```

If the file is absent, it is fetched from `https://standards-oui.ieee.org/oui/oui.csv` and written to `data/oui.csv`; on fetch failure, vendor stays `null` (non-fatal). The Map is a module-level singleton loaded once.

Note: IEEE `oui.csv` contains three registries (MA-L / MA-M / MA-S). This design matches **MA-L only** (24-bit prefix, the `oui.csv` Assignment column). MA-M (28-bit) and MA-S (36-bit) are intentionally not matched in v0.1; an un-matched prefix returns `null`. Extending to longest-prefix match is future scope.

---

## 6. Port Scan Throttling Strategy

Implemented in `src/net/portscanner.js`.

### 6.1 Port Presets

```js
const PRESETS = {
  top20: [21,22,23,25,53,80,110,111,135,139,143,443,445,993,995,1723,3306,3389,5900,8080],
  top100: [...new Set([
    // top20 +
    21,22,23,25,53,80,110,111,135,139,143,443,445,993,995,1723,3306,3389,5900,8080,
    8,20,24,26,81,82,83,8443,8888,1080,4444,5432,6379,27017,9090,9200,10000,2181,
    11211,1521,5000,5001,6667,7000,7001,7070,8000,8001,8008,8009,8161,8181,8983,
    9000,9300,10050,10051,49152,49153
    /* ... trimmed for brevity */ ])]
};
```

The `new Set([...])` wrapper dedupes the array (removes the prior accidental duplicates of 27017 and 5432).

### 6.2 Concurrency, Ordering & Timeout

Host scan ordering and global socket cap:

- **Hosts are scanned sequentially** (one host at a time). The scanner moves to the next host only after the current host's ports are settled.
- **A single global semaphore** `MAX_TOTAL_OPEN_SOCKETS = 64` governs every TCP connect across the whole scan. Each port connect acquires one slot; host sequencing plus this cap means the worst-case total open sockets across the entire app during a port scan is **64** — well within Node's handle pool and Windows' socket limits, regardless of how many hosts the subnet contains.

Per-port:
- Per-port TCP connect timeout: **1500 ms** (via `socket.setTimeout`).
- A refused connection (`ECONNREFUSED`) is an immediate, definitive "closed" — no wait.
- A timeout is treated as "filtered/closed" — port not added to the open list.

Worst case for a `/24` (254 hosts) with `top20`: 254 hosts × up to 20 ports, sequential hosts, 64 concurrent ports/host, 1500 ms timeout on filtered ports. Closed ports return near-instantly via `ECONNREFUSED`, so the practical time is far below the worst-case bound; only filtered ports pay the full timeout.

### 6.3 Banner Grabbing

Attempted only on ports known to send banners: `22, 21, 25, 110, 143, 3389, 80, 443, 8080, 8443`.

After connect:
- Send a minimal probe: `GET / HTTP/1.0\r\n\r\n` for HTTP ports; `\r\n` for others.
- Read up to **512 bytes** with a 2-second read timeout.
- Store raw banner string in `scan_results.banner_info` as JSON: `{ "80": "HTTP/1.1 200 OK Server: nginx..." }`.
- HTTPS (443, 8443): connect via `tls.connect`; read server certificate subject + issuer instead of raw banner.

### 6.4 Error Handling (port scan)

| Condition | Handling |
|---|---|
| ECONNREFUSED | Port closed — skip silently |
| ETIMEDOUT / socket timeout | Port filtered — skip silently |
| EHOSTUNREACH | Host offline — abort remaining ports for this IP, log warning |
| TLS handshake error on 443 | Banner = `null`, port still marked open (connection was accepted) |
| Any unexpected error | Log to `console.error`, skip port, do not crash scan |

---

## 7. Visual Identity

### 7.1 Palette (CSS custom properties, `base.css`)

```css
/* Dark theme (default) */
:root, [data-theme="dark"] {
  --bg-base:      #0f1117;   /* deep slate-black */
  --bg-surface:   #1a1d27;   /* card background */
  --bg-elevated:  #252836;   /* hover / active state */
  --border:       #2e3147;   /* subtle separator */
  --text-primary: #e8eaf0;   /* main text */
  --text-muted:   #7b82a0;   /* secondary labels */
  --accent:       #4f9cf9;   /* electric blue — buttons, links, active states */
  --accent-glow:  rgba(79,156,249,0.18);
  --success:      #2ecc8a;   /* online indicator */
  --warning:      #f5a623;   /* medium-risk ports */
  --danger:       #e05c5c;   /* high-risk ports, errors */
  --new-device:   #7c4dff;   /* purple pulse for new/unknown devices */
}

/* Light theme */
[data-theme="light"] {
  --bg-base:      #f0f2f8;
  --bg-surface:   #ffffff;
  --bg-elevated:  #e4e8f5;
  --border:       #cdd2e8;
  --text-primary: #1a1d27;
  --text-muted:   #5a6080;
  --accent:       #2563eb;
  --accent-glow:  rgba(37,99,235,0.12);
  --success:      #16a34a;
  --warning:      #d97706;
  --danger:       #dc2626;
  --new-device:   #6d28d9;
}
```

### 7.2 Typography

Font stack: `'Inter', 'Segoe UI', system-ui, sans-serif`. Inter loaded from bundled WOFF2 or fallback to system. Monospace for IPs/MACs: `'JetBrains Mono', 'Consolas', monospace`.

### 7.3 Logo

Original wordmark: the letters **NS** in a rounded-rectangle badge, stroke in `--accent` blue, fill transparent. SVG rendered inline. No resemblance to Fing's orange circle.

### 7.4 Cards

`border-radius: 12px`, `box-shadow: 0 2px 12px rgba(0,0,0,0.3)` (dark) / `0 1px 6px rgba(0,0,0,0.1)` (light). Hover lifts: `transform: translateY(-2px)`. New-device cards get `border: 1.5px solid var(--new-device)` with a CSS keyframe pulse on the status dot.

---

## 8. Error Handling — Main Process Operations

| Operation | Failure condition | Fatal? | Caller receives | Logged |
|---|---|---|---|---|
| `scan:start` while active | A scan is already running | Fatal to this request | IPC rejects `{ error:'SCAN_IN_PROGRESS' }` | No |
| `ipconfig /all` parse | No adapter found | Fatal to scan | IPC rejects `{ error:'NO_ADAPTER' }` | `console.error` |
| Subnet size | `hostCount > MAX_HOSTS` | Fatal to scan | IPC rejects `{ error:'SUBNET_TOO_LARGE', detail }` | `console.error` |
| Subnet confirm declined (manual) | User clicks No | Fatal to this request | IPC rejects `{ error:'SCAN_DECLINED' }` | No |
| `arp -a` exec | Command fails / empty | Non-fatal | ARP phase skipped; ping sweep continues | `console.warn` |
| Ping sweep single host | exec timeout | Non-fatal | Host marked offline | No (too noisy) |
| DNS reverse lookup | Timeout / NXDOMAIN | Non-fatal | `hostname: null` | No |
| OUI fetch from IEEE | Network error | Non-fatal | `vendor: null` for all | `console.warn` once |
| SQLite `upsertDevice` | DB locked / corrupt | Fatal to scan | IPC rejects `{ error:'DB_ERROR', detail }` | `console.error` |
| Port connect (unexpected) | Any Error | Non-fatal | Port skipped | `console.error` per port |
| Speed test fetch | Network error / timeout | Non-fatal to scans | IPC rejects `{ error:'SPEEDTEST_FAILED' }` | `console.warn` |
| CSV export write | Disk full / permission | Non-fatal | IPC rejects `{ error:'EXPORT_FAILED', detail }` | `console.error` |

---

## 9. Input Validation

| Input | Rules | On failure |
|---|---|---|
| `scan:start` portRange | Array of two integers, both 1–65535, first ≤ second | IPC rejects `{ error:'INVALID_PORT_RANGE' }` |
| `scan:start` (active scan) | A scan must not already be running | IPC rejects `{ error:'SCAN_IN_PROGRESS' }` |
| `scan:start` subnet size | `hostCount ≤ MAX_HOSTS`; `> CONFIRM_HOSTS` requires user confirm (manual) | `SUBNET_TOO_LARGE` / `SCAN_DECLINED` |
| `device:*` `mac` param (detail, rename, delete, acknowledge) | Normalize: trim → uppercase → replace `[-.]` with `:` → must match `^([0-9A-F]{2}:){5}[0-9A-F]{2}$` | IPC rejects `{ error:'INVALID_MAC' }` |
| `device:rename` label | Trimmed, then must match `^[ -~\u00A0-\u024F]{1,64}$` (ASCII 0x20–0x7E + Latin-1 letters), max 64 chars | IPC rejects `{ error:'INVALID_LABEL' }` |
| `settings:set` scanIntervalMinutes | Integer 0–1440 | Clamped to range, no error |
| `settings:set` theme | Enum `'dark'` or `'light'` | IPC rejects `{ error:'INVALID_THEME' }` (invalid value is rejected, not silently kept) |
| MAC from arp/discovery | Must match `^([0-9A-F]{2}:){5}[0-9A-F]{2}$` after normalization | Row discarded with `console.warn` |
| IP from arp/discovery | Must be valid private-range IPv4 (10.x, 172.16–31.x, 192.168.x) | Row discarded — prevents accidental remote scan |
| Subnet from discovery | Derived from localIp + mask; `hostCount` computed by `getNetworkInfo` | Enforced by the §5 Step 0 guard; no user-supplied target IP exists in any IPC channel |

The no-remote-scanning invariant is enforced in `discovery.js`: every scanned IP is generated from the detected private subnet, and the private-IP validation discards any non-private IP. There is no IPC channel that lets the user feed an arbitrary target IP into the scanner.

---

## 10. Scheduling & Notifications

`src/scheduler.js` uses `setInterval` (no extra dependency) keyed off `settings.scanIntervalMinutes`. On app start, if `scheduledScansEnabled = true` and interval > 0, the timer is armed. Changing settings re-arms the timer.

When a scheduled scan fires:
1. Send `notification:scheduled` IPC push to renderer (updates UI status bar).
2. Call `discoverNetwork()` + port scan exactly like a manual scan, with `trigger = 'scheduled'`. Scheduled scans **skip the large-subnet confirmation dialog** (no user present): they proceed if under `MAX_HOSTS` and skip+log if over.
3. On completion, if `desktopNotificationsEnabled = true`, fire `src/notifier.js`, which uses `node-notifier` (its Windows toast path relies on the bundled `snoretoast.exe`). Reliable delivery on Windows 10/11 benefits from the app's AppUserModelID, set in `main.js` via `app.setAppUserModelId('com.netscope.app')` at startup. Toast text: "NetScope scan complete — N devices found, M new".
4. If new (unacknowledged) devices were found, the toast also says "⚠ Unknown device(s) detected".

---

## 11. Security Checklist Logic

Owned by `renderer/app.js` (pure client-side, no server involvement). One shared constant drives both the device-card badge and the checklist:

```js
// renderer/app.js — single source of truth
const HIGH_RISK_PORTS = {
  22:   { label: 'SSH',     msg: 'Ensure key-based auth is enabled; disable password login.' },
  23:   { label: 'Telnet',  msg: 'Telnet is unencrypted. Disable if possible.' },
  3389: { label: 'RDP',     msg: 'RDP is a common attack vector. Use VPN + NLA.' },
  445:  { label: 'SMB',      msg: 'SMB can expose file shares. Verify it is intentional.' },
};

// Ports that are notable but NOT high-risk — shown in the checklist, never as a ⚠ badge.
const INFORMATIONAL_PORTS = {
  21:  { label: 'FTP',  msg: 'FTP transfers credentials in plaintext. Use SFTP.' },
  25:  { label: 'SMTP', msg: 'Open SMTP relay can be abused. Restrict if unintended.' },
};

const DEFAULT_PASSWORD_REMINDER = 'Change default router/device credentials if not already done.';
```

The device-card ⚠ badge checks `openPorts.some(p => p in HIGH_RISK_PORTS)`. The device-detail checklist renders entries from `HIGH_RISK_PORTS` (⚠) and `INFORMATIONAL_PORTS` (ℹ) for any open port in those maps, plus the default-password reminder. No network request or exploit attempt is made. The checklist is renderer-only and requires no IPC.

---

## 12. CSV Export Format

Written by `src/exporter.js`, invoked from the `export:csv` IPC handler in `main.js`. The handler branches on whether `scanId` is present.

**Mode A — `scanId` omitted: export the current device list** (from `devices`):

```
mac,ip,hostname,vendor,label,open_ports,first_seen,last_seen,is_online
AA:BB:CC:DD:EE:FF,192.168.1.1,router.local,TP-Link Technologies,My Router,"[22,80,443]",2024-01-15T10:00:00Z,2024-01-20T14:32:00Z,true
```

**Mode B — `scanId` provided: export that scan's `scan_results`**:

```
mac,ip,hostname,open_ports,banner_info,responded_at
AA:BB:CC:DD:EE:FF,192.168.1.1,router.local,"[22,80,443]","{""22"":""OpenSSH 8.2p1""}",2024-01-20T14:32:00Z
```

Timestamps are ISO-8601 UTC strings in the CSV (rendered from the stored Unix-ms integers). The save path is chosen via `dialog.showSaveDialog` in main; the result path is returned to the renderer via the IPC response.

---

## 13. Smoke Test (headless)

`smoke-test.js` (run with `node smoke-test.js`, no Electron — executes against the **Node ABI** build of `better-sqlite3`):

1. `require('./src/db/database')` — opens SQLite (in-memory `:memory:`), runs migrations, closes.
2. `require('./src/net/oui')` — loads OUI CSV, asserts vendor lookup for a known prefix (uses colon-free key normalization, §5 Step 5).
3. `require('./src/net/discovery')` — calls `getNetworkInfo()` (ipconfig parse on a fixture string), logs gateway/IP/CIDR/hostCount. Does NOT run a full ping sweep in CI (would take minutes); asserts the exported function signatures are callable.
4. **New-vs-known diff logic** (the core path flagged in review finding #7), using an in-memory DB and `queries.js`:
   - `upsertDevice(deviceA)` with `is_known=0`; assert `getKnownMacs()` = `[]`.
   - Simulate scan: `isNew = !knownMacs.has(macA)` → `true`.
   - `acknowledgeDevice(macA)`; assert `getKnownMacs()` = `[macA]`.
   - Simulate scan: find `deviceA` → `isNew = false`; find `deviceB` (new) → `isNew = true`.
   - `markOfflineExcept(db, [macA, macB])` then re-run with `[macA]` only; assert `deviceA.isOnline=1`, `deviceB.isOnline=0`.

Note: `npm start` runs under the **Electron ABI**; the `better-sqlite3` native build is not interchangeable between the two without rebuilding (see §14).

---

## 14. Dependencies (package.json excerpt)

```json
{
  "name": "netscope",
  "version": "0.1.0",
  "main": "main.js",
  "scripts": {
    "start": "electron .",
    "smoke": "node smoke-test.js",
    "postinstall": "electron-rebuild -f -w better-sqlite3"
  },
  "dependencies": {
    "better-sqlite3": "9.4.3",
    "node-notifier": "10.0.1"
  },
  "devDependencies": {
    "electron": "29.1.0",
    "@electron/rebuild": "3.6.0"
  }
}
```

`postinstall` runs `electron-rebuild -f -w better-sqlite3` to compile `better-sqlite3` against Electron's ABI, so `npm start` does not throw `NODE_MODULE_VERSION mismatch` on `require('better-sqlite3')`. `npm run smoke` rebuilds for the Node ABI implicitly via the `node` runtime that invokes it; the two builds (Node ABI for smoke, Electron ABI for start) are not interchangeable without rebuilding. `node-notifier` bundles `snoretoast.exe` for Windows toasts. No bundler. Node's built-in `net`, `dns`, `child_process`, `fs`, `tls`, `https` cover all networking. No React, Vue, or other renderer framework.

---

## 15. Testability Notes

- `src/net/discovery.js` exports `getNetworkInfo()` (pure parse of ipconfig output string) — fully unit-testable by passing a fixture string. `maskToCidr` and the adapter-selection rule are pure functions, unit-testable independently.
- `src/net/portscanner.js` accepts an injectable `connectFn` parameter for mocking TCP connections in unit tests; the global semaphore is instantiated with a configurable cap.
- `src/db/queries.js` accepts a `db` parameter (better-sqlite3 instance) so tests can pass an in-memory `:memory:` DB (exercised by smoke-test step 4).
- `src/net/oui.js` key normalization is a pure function, unit-testable.
- Integration tests (run on a real Windows machine) cover the full scan cycle via `smoke-test.js`.
- UI logic in `renderer/app.js` is pure DOM manipulation — testable with jsdom if needed.

---

## 16. Internet Speed Test

Implemented in `src/net/speedtest.js`, exposed via the `speedtest:run` IPC channel. It is a standalone user action, non-fatal to scans.

- **Download**: `GET https://speed.cloudflare.com/__down?bytes=25000000` (25 MB). Record `t0 = Date.now()` at request start; on the response `'end'` event, `seconds = (Date.now() - t0) / 1000`; `downloadMbps = (25000000 * 8) / seconds / 1e6`.
- **Latency**: three sequential `HEAD` requests to `https://speed.cloudflare.com/__down?bytes=0`; `latencyMs` = median of the three round-trip times. (Equivalently, TCP connect RTT to `speed.cloudflare.com:443` via `net.connect` may be used; the implementation picks the HEAD-request median for simplicity.)
- **Upload**: **omitted by design in v0.1** (a conscious scope decision — the requirement is "internet speed test" and download+latency satisfy it). Upload is deferred to a later iteration.
- **Timeout**: 20 s overall; if any request exceeds it or errors, reject the IPC with `{ error: 'SPEEDTEST_FAILED' }`.
- Returns `{ downloadMbps: number, latencyMs: number }`.

`speedtest.js` uses only Node's `https` built-in (no `fetch` dependency assumption), so it remains require-able in the headless smoke context. It is not exercised by the smoke test (it makes a real network request); its function signature is asserted callable only.

---

## 17. Review Responses

Disposition of every finding from `design-review.json` (review 1). All 27 findings are addressed; none are backlogged.

| # | Severity | Finding | Disposition |
|---|---|---|---|
| 1 | HIGH | Unbounded subnet size | **Addressed.** Added §5 Step 0 guard (`MAX_HOSTS=1024`, `CONFIRM_HOSTS=256`, `hostCount()`), `SUBNET_TOO_LARGE` fatal row in §8, subnet-size rules in §9. Scheduled scans skip the prompt; manual scans confirm via dialog; declined → `SCAN_DECLINED`. |
| 2 | MEDIUM | Port-scan concurrency not global | **Addressed.** §6.2 now specifies sequential host scanning + a single global semaphore `MAX_TOTAL_OPEN_SOCKETS=64`; worst-case total open sockets = 64 regardless of subnet size. |
| 3 | MEDIUM | `devices` schema missing `open_ports`/`is_online` | **Addressed.** Added `open_ports TEXT` and `is_online INTEGER` columns to `devices` (§3); `upsertDevice` updates both (§3.1); `markOfflineExcept` resets `is_online=0` for unseen MACs at `scan:complete` with empty-set handling; `device:list` is now a single-table read (§3.3). |
| 4 | MEDIUM | OUI key format mismatch | **Addressed.** §5 Step 5 defines canonical colon-free keys: loader `row.Assignment.toUpperCase()`, lookup `mac.replaceAll(':','').slice(0,6).toUpperCase()`. Noted MA-L-only matching; MA-M/MA-S deferred. |
| 5 | MEDIUM | Adapter selection picks virtual NICs | **Addressed.** §5 Step 1 selection rule: require non-empty Default Gateway, skip `/VirtualBox|Hyper-V|VMware|Docker|WSL|TAP|Tunnel|Loopback/i`, prefer 192.168/10.x over 172.16-31.x, fall back to first non-loopback IPv4 with a UI warning if none qualify. |
| 6 | MEDIUM | `Scan` type undefined | **Addressed.** Added the `Scan` shape to §2.3 with numeric Unix-ms timestamps, `trigger`, `deviceCount`, `newDeviceCount`, `durationMs`. |
| 7 | MEDIUM | Diff logic contradicts; auto-ack disables alerts | **Addressed.** §3.2 now uses the single rule `isNew = !knownMacs.has(mac)` (dropped the "not in devices table" clause). Acknowledgement is user-driven via new `device:acknowledge` IPC (§2.1); `is_known` is never flipped at `scan:complete`, so unacknowledged devices re-alert until acknowledged. Validation row added in §9. |
| 8 | MEDIUM | `export:csv` `scanId` semantics | **Addressed.** §12 defines two modes: `scanId` omitted → device-list CSV (Mode A); `scanId` provided → per-scan `scan_results` CSV (Mode B), with distinct column sets. |
| 9 | MEDIUM | `settings:set` storage mapping | **Addressed.** §3.4 specifies per-field rows, booleans as `'true'`/`'false'`, `portRange` as JSON, `ON CONFLICT(key) DO UPDATE` upsert of only provided keys, and `settings:get` reassembly with defaults. |
| 10 | MEDIUM | Speed test underspecified | **Addressed.** New §16 pins the Cloudflare download endpoint (25 MB), latency = median of 3 HEAD requests, 20 s timeout, `SPEEDTEST_FAILED` rejection, and explicitly defers upload to a later iteration. |
| 11 | MEDIUM | MAC validation missing for `device:*` | **Addressed.** §9 adds a canonical-MAC normalize+regex rule applied to `device:detail`, `device:rename`, `device:delete`, and `device:acknowledge`, rejecting with `INVALID_MAC`. |
| 12 | MEDIUM | `better-sqlite3` Electron ABI rebuild | **Addressed.** §14 adds `@electron/rebuild` devDep and a `postinstall` script; §13/§15 note the Node-ABI (smoke) vs Electron-ABI (start) builds are not interchangeable without rebuilding. |
| 13 | NIT | §1 mentions `node-schedule` | **Addressed.** §1 scheduler.js comment now reads "setInterval-based background scan timer"; no `node-schedule` mention, consistent with §10/§14. |
| 14 | NIT | High-risk port list differs (badge vs checklist) | **Addressed.** §11 defines one shared `HIGH_RISK_PORTS = {22,23,3389,445}` used by both the badge and checklist; 21 (FTP) moved to a separate `INFORMATIONAL_PORTS` map shown only in the checklist. |
| 15 | NIT | `top100` duplicate ports | **Addressed.** §6.1 builds `top100` via `new Set([...])` to dedupe. |
| 16 | NIT | Per-host ARP re-read | **Addressed.** §5 Step 3 restructured: full ping sweep first (3a), single `arp -a` read (3b), match MACs by IP (3c). |
| 17 | NIT | IPC error shape not formalized | **Addressed.** Added `IPCError { error: string; detail?: string }` to §2.3; noted `invoke` rejects with it. |
| 18 | NIT | `scan:cancel` edge cases | **Addressed.** §2.1 `scan:cancel` returns `{ ok:false }` when no scan is running; §2.2 `scan:complete` carries `cancelled:boolean`; no further `scan:progress` after cancel. |
| 19 | NIT | `ipconfig` parse execSync/exec | **Addressed.** §5 Step 1 commits to async `child_process.exec` only; states "never `execSync`". |
| 20 | NIT | `device:rename` printable chars undefined | **Addressed.** §9 defines the label regex `^[ -~\u00A0-\u024F]{1,64}$` (ASCII 0x20–0x7E + Latin-1), max 64, trimmed; rejects with `INVALID_LABEL`. |
| 21 | NIT | Card wireframe omits hostname + first_seen | **Addressed.** §4.2 card now includes a `Hostname:` line and a `First seen:` line. |
| 22 | NIT | `settings:set` invalid theme ambiguous | **Addressed.** §9 rejects invalid `theme` with `INVALID_THEME` (not silently kept). |
| 23 | NIT | `device:delete` orphaning | **Addressed.** §2.1 and §3 state `device:delete` removes only the `devices` row; `scan_results` has no FK and retains history; future purge-history is separate scope. |
| 24 | NIT | Smoke test doesn't exercise diff logic | **Addressed.** §13 step 4 exercises `upsertDevice`/`getKnownMacs`/`isNew`/`acknowledgeDevice`/`markOfflineExcept` on an in-memory DB. |
| 25 | NIT | Default Settings unspecified | **Addressed.** §2.3 lists first-run defaults (theme dark, scanIntervalMinutes 0, portPreset top20, portRange [1,1024], scheduledScansEnabled false, desktopNotificationsEnabled true). |
| 26 | NIT | `scan:start` while running unspecified | **Addressed.** §2.1, §8, §9 reject with `SCAN_IN_PROGRESS` if a scan is active. |
| 27 | NIT | Toast snoretoast + AppUserModelID | **Addressed.** §1 notes `main.js` sets `app.setAppUserModelId('com.netscope.app')`; §10 notes `node-notifier` relies on bundled `snoretoast.exe`. |
