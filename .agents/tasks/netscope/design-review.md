# NetScope Design Review — Review 2

**Document reviewed:** `design.md` (Revision 2)
**Verdict:** `NETSCOPE_DESIGN_CHANGES_REQUESTED`
**Counts:** 2 HIGH, 8 MEDIUM, 11 NIT

This review was performed fresh against the design document, with key external
assumptions verified by running the actual Windows commands on the target
machine (Portuguese-locale Windows 10/11). Where the design makes claims about
system behavior, the claims were checked against real output rather than taken
at face value.

---

## Summary

The design is well-structured and covers every must-have feature and every
improvement from the requirements, with sound separation of networking and
persistence into `require()`-able modules, an original visual identity, and a
bounded port-scan throttling strategy. The SQLite schema is sound and supports
new-vs-known diffing.

However, two HIGH findings block implementation:

1. **Discovery assumes English-locale Windows command output.** This was
   *verified broken on the user's own machine* (Portuguese Windows): `ipconfig`
   field names, the `arp -a` "dynamic" literal, and the "(Preferred)" suffix
   are all localized and do not match the design's English-string parsers. The
   entire discovery flow — and therefore the entire app — fails on every
   non-English Windows install.
2. **The headless smoke test cannot run as specified.** The design's claim that
   `node smoke-test.js` "rebuilds better-sqlite3 for the Node ABI implicitly" is
   false; the `postinstall` builds for the Electron ABI, so the smoke test
   crashes with `NODE_MODULE_VERSION mismatch`.

Eight MEDIUM findings address contract ambiguity, a scheduled/active-scan race,
OUI storage, live-progress event timing, CSV escaping, missing contract
behavior, an undefined preset, and the "background scan" operational model.

---

## Verified Assumptions

These claims in the design were checked and hold:

- **Cloudflare speed-test endpoints** (`https://speed.cloudflare.com/__down?bytes=N`
  download, `bytes=0` latency) are real and behave as described in §16. *(Verified by
  knowledge; could not live-fetch.)*
- **IEEE `oui.csv` columns** are `Registry,Assignment,Organization Name,Organization Address`
  with a colon-free hex `Assignment` (MA-L = 6 hex digits). The colon-free key
  normalization in §5 Step 5 (`replaceAll(':','').slice(0,6).toUpperCase()`) is correct
  for MA-L matching. *(Verified by knowledge.)*
