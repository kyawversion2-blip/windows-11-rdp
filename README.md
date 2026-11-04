# Windows 11 RDP on a Free GitHub Actions VM

A self-contained setup that spins up a Windows 11 VM (GitHub Actions `windows-latest` runner), installs **ngrok**, opens port **3389** (RDP), forwards it through an ngrok TCP tunnel, and prints the live connection endpoint.

There are two variants in this repo:

| File | Method | Notes |
|------|--------|-------|
| `.github/workflows/rdp-ngrok.yml` | ngrok TCP tunnel | classic, simple; requires a free ngrok authtoken |
| `.github/workflows/rdp-tailscale.yml` | Tailscale VPN | newer, more secure; requires a Tailscale auth key (free SSO) |

Both expose the **same** Windows credentials:

| Username | Password |
|----------|----------|
| `runneradmin` | `P@ssw0rd123!` (or value of `PASSWORD` secret) |

---

## 1. Environment Setup (GitHub Actions workflow)

### 1.1 ngrok variant — `.github/workflows/rdp-ngrok.yml`

This workflow downloads the latest ngrok v3 Windows client, registers your
auth token, sets up a local user, opens the RDP firewall rule, and starts a
**TCP** tunnel that forwards the public endpoint to the runner's 3389 port.

The final step is a `pause` / sleep loop so the job never terminates while the
tunnel is alive (the runner is auto-killed by GitHub after 6 hours).

### 1.2 Tailscale variant — `.github/workflows/rdp-tailscale.yml`

Uses `tailscale/tailscale-action` to bring up a Tailscale node in an **ephemeral
auth key** with `ssh: true`. It then waits for the tunnel (`100.x.x.x`) and
exposes RDP directly over the tailnet — no public port forwarding needed.

### Required secrets

Set these in your GitHub repo → Settings → Secrets and variables → Actions →

| Secret name | Required for | Value |
|-------------|--------------|-------|
| `NGROK_AUTH_TOKEN` | ngrok variant | from https://dashboard.ngrok.com/get-started/your-authtoken |
| `TAILSCALE_AUTH_KEY` | Tailscale variant | generate an **ephemeral** key in https://login.tailscale.com/admin/settings/keys |
| `PASSWORD` (optional) | both | overrides the default `runneradmin` password |

### Helper script — `scripts/setup-rdp.ps1`

A reusable PowerShell snippet invoked from the workflow that:

1. Enables the Windows Remote Desktop listener (`fDenyTSConnections = 0`).
2. Enables the "Remote Desktop" Windows Defender Firewall rule group.
3. Disables the requirement for **NLA** (Network Level Authentication) so that
   the Microsoft Store Remote Desktop client on phones / older OS versions
   works out-of-the-box.
4. Sets a known password on the local `runneradmin` account (defaults to
   `Env:PASSWORD` or `P@ssw0rd123!`).
5. Writes the connection info to `%USERPROFILE%\rdp-info.txt`.

---

## 2. Ngrok / Tunneling Integration

### ngrok (simplest)

```powershell
.\ngrok.exe authtoken "$Env:NGROK_AUTH_TOKEN"
.\ngrok.exe tcp 3389 --region us --log=stdout
```

The first line of ngrok output looks like:

```
Forwarding tcp://1.tcp.us.ngrok.io:XXXXX -> localhost:3389
```

gives you the **RDP connection address**: `1.tcp.us.ngrok.io:XXXXX`

### Tailscale (recommended)

```yaml
- uses: tailscale/tailscale-action@v3
  with:
    authKey: '${{ secrets.TAILSCALE_AUTH_KEY }}'
    ephemeral: true
```

The runner advertises a Tailscale IP (`100.x.x.x`). You RDP to that IP:3389.

---

## 3. Credentials & Access

| Field | Value |
|-------|-------|
| **Username** | `runneradmin` (built-in GitHub Actions admin) |
| **Password** | `P@ssw0rd123!` — or the value of the `PASSWORD` secret |
| **RDP server** | ngrok: `1.tcp.us.ngrok.io:XXXXX` OR Tailscale: `100.x.x.x:3389` |

---

## 4. Step-by-Step Guide (Myanmar / Burmese)

### 4.1 Requirements

| Need | Requirement |
|------|-------------|
| GitHub account | GitHub username/password |
| Ngrok authtoken | https://dashboard.ngrok.com → sign-up → authtoken |
| Tailscale auth key | (optional) https://login.tailscale.com/admin/settings/keys |

### 4.2 Ngrok Version

1. **Create Repository**
   GitHub → New repository → Create
   ```
   https://github.com/<yourname>/windows-11-rdp.git
   ```

2. **Create Secret**
   GitHub → Settings → Secrets and variables → Actions → New repository secret
   ```
   Name: NGROK_AUTH_TOKEN
   Value: <dashboard.ngrok.com authtoken>
   ```

3. **Run Workflow**
   - Actions tab → "Windows 11 RDP (ngrok)" → Run workflow
   - OR push and run workflow

4. **RDP Connection Info**
   - Actions → RDP run → Log → read `RDP_CONNECTION_INFO`
   - Get the RDP server endpoint (e.g., `tcp://1.tcp.us.ngrok.io:57584`)

5. **Remote Desktop**
   - Windows → mstsc.exe
   - Computer: `1.tcp.us.ngrok.io:57584`
   - Username: `runneradmin`
   - Password: `P@ssw0rd123!` (or `PASSWORD` secret)

6. **Connect**
   - Windows 11 Desktop → RDP client → connect

### 4.3 Tailscale Version

1. **Repository + Tailscale secret**
   GitHub → Settings → Secrets → New repository secret
   ```
   Name: TAILSCALE_AUTH_KEY
   Value: <Tailscale admin console ephemeral key>
   ```

2. **Run Actions → "Windows 11 RDP (Tailscale)"**

3. **Get Tailscale IP** (`100.x.x.x`) → use as `100.x.x.x:3389`

4. **mstsc.exe → Computer: IP**

### 4.4 Time Limit

| Time | Note |
|------|------|
| 6 hours | GitHub runner auto-terminates |

### 4.5 Credentials

| Field | Value |
|-------|-------|
| Username | `runneradmin` |
| Password | `P@ssw0rd123!` |
| RDP Port | TCP/3389 |

### 4.6 Final Steps

1. Create repo with ngrok/Tailscale setup
2. Run Actions
3. Connect via RDP
