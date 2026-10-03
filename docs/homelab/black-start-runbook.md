# Home Lab Black-Start Runbook

How to bring the lab back after a power outage without being there, and how
to keep a Claude Code beachhead alive so the next outage is a one-liner.

Hosts: `upserver` (always-on, Stash, cloudflared), `mini`, `micro`,
`workstation`, Carrie's laptop. Domain: `westergard.vip` (Cloudflare).
Only `stash.westergard.vip` has a public DNS record; everything else is
Tailscale-only.

## 0. Before the next outage (do once)

| Layer | Setting | Why |
|---|---|---|
| BIOS | Restore on AC Power Loss = **Power On** (upserver, mini, micro, workstation) | Box boots itself. Nothing below matters without this. |
| BIOS | Wake-on-LAN = Enabled | Lets upserver wake the others. |
| OS | Linux: `ethtool -s eth0 wol g` in a systemd unit. Windows: disable Fast Startup, untick NIC power saving. | WoL survives reboot. |
| Network | Tailscale on every host, running as a service | IP changes and CGNAT stop mattering. |
| Laptops | Lid close on AC = do nothing, sleep on AC = never | A sleeping laptop is an offline laptop. |
| UPS | NUT on upserver, graceful shutdown | No trashed filesystems. |
| Optional | Smart plug (Shelly/Kasa) on upserver + BIOS Power On | Phone-driven hard power cycle. |
| Optional | JetKVM / PiKVM on upserver | Full out-of-band, reinstall-from-anywhere tier. |

## 1. After an outage: triage from your phone

1. Tailscale app: which nodes are green. That is the liveness board.
2. `https://stash.westergard.vip`: loads = upserver + cloudflared up. Cloudflare
   5xx (521/522/530) = Cloudflare fine, upserver or cloudflared down.
   Cloudflare 1020 = WAF geo rule blocked you; add your current state to the
   rule (Security > WAF > Custom rules).
3. Nothing green at all: ISP or router. Someone at the house power-cycles the
   modem and router first, then upserver.

## 2. Wake the rest from upserver

```bash
ssh upserver
tailscale status
wakeonlan <workstation-mac>
wakeonlan <mini-mac>
wakeonlan <micro-mac>
sleep 60; for h in workstation mini micro; do ping -c1 -W2 $h >/dev/null && echo "$h up" || echo "$h DOWN"; done
```

Fill in MACs here once you have them:

| Host | MAC | Notes |
|---|---|---|
| workstation | `xx:xx:xx:xx:xx:xx` | |
| mini | `xx:xx:xx:xx:xx:xx` | |
| micro | `xx:xx:xx:xx:xx:xx` | |

## 3. Carrie's laptop (Windows): reachable, not hosting

Do not run a persistent Claude session on it. Make it reachable and let
upserver do the hop. No WSL needed.

```powershell
# admin PowerShell, once
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Set-Service sshd -StartupType Automatic; Start-Service sshd
powercfg /change standby-timeout-ac 0
powercfg /setacvalueindex SCHEME_CURRENT SUB_BUTTONS LIDACTION 0
powercfg /setactive SCHEME_CURRENT
winget install tailscale.tailscale; tailscale up
```

Then from upserver: `ssh carrie-laptop`.

## 4. The Claude beachhead (persistent Remote Control on upserver)

Goal: after any reboot, a Claude Code session is already waiting in the Code
tab of the phone app.

First run, interactively, so the trust/enable prompts are answered once:

```bash
mkdir -p ~/lab && cd ~/lab
claude remote-control   # answer y, y; confirm it prints the "live" line; Ctrl+C
```

Then install the user unit from `claude-rc.service` in this folder:

```bash
mkdir -p ~/.config/systemd/user
cp claude-rc.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now claude-rc
loginctl enable-linger $USER     # start at boot with no login
systemctl --user status claude-rc
```

Windows equivalent (workstation): Task Scheduler, trigger At log on, action
`claude.exe remote-control`, Start in `C:\IT`, "Run whether user is logged on
or not", restart on failure every 1 min x3. Needs auto-logon (netplwiz).

Known gotchas:

- Login token in `~/.claude` can expire after weeks. Symptom: session stops
  appearing. Fix: SSH in, run `claude` once, re-auth.
- If `claude remote-control` sits on a blank line after `y`, the box cannot
  reach Anthropic's relay. Plain `claude` (local, no relay) may still work.
  Check for a filtering proxy or DNS filter on that network.

## 5. Working from a laptop that is already on the tailnet

```powershell
irm https://claude.ai/install.ps1 | iex      # or: winget install OpenJS.NodeJS.LTS; npm i -g @anthropic-ai/claude-code
tailscale status                             # confirm lab nodes are listed
ssh-keygen -t ed25519
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh <user>@upserver "mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"
mkdir C:\IT -Force; cd C:\IT
claude
```

Then ask it: "ssh upserver, tailscale status, ping mini micro workstation,
wakeonlan workstation."

## 6. What the cloud session (claude.ai/code) can and cannot do

The cloud container has no LAN, no Tailscale, no SSH keys, and its egress is
allowlisted. It cannot reach any lab host, and it cannot use credentials
pasted into chat. To let it probe Stash or call the Cloudflare API, edit the
cloud environment (claude.ai/code, environment menu in the title bar, Edit):

- Network access: add `*.westergard.vip` and `api.cloudflare.com`.
- Secrets: `CLOUDFLARE_API_TOKEN` scoped to Zone:WAF:Edit + Zone:DNS:Read.

Even then it only gets HTTPS. For SSH, WoL and LAN work, use the beachhead.