- **`arp -a` MAC format** is dash-separated lowercase hex (e.g. `6e-a5-1c-38-66-9d`).
  The design's `[\da-fA-F-]+` capture plus uppercase-colon normalization handles this.
  *(Verified on the user's machine.)*
- **`better-sqlite3`** supports `journal_mode=WAL`, `foreign_keys=ON`,
  `INSERT ... ON CONFLICT DO UPDATE`, and is synchronous (no async race within one
  process). The §3 schema and upsert are sound. *(Verified by knowledge.)*
- **`ping -n 1 -w 500`** flags are correct (count = 1, timeout = 500 ms). *(Verified
  by knowledge.)*
- **`node-notifier`** bundles `snoretoast.exe` for Windows toasts, and
  `app.setAppUserModelId` improves toast attribution. *(Verified by knowledge.)*
- **Port-scan throttling (§6.2)** — sequential hosts + a single 64-socket global
  semaphore + 1500 ms per-port timeout — is safe and bounded. Worst-case total open
  sockets during a scan = 64 regardless of subnet size. *(Verified by analysis.)*
- **SQLite schema (§3)** — `devices` (mac PK; `is_known`/`is_online`/`open_ports`/
  `first_seen`/`last_seen`), `scans`, `scan_results` (`scan_id` FK `ON DELETE CASCADE`;
  `mac` intentionally without FK so history survives device deletion), with indexes
  on `mac` and `scan_id` — is sound and supports new-vs-known diffing via
  `getKnownMacs()`/`is_known`. *(Verified by analysis.)*
- **No-remote-scanning is enforced**: every scanned IP is generated from the
  detected private subnet, discovered IPs are validated as private-range (§9), and
  no IPC channel accepts an arbitrary target IP. The speed test hits a single fixed
  Cloudflare endpoint (not a scan). *(Verified by analysis.)*
- **No-exploitation is maintained**: banner grabbing sends only benign probes
  (`GET / HTTP/1.0`, `\r\n`) and reads responses / TLS certificate metadata; the
  security checklist (§11) is renderer-only and informational; there is no exploit
  payload, brute force, or intrusion. *(Verified by analysis.)*
- **Visual identity is original (§7)**: blue/purple palette, an "NS" wordmark badge,
  no Fing orange/circle/copy. *(Verified by analysis.)*
- **All must-have features and all improvements are represented** in the design.
  *(Verified by cross-check against the requirements list.)*

## Unverified / Wrong Assumptions

- **WRONG** — "npm run smoke rebuilds for the Node ABI implicitly via the node
  runtime that invokes it" (§13/§14). Node does not rebuild native addons; the
  Electron-ABI build throws `NODE_MODULE_VERSION mismatch` under Node. → Finding 2.
- **WRONG** — `ipconfig /all` output uses English labels "IPv4 Address" / "Subnet
  Mask" / "Default Gateway" / "Description" (§5 Step 1). Verified on the user's
  Portuguese Windows that the labels are localized ("Endereço IPv4", "Máscara de
  Sub-rede", "Gateway Padrão", "Descrição"). → Finding 1.
- **WRONG** — the `arp -a` type column is the literal "dynamic" (§5 Step 2).
  Verified it is localized ("dinâmico" / "estático"). → Finding 1.
- **WRONG** — the "(Preferred)" suffix is " (Preferred)" with a leading space
  (§5 Step 1). Verified it is "(Preferencial)" with NO leading space and localized.
  → Finding 1.
- **WRONG** — the gateway is a single value on one line (§5 Step 1). Verified the
  gateway field can span multiple lines (IPv6 link-local on the first line, IPv4 on
  a continuation line). → Finding 1.
- **CONTRADICTORY** — the OUI file is described as both "bundled" and "fetched on
  first run if absent" (§1/§5 Step 5), and is stored in the project/app dir which is
  read-only when packaged. → Finding 5.

---

## Findings

### 1. HIGH — Discovery assumes English-locale Windows output (verified broken on the user's machine)

**Location:** §5 Step 1 (ipconfig parse + "(Preferred)" strip), §5 Step 2
(arp -a regex), §9 (IP/MAC validation), §8 (NO_ADAPTER).

**Problem:** The discovery flow parses English string literals from Windows
commands. Verified on the user's actual machine (Portuguese Windows) that none of
these match:

| Design expects | Actual (Portuguese Windows) |
|---|---|
| `IPv4 Address` | `Endereço IPv4` |
| `Subnet Mask` | `Máscara de Sub-rede` |
| `Default Gateway` | `Gateway Padrão` |
| `Description` | `Descrição` |
| `Ethernet adapter` (section header) | `Adaptador Ethernet` / `Adaptador de Rede sem Fio` |
| arp type `dynamic` | `dinâmico` |
| ` (Preferred)` (leading space) | `(Preferencial)` (no space) |
| gateway on one line | IPv6 link-local + IPv4 on a continuation line |

Consequences: the ipconfig parser finds no adapter → `NO_ADAPTER` on every scan;
even if it parsed, the ARP regex anchors on `dynamic` which matches nothing → all
MACs are `null` → the device model (keyed on MAC, diffed by MAC) collapses. This
breaks the core feature on every non-English Windows install — including the
user's.

**Fix:** Drop English-label text parsing entirely.

- **Local IP / netmask / MAC:** use Node's `os.networkInterfaces()` (locale-
  independent). Apply the existing virtual-NIC filter
  (`/VirtualBox|Hyper-V|VMware|Docker|WSL|TAP|Tunnel|Loopback/i` against the
  interface description) and private-range filter to the returned interfaces.
  Each entry already provides `address` (IPv4), `netmask`, and `mac`.
- **Gateway:** derive from the default route via `route print -4` (or
  `netstat -r -4`). Locate the row whose destination is the numeric `0.0.0.0` and
  read its gateway column — `0.0.0.0` and the gateway IP are numeric, not
  localized, so this is locale-independent and also fixes the multi-line gateway
  problem (the IPv4 gateway is a single numeric token on that row).
- **CIDR:** `cidr = localIp + "/" + countSetBits(netmask)`.
- **ARP:** replace the `dynamic`-anchored regex with a locale-independent pattern
  that captures IP + MAC regardless of the type word, then keep only entries whose
  IP is private-range AND within the scanned subnet:
  ```js
  const re = /^\s+(\d{1,3}(?:\.\d{1,3}){3})\s+([\da-fA-F]{2}(?:[-:][\da-fA-F]{2}){5})\s+\S+/;
  // then: keep entry only if isPrivateIp(ip) && ipInSubnet(ip, cidr)
  ```
  (Static broadcast/multicast rows like `255.255.255.255` and `224.0.0.x` already
  fail the private-range/subnet check, so dropping the `dynamic` anchor is safe.)
- **(Preferred) suffix:** strip any trailing parenthetical from an IP value with
  `\s*\([^)]*\)\s*$` rather than the literal ` (Preferred)`.

### 2. HIGH — Smoke test cannot run; the ABI "implicit rebuild" claim is false

**Location:** §13 (Smoke Test), §14 (Dependencies).

**Problem:** §14's `postinstall` runs `electron-rebuild -f -w better-sqlite3`,
which compiles the native addon against the **Electron** ABI. §13/§14 then state
that `npm run smoke` (which runs `node smoke-test.js`) "rebuilds for the Node ABI
implicitly via the node runtime that invokes it." That is false — invoking `node`
does not rebuild a native addon. Loading an Electron-ABI build under Node throws
`NODE_MODULE_VERSION mismatch`, so smoke-test step 1 (`require('./src/db/database')`)
crashes immediately. The required headless smoke test (the testability
requirement) is not executable as specified.

**Fix:** Add explicit, documented rebuild steps. For example:
```json
"scripts": {
  "start": "electron .",
  "smoke": "node smoke-test.js",
  "rebuild:node": "npm rebuild better-sqlite3 --build-from-source",
  "rebuild:electron": "electron-rebuild -f -w better-sqlite3",
  "presmoke": "npm run rebuild:node",
  "postinstall": "npm run rebuild:electron"
}
```
and document that `npm start` requires `npm run rebuild:electron` after a smoke
run. Alternatively, run the smoke DB exercises against a pure-JS SQLite
(e.g. `sql.js` in-memory) so no native rebuild is needed for headless tests,
keeping `better-sqlite3` only for the Electron app path.

### 3. MEDIUM — `confirm` field in `scan:start` is undefined and conflicts with the main-process dialog

**Location:** §2.1 (`scan:start` payload `confirm?: boolean`), §5 Step 0
(`dialog.showMessageBox` in main), §8.

**Problem:** The `scan:start` request carries `confirm?: boolean` but its meaning
is never defined, while §5 Step 0 says the large-subnet confirmation is a
`dialog.showMessageBox` shown by the main process. It is unclear whether the
renderer is expected to pre-confirm (pass `confirm:true` to skip the dialog) or
whether main always shows the dialog and ignores the field. Two valid
interpretations exist.

**Fix:** Remove `confirm` from the `scan:start` payload and have main always
show the dialog for manual scans over `CONFIRM_HOSTS` (scheduled scans skip it).
Or, if renderer-driven confirmation is desired, define it explicitly: "main
shows no dialog; it rejects `{ error:'SUBNET_CONFIRM_REQUIRED' }` when
`hostCount > CONFIRM_HOSTS && confirm !== true`, and the renderer re-invokes
with `confirm:true` after its own dialog." Pick one and state it.

### 4. MEDIUM — Scheduled scan vs active scan overlap is unspecified (race on `is_online`)

**Location:** §10 (Scheduling), §6.2, §3.1 (`markOfflineExcept`).

**Problem:** §10 step 2 has the scheduler call `discoverNetwork()` + port scan
"exactly like a manual scan" directly, bypassing the `scan:start` IPC and its
`SCAN_IN_PROGRESS` guard. If a manual scan is running when the timer fires, two
scans run concurrently. `markOfflineExcept` (§3.1) at one scan's completion marks
the other scan's not-yet-seen devices `is_online=0`, producing incorrect online
state. Separately, §6.2 describes the 64-socket semaphore as governing "across
the whole scan" (per-scan), so two concurrent scans could exceed the 64-socket
bound.

**Fix:** Make "a scan is active" a single module-level flag shared by both the
`scan:start` handler and the scheduler. The scheduler must check it before
firing and skip (log + optional notify) when a scan is in progress. Also state
that the 64-socket semaphore is one module-level instance shared across all
scans, so the bound holds even if concurrent scans are ever permitted.

### 5. MEDIUM — OUI file is contradictory (bundled vs fetched) and stored in a read-only location

**Location:** §1 (`data/oui.csv` "bundled IEEE OUI file (fetched on first run if
absent)"), §5 Step 5, §3 (DB uses `app.getPath('userData')`).

**Problem:** "bundled" (ships with the app) contradicts "fetched on first run if
absent." More importantly, `data/oui.csv` lives in the project/app install
directory, which is read-only in a packaged Windows app, so a runtime fetch
writing there fails with `EACCES`. The DB correctly resolves to
`app.getPath('userData')` but the OUI file does not, so the two persistence
locations are inconsistent. (This also breaks the `oui.js` smoke test unless the
path is injectable.)

**Fix:** Store `oui.csv` under `app.getPath('userData')` (writable). Pick one
model: ship a bundled read-only fallback and fetch updates into `userData`, or
fetch-only into `userData` on first run. Resolve the path in `oui.js` via a
parameter (defaulting to `userData`) so the headless smoke test can pass a temp
path.

### 6. MEDIUM — `scan:device-found` timing and `openPorts` content unspecified

**Location:** §2.2 (`scan:device-found`), §4.5, §6.

**Problem:** The live-progress must-have depends on when `scan:device-found`
fires and what `openPorts` contains. The ping/ARP phase discovers devices before
the per-host port scan. It is unspecified whether `device-found` fires during
discovery (with `openPorts:[]`) and again after each host's port scan (with
populated ports), or only once. If only once and only after the (slow) port
scan, the list updates one host at a time and "live" progress is defeated; if
during discovery, cards show empty ports until a possibly-much-later re-emit.

