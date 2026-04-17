# CyberOne GVM Port Lists

Vendored port list definitions for GVM/OpenVAS (Greenbone Community Edition).
These XML files are applied to the agent's local GVM instance so every customer
agent exposes the same scan-depth tiers regardless of GVM upstream defaults.

## Why vendor these?

Greenbone Community Edition only ships 3 port lists:

| UUID | Name | TCP | UDP |
|------|------|-----|-----|
| `33d0cd82-57c6-11e1-8ed1-406186ea4fc5` | All IANA assigned TCP | 5,836 | 0 |
| `730ef368-57e2-11e1-a90f-406186ea4fc5` | All TCP and Nmap top 100 UDP | 65,535 | 100 |
| `4a4717fe-57d2-11e1-9a26-406186ea4fc5` | All IANA assigned TCP and UDP | 5,836 | 5,482 |

The built-in "All IANA TCP and UDP" list only covers 5,836 TCP ports — fewer
than the "All TCP + top 100 UDP" list. On a 2026-04-12 external CHA scan this
gap caused 34 of 37 hosts to be missed (FortiGate VPN endpoints that only
answer on UDP 500 were invisible to the default alive detection + port list
combination).

These vendored lists close that gap by guaranteeing full 65,535 TCP coverage
at every depth, with progressively wider UDP coverage.

## What's here

| File | Name in GVM | TCP | UDP | Use |
|------|-------------|-----|-----|-----|
| `cyberone-standard.xml` | CyberOne Standard - All TCP + Top 100 UDP | 65,535 | 100 | `scan_depth: standard` |
| `cyberone-comprehensive.xml` | CyberOne Comprehensive - All TCP + Top 500 UDP | 65,535 | ~1,000 | `scan_depth: comprehensive` |
| `cyberone-maximum.xml` | CyberOne Maximum - All TCP + All UDP | 65,535 | 65,535 | `scan_depth: maximum` |

Note: the "Top 500 UDP" string contains multi-port ranges that expand to
~1,000 distinct UDP ports. This is intentional — broader comprehensive
coverage is desirable.

## How they're used

1. `install-vulnassess.sh` clones this repo to `/opt/cyberone-vulnassess-vendor/`
   on the agent machine during install.
2. At scan time, the agent's `scanner.js` calls `ensureCyberOnePortLists()`:
   - Queries local GVM via `<get_port_lists/>`
   - Looks up each CyberOne list by name
   - If any are missing, reads the XML from `/opt/cyberone-vulnassess-vendor/gvm-port-lists/`
     and submits `<create_port_list>` to GVM
   - Caches the resulting UUIDs for the daemon's lifetime
3. Scanner falls back to inline XML (embedded in `scanner.js`) if this
   vendored directory isn't present — belt-and-braces so the agent always
   works, even if the vendor clone failed.

## Updating a port list

Port lists in GVM are immutable after creation (GVM doesn't support editing
ports on an existing list). To "update" a list:

1. Edit the XML file in this directory.
2. Change the `<name>` to include a version suffix (e.g. add ` v2` to the name)
   so the old list can coexist with the new one.
3. Commit and push to `origin/www`.
4. Bump the agent version, rebuild the agent package.
5. On next agent update, `scanner.js` will see that the new-named list is
   missing and create it. Old tasks still reference the old list by UUID;
   new tasks get the new list.

Alternatively, the agent could delete the old list before creating the new
one, but this would orphan any in-progress scans — not recommended.

## Repo layout

```
/Programs/vulnassess/                        (local mirror on security server)
https://github.com/tobyarnett/vulnassess.git (remote, branch: www)
├── README.md
├── LICENSE
└── gvm-port-lists/
    ├── README.md                    (this file)
    ├── cyberone-standard.xml
    ├── cyberone-comprehensive.xml
    └── cyberone-maximum.xml
```

## Version history

| Date | Change |
|------|--------|
| 2026-04-17 | Initial vendoring — 3 CyberOne port lists |
