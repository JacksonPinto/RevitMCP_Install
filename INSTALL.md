# Revit MCP Server — Installation Guide

Connect Claude (or any MCP client) to a **live Autodesk Revit model**. Tested with
**Revit 2025 / 2026**, **pyRevit 6.4.0+**, **Python 3.12.10**, Windows 11.

---

## How it fits together (read this first)

There are **two pieces on two (or one) machines**:

```
Claude Desktop  --stdio-->  MCP server (revit_mcp_server.py)  --HTTP/LAN-->  pyRevit Routes (inside Revit)
   (your PC)                 (same PC as Claude Desktop)                       (the Revit PC, Windows)
```

- **pyRevit Routes** = the RevitMCP extension, runs **inside Revit** on the Windows PC. (Part A)
- **MCP server** = a small Python script Claude Desktop launches. It runs on **whatever machine Claude Desktop is on** and calls Revit over the network. (Part B)

Two valid setups:

- **Single PC** — Claude Desktop + Revit on the same Windows PC → MCP server talks to `127.0.0.1`.
- **Split** — Claude Desktop on another machine (e.g. a Mac) → MCP server talks to the Revit PC's **LAN IP** (e.g. `192.168.0.171`).

---

> **Before you begin:** the RevitMCP extension is distributed from a private repository.
> Request an access token from **Jackson Pinto — jackson.pinto@wwt.com** before starting
> the extension install in Part A2.

# Part A — Revit PC (Windows)

## A1. Install pyRevit (6.4.0 or newer — required for Revit 2026)