**Fix:** Specify two emissions per device: first at discovery
`{ device, isNew }` with `openPorts: []` and `isOnline:true`; second after that
device's port scan completes with the same device plus populated `openPorts`.
State that the renderer merges by MAC (updates the existing card), so the card
appears immediately and ports fill in as each host is scanned.

### 7. MEDIUM — CSV escaping/quoting unspecified (malformed CSV risk)

**Location:** §12.

**Problem:** The examples quote only `open_ports` and `banner_info`, but vendor
names (e.g. "Cisco Systems, Inc") and user labels can contain commas, and banner
strings can contain quotes/newlines. With no stated escaping rule, an
implementer following the examples would emit unquoted comma-bearing fields,
producing broken CSV.

**Fix:** Specify RFC 4180 quoting for every field in both Mode A and Mode B:
wrap any field whose value contains a comma, double-quote, or newline in double
quotes, and escape internal double quotes by doubling them. Apply uniformly to
all columns.

### 8. MEDIUM — `device:*` behavior for a valid-but-absent MAC is unspecified

**Location:** §2.1 (`device:detail`/`acknowledge`/`rename`/`delete`), §9 (MAC
validation), §8.

**Problem:** §9 validates MAC *format* (rejecting `INVALID_MAC`) but never
defines behavior when a valid-format MAC is not in the `devices` table.
`device:detail`'s response type is `{ device: Device, ... }` (non-null), implying
existence, but a deleted/stale MAC could be passed. `device:rename/delete/acknowledge`
return `{ ok: boolean }` but it is unstated whether `ok:false` means "not found"
vs "no-op."