1. Download the signed installer from the releases page:
   **https://github.com/pyrevitlabs/pyRevit/releases**
   (Pick the latest `pyRevit_<version>_signed.exe`. Do **not** use 4.8 — it doesn't support Revit 2025/2026.)
2. Run the installer, then open Revit once and confirm the **pyRevit** ribbon tab appears.

## A2. Install the RevitMCP extension (direct from Git)

> **Access token required.** The extension repository is private. **Before you start**,
> request a GitHub access token (read‑only, scoped to this repo):
>
> **Contact Jackson Pinto — jackson.pinto@wwt.com**
>
> You'll paste that token into the **Token** field of the Extension Manager (below). Do
> not commit or share the token; treat it like a password.

1. In Revit: **pyRevit tab → Extensions** (Extension Manager).
2. In the **Git information** box at the bottom:
   - **Git URL:**
     ```
     https://github.com/JacksonPinto/RevitMCP.extension.git
     ```
   - **Path:** leave as default (`%APPDATA%\pyRevit\Extensions`)
   - **Token:** paste the access token you received from Jackson Pinto
   - Click **Add and install**.

   It clones to: `%APPDATA%\pyRevit\Extensions\RevitMCP.extension.extension\`
3. **pyRevit tab → Reload** (or restart Revit).

**To update later** (cmd) — include the token in the URL for the private repo:
```cmd
cd "%APPDATA%\pyRevit\Extensions\RevitMCP.extension.extension"
git pull https://YOUR_TOKEN@github.com/JacksonPinto/RevitMCP.extension.git main
```
Then Reload pyRevit (restart Revit if a route change doesn't take effect).
(If the clone was made with the token, a plain `git pull origin main` also works.)

## A3. Turn on the Routes server

1. **pyRevit tab → Settings → Routes** section: enable **Routes Server**, set **port `48884`**.
2. To allow access from another machine (split setup), set the host to **`0.0.0.0`** (all interfaces). For single‑PC use, `127.0.0.1` is fine.
3. **Save Settings**, then **Reload**. Open any Revit model.

## A4. Allow the port through Windows Firewall (only needed for the split/LAN setup)

Open **cmd as Administrator** and run:
```cmd
netsh advfirewall firewall add rule name="pyRevit Routes 48884" dir=in action=allow protocol=TCP localport=48884
```
(Equivalent PowerShell, run as admin:
`New-NetFirewallRule -DisplayName "pyRevit Routes 48884" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 48884 -Profile Any`)

## A5. Verify (on the Revit PC)

Open a model, then in a browser go to:
```
http://localhost:48884/revit/ping
```
Expected: `{"status": "ok", "revit_version": "2026", ...}`. In the pyRevit output you should see
`[RevitMCP] Registered 13/13 route modules`.

Find the PC's LAN IP for the split setup (cmd): `ipconfig` → **IPv4 Address**.

---

# Part B — MCP server + Claude Desktop

Do this on the machine where **Claude Desktop** runs (the Windows PC for single‑PC, or the Mac for split).

## B1. Install Python 3.12.10

- **Windows installer:** https://www.python.org/ftp/python/3.12.10/python-3.12.10-amd64.exe
  (release page: https://www.python.org/downloads/release/python-31210/)
  During setup, tick **"Add python.exe to PATH"**.
- **macOS installer:** https://www.python.org/ftp/python/3.12.10/python-3.12.10-macos11.pkg

Verify (cmd on Windows):
```cmd
py -3.12 --version
```
(macOS: `python3 --version` → must be 3.10+.)

## B2. Get the MCP server file

`revit_mcp_server.py` ships inside the extension repo, so on the **Revit PC** it's already at:
```
%APPDATA%\pyRevit\Extensions\RevitMCP.extension.extension\revit_mcp_server.py
```
For a different machine (e.g. the Mac), copy that file over, or download it from the repo. A clean place to keep it:
```cmd
mkdir "%USERPROFILE%\revit-mcp"
copy "%APPDATA%\pyRevit\Extensions\RevitMCP.extension.extension\revit_mcp_server.py" "%USERPROFILE%\revit-mcp\"
```

## B3. Install the Python dependency (cmd)

The server uses only the Python standard library plus the official **`mcp`** package:
```cmd
py -3.12 -m pip install --upgrade pip
py -3.12 -m pip install --upgrade "mcp>=1.2.0"
```
Find the exact python.exe path (you'll need it for the config):
```cmd
py -3.12 -c "import sys; print(sys.executable)"
```
(macOS: `python3 -m pip install --upgrade "mcp>=1.2.0"` and `which python3`.)

## B4. Self‑test the bridge (proves it reaches Revit — no Claude needed)

Windows, single‑PC:
```cmd
set REVIT_HOST=127.0.0.1
py -3.12 "%USERPROFILE%\revit-mcp\revit_mcp_server.py" --selftest
```
Split (from the Mac, pointing at the Revit PC's IP):
```bash
REVIT_HOST=192.168.0.171 python3 ~/revit-mcp/revit_mcp_server.py --selftest
```
Expected: the ping JSON. If you get "Cannot reach Revit", revisit Part A5 / the firewall / the IP.

## B5. Configure Claude Desktop

Open Claude Desktop → **Settings → Developer → Edit Config**. This opens `claude_desktop_config.json`:
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`

Add a `revit` entry under `mcpServers` (keep any servers you already have).

**Windows (single PC) sample:**
```json
{
  "mcpServers": {
    "revit": {
      "command": "C:\\Users\\YOURNAME\\AppData\\Local\\Programs\\Python\\Python312\\python.exe",
      "args": ["C:\\Users\\YOURNAME\\revit-mcp\\revit_mcp_server.py"],
      "env": {
        "REVIT_HOST": "127.0.0.1",
        "REVIT_PORT": "48884"
      }
    }
  }
}
```

**macOS (split — Claude on Mac, Revit on the Windows PC) sample:**
```json
{
  "mcpServers": {
    "revit": {
      "command": "/Library/Frameworks/Python.framework/Versions/3.12/bin/python3",
      "args": ["/Users/YOURNAME/revit-mcp/revit_mcp_server.py"],
      "env": {
        "REVIT_HOST": "192.168.0.171",
        "REVIT_PORT": "48884"
      }
    }
  }
}
```

Rules that avoid 90% of problems:
- Use the **absolute path** to the python that you ran `pip install mcp` with (from B3). Claude Desktop launches with a minimal PATH — bare `python`/`python3` often fails.
- `REVIT_HOST` = `127.0.0.1` on the same PC, or the Revit PC's LAN IPv4 for the split setup.
- Optional: `"REVIT_TIMEOUT": "60"` in `env` if the model is large/cloud and requests time out while it loads.

## B6. Restart Claude Desktop and test

Fully quit Claude Desktop (Windows: system tray → Quit; macOS: Cmd‑Q) and reopen. Then ask:
```
ping revit
```
You should get the Revit version + open document. Try also: *"get the Revit project info"*, *"list the levels"*, *"what's in space X"*.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Claude shows **"revit: Server disconnected"** | Wrong Python in `command`, or `mcp` not installed for it. Use the B3 `sys.executable` path; confirm Python ≥ 3.10. |
| **Cannot reach Revit / timeout** | Model not open, Revit still loading (esp. cloud/large models), or firewall/IP. Confirm Part A5 works locally first. |
| Works on the Revit PC (`localhost`) but not from another machine | Routes host must be `0.0.0.0` (A3) **and** firewall rule added (A4); confirm the Revit PC's current IPv4 with `ipconfig`. |
| **RouteHandlerNotDefined** on a route | Extension is out of date — run the A2 update (`git pull`) and reload/restart Revit. |
| IP keeps changing each session | Set a **DHCP reservation** for the Revit PC in your router so its IP is fixed; then `REVIT_HOST` never changes. |
| A route returns `{... "truncated": true}` | The list was capped; narrow the query (by active view or by room/space). |

## Notes

- The routes are currently **unauthenticated**. On a LAN (`0.0.0.0`), any machine on the network can call them, including write operations. Keep it on `127.0.0.1`, use a trusted network, or add token auth before production use.
- Optional companion: install the **`revit-mcp` skill** so Claude knows how to use all the tools on any model.

## Quick reference — Windows cmd, start to finish (single PC)

```cmd
:: 1) after installing pyRevit + adding the extension via the Extension Manager, and
::    enabling Routes on port 48884, verify locally in a browser: http://localhost:48884/revit/ping

:: 2) Python deps
py -3.12 -m pip install --upgrade pip
py -3.12 -m pip install --upgrade "mcp>=1.2.0"
py -3.12 -c "import sys; print(sys.executable)"

:: 3) copy the server file somewhere stable
mkdir "%USERPROFILE%\revit-mcp"
copy "%APPDATA%\pyRevit\Extensions\RevitMCP.extension.extension\revit_mcp_server.py" "%USERPROFILE%\revit-mcp\"

:: 4) self-test
set REVIT_HOST=127.0.0.1
py -3.12 "%USERPROFILE%\revit-mcp\revit_mcp_server.py" --selftest

:: 5) edit %APPDATA%\Claude\claude_desktop_config.json (see B5), fully restart Claude Desktop, then: "ping revit"
```