**Fix:** Define: `device:detail` rejects with `{ error:'DEVICE_NOT_FOUND' }` for a
valid-format absent MAC; `device:rename`/`delete`/`acknowledge` return `{ ok:false }`
when the MAC is absent (idempotent, not an error). Add a `DEVICE_NOT_FOUND` row to
§8 and a note to §9 that existence is checked after format validation.

### 9. MEDIUM — `top100` preset is truncated; the exact port list is undefined

**Location:** §6.1.

**Problem:** The `top100` array is shown with `/* ... trimmed for brevity */`, so
the exact 100 ports are not specified. An implementer cannot reproduce the
preset that the IPC contract exposes as a selectable option (`portPreset:'top100'`).

**Fix:** Provide the full 100-element array in the design, or pin it to a
canonical citable list (e.g. "the nmap top-100 TCP ports") and state that the
array is built deterministically from that source.

### 10. MEDIUM — "Background scans" operational model unspecified (app must stay open)

**Location:** §10, §1.

**Problem:** The improvement is "scheduled background scans," but §10's
`setInterval` only runs while the Electron app is open. There is no
tray/minimize-to-background mode, and the design does not state that the app
must remain running for scheduled scans to fire. A user who closes the window
reasonably expects "background" scans to continue.

**Fix:** Either add a minimize-to-tray mode so the window can close while the
process (and the interval) keeps running, with the tray icon reflecting scan
state; or explicitly document the limitation: "scheduled scans run only while
NetScope is open; closing the app stops the schedule."

---

## NITs

1. **NIT** — `scan:progress` `total` semantics per phase are unspecified (for
   `'ports'`, is `total` the host count or host×port count?). Define per phase.
2. **NIT** — OUI CSV parser approach is unspecified. A naive `split(',')` breaks
   on the quoted `Organization Address` field (contains commas); header-row
   handling is implied but not stated. Use a header-aware CSV parser or split only
   the first columns needed (`Assignment`, `Organization Name`).
3. **NIT** — Device-detail scan-history diff ("← diff: 22 appeared") computation
   location is unspecified (renderer comparing consecutive `ScanResult.open_ports`,
   or main). State it is renderer-side.
4. **NIT** — TLS `banner_info` format is unspecified (cert subject+issuer string
   layout). Define e.g. `{ "443": "CN=example.com; O=Acme; Issuer=CN=Let's Encrypt R3" }`.
5. **NIT** — `scan:progress` phase enum order (`'arp'` before `'ping'`) does not
   match execution order (ping then ARP). Reorder or note the enum is a set, not a
   sequence.
6. **NIT** — `settings:set` unknown-key handling is unspecified (ignore vs reject).
   State unknown keys are ignored.
7. **NIT** — `export:csv` with a non-existent `scanId` (Mode B) is unspecified
   (empty CSV vs error). State it returns an empty CSV (header only) or rejects
   `EXPORT_NOT_FOUND`.
8. **NIT** — §9 has no enum validation for `scan:start` `portPreset`, and does not
   state that `portRange` is ignored unless `portPreset='custom'`. Add both rules.
9. **NIT** — `scans.new_device_count` and `scans.device_count` computation is not
   specified (count of `isNew` devices; count of `seenMacs` / `scan_results`).
10. **NIT** — Unacknowledged devices re-alert on every scheduled scan until
    manually acknowledged (§3.2), which can be noisy for frequent background scans.
    Consider a "suppress repeat alerts for already-seen unacknowledged devices"
    toggle or auto-ack after N sightings. (Deliberate per §17 #7; noted as product
    UX.)
11. **NIT** — `vendor` is overwritten to `null` on a scan whose OUI lookup yields
    `null` (e.g. OUI file absent this session), losing a previously-known vendor.
    Consider preserving the prior `vendor` when the new lookup is `null`.
