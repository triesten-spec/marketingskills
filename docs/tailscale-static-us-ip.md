# Solutions R Us — Tailscale Static U.S. IP Implementation Plan

**Audience:** implementing developer
**Status:** ready to execute
**Owner:** Triesten (tailnet admin) — pilot with Louis + one agent before general rollout

---

## Objective

Give authorized remote agents a secure U.S. internet route through the existing Miami VPS so Atomicity and approved business systems see one consistent public IP:

**104.156.244.86 — Miami, Florida**

## Architecture

```
Agent Windows PC → Tailscale encrypted tunnel → "mei" Miami VPS exit node → Atomicity / internet
```

The VPS becomes the agency's internet gateway. Agents must use individual Tailscale identities — **never** Triesten's administrator login.

---

## Read this before you start

Three items in the original plan need adjusting. They are folded into the phases below, but call them out here so they are not missed:

1. **Do not run a bare `sudo tailscale up` on the VPS.** `tailscale up` resets every unspecified flag to its default, which will silently undo existing configuration on a node that is already connected. `tailscale set` changes only the flag you pass. Use `set` — see Phase 2. ([tailscale set](https://linuxcommandlibrary.com/man/tailscale-set), [tailscale up](https://tailscale.com/docs/reference/tailscale-cli/up))
2. **Use grants with `via`, not a broad `autogroup:internet` accept.** A plain internet grant lets agents use *any* exit node they can see. The `via` field pins them to the approved node. See Phase 3.
3. **Windows has no per-app split tunneling.** Once an agent selects the exit node, *all* traffic goes through it, including MicroSIP/D1AL SIP and RTP. This is the crux of the Phase 6 voice risk — there is no setting that routes only the browser.
4. **D1al confirmed the SIP/login risk is resolved, but shifts responsibility onto the Tailscale ACL.** D1al gates access with a manually-managed firewall IP allowlist (no fraud/IP scoring) — once 104.156.244.86 is added, *any* traffic arriving from that IP is granted access immediately. That means D1al is no longer the control that decides which agents can reach it; the Phase 3 policy grant is. Test that grant thoroughly before the IP goes on D1al's allowlist.

---

## Phase 1 — Validate the VPS

1. Confirm the VPS public IP is **104.156.244.86**.
2. Confirm the IP will remain assigned while the VPS is active.
3. Obtain a reserved/static IP from the hosting provider if the VPS could be rebuilt or reassigned. On most providers the primary IP survives reboots and stops but is **released on destroy/rebuild** — a reserved IP product must be explicitly attached to survive that.
4. Confirm adequate bandwidth and monitor CPU, memory, packet loss, latency, and monthly transfer limits. Every agent's full browsing volume now lands on this VPS's transfer allowance.
5. Preserve existing firewall and Tailscale settings before making changes.
6. **Request D1al add 104.156.244.86 to their firewall allowlist.** Confirmed with D1al's dev team: they allow only specified IPs onto their iptables allowlist (no IP scoring), and once an IP is added, all traffic from it is granted access immediately. Send them the IP and get written confirmation it's live before Phase 5 testing — this is a hard dependency for the pilot, not something to discover mid-test-call. See the access-control note below.

Capture the current state so you can roll back:

```bash
tailscale status
tailscale ip -4
tailscale debug prefs > ~/tailscale-prefs-backup.json
sudo iptables-save > ~/iptables-backup.rules
curl -4 -s https://ifconfig.me    # expect 104.156.244.86
```

---

## Phase 2 — Configure the exit node

On the "mei" Linux VPS:

```bash
sudo tee /etc/sysctl.d/99-tailscale-forwarding.conf >/dev/null <<'EOF'
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF

sudo /usr/sbin/sysctl --system
```

Verify forwarding actually took effect before continuing:

```bash
sysctl net.ipv4.ip_forward net.ipv6.conf.all.forwarding
# both must report = 1
```

Then advertise the exit node:

```bash
sudo tailscale set --advertise-exit-node
```

**Use `tailscale set`, not `tailscale up`.** The node is already connected; `tailscale set` changes only that one preference and leaves the rest of the configuration alone. Run `sudo tailscale up` **only** if `tailscale status` shows the node is logged out — and if you must, pass every flag the node already had.

Confirm it advertised:

```bash
tailscale status --json | grep -i exitnode
```

If `sysctl` is installed somewhere else, use:

```bash
command -v sysctl
```

### Throughput tuning (recommended)

Tailscale 1.54+ on Linux kernel 6.2+ supports a receive offload that materially raises forwarded-traffic throughput. This matters here because all agent traffic is forwarded traffic. ([Performance best practices](https://tailscale.com/docs/reference/best-practices/performance))

```bash
sudo apt install -y ethtool
NETDEV=$(ip -o route get 8.8.8.8 | cut -f 5 -d " ")
sudo ethtool -K "$NETDEV" rx-udp-gro-forwarding on rx-gro-list off
```

`ethtool` settings do not survive a reboot. Make them persistent with a systemd oneshot that runs before `tailscaled`, or a `/etc/networkd-dispatcher/routable.d/50-tailscale` script on distros using networkd-dispatcher.

### Approve in the admin console

1. Open https://login.tailscale.com/admin/machines
2. Locate **mei**.
3. Open **Edit route settings**.
4. Enable **Use as exit node**.
5. Confirm "mei" appears as an available exit node on an authorized test computer.

While you are there, disable key expiry on **mei**. If its key expires the exit node drops and every agent loses internet at once.

---

## Phase 3 — Lock down access

Before onboarding agents:

1. Review the existing Tailscale access policy — **do not overwrite it blindly**.
2. Create an **SRS Agents** group.
3. Tag the VPS as an agency exit node.
4. Permit SRS agents to use `autogroup:internet` through the approved exit node.
5. Do **not** give agents access to Mei, Eva, personal computers, servers, SSH, databases, or other tailnet devices.
6. Agents receive the basic **Member** role only.
7. Enable device approval if available.
8. Disable unnecessary file sharing and administrative privileges.
9. Set a removal process for terminated or inactive agents.

### Policy file additions

Add these blocks to the **existing** policy file. Do not replace the file. Use the admin console's policy editor, which previews the diff and refuses syntactically invalid policies.

```jsonc
{
  "tagOwners": {
    "tag:srs-exit-node": ["autogroup:admin"]
  },

  "groups": {
    "group:srs-agents": [
      "paraanjoy1991@gmail.com",   // Joy
      "rymajhoy77@gmail.com",      // Mary
      "coliflores00@gmail.com"     // Lucas
    ]
  },

  "grants": [
    {
      // Agents may reach the internet, but ONLY via the tagged Miami node.
      "src": ["group:srs-agents"],
      "dst": ["autogroup:internet"],
      "via": ["tag:srs-exit-node"],
      "ip":  ["*"]
    }
  ]
}
```

The `via` field is what confines agents to the approved exit node — without it, an internet grant lets them route through any exit node in the tailnet. ([Grant examples](https://tailscale.com/docs/reference/examples/grants), [Policy file syntax](https://tailscale.com/docs/reference/syntax/policy-file))

Note that `group:srs-agents` gets **no** rule granting access to any tailnet device. That absence is the point — it satisfies requirement 5. Confirm no pre-existing broad rule (for example a catch-all `"src": ["autogroup:member"], "dst": ["*:*"]`) already grants them more than intended; if one exists, it must be narrowed or the agents excluded from it.

Apply the tag to the VPS from the admin console (Machines → mei → Edit machine settings), or on the host:

```bash
sudo tailscale set --advertise-tags=tag:srs-exit-node
```

Tagging a device transfers ownership from the user who authenticated it to the tag. Re-authentication may be required, and the device's ACL identity changes — this is why the policy must be tested before it is relied on. ([Tags](https://tailscale.com/docs/features/tags))

The developer must test the final access policy before deployment, because replacing the existing policy incorrectly could disrupt current Tailscale access.

**Verification:** use the admin console's access-rule tester to confirm, for a test agent account, that internet access via `tag:srs-exit-node` is permitted and that Mei, Eva, and any SSH/database host are denied.

---

## Phase 4 — Agent onboarding

For every agent:

1. Invite their individual work email from **Tailscale Admin → Users → Invite external users**.
2. Install Tailscale from https://tailscale.com/download/windows
3. Accept the invitation using the invited email.
4. Open Tailscale from the Windows system tray.
5. Select **Exit Nodes → mei**.
6. Keep Tailscale connected during approved work hours.
7. Do not share accounts or invitation links.

If an agent needs their local printer or LAN devices while connected, enable **Allow local network access** in the tray menu (CLI equivalent: `--exit-node-allow-lan-access`). Without it, the exit node captures LAN-bound traffic too.

Add the agent's account to `group:srs-agents` in the policy file before they connect, or the exit node will not be selectable for them.

---

## Phase 5 — Required testing

Pilot with **Louis and one agent** before deploying to everyone.

Confirm:

- [ ] https://whatismyip.com shows **104.156.244.86**
- [ ] Location shows Miami, Florida
- [ ] Atomicity login succeeds
- [ ] Accounts load normally
- [ ] Outbound calls connect
- [ ] Two-way audio works
- [ ] Caller ID is correct
- [ ] Recordings are saved
- [ ] Dispositions attach to the correct account
- [ ] No material audio delay, jitter, or dropped calls
- [ ] Payment and client portals do not trigger security blocks
- [ ] Disconnecting the exit node returns the agent to their normal public IP

**Run at least 10 test calls before approval.**

One failure mode to watch for specifically:

- **Datacenter IP reputation on other portals.** Fraud and bot detection on payment and client-facing portals (other than D1al) can score datacenter IP ranges more harshly than residential ones. A block here is a property of the IP, not a misconfiguration, and may need to be resolved with the relevant vendor via allowlisting, same as was done with D1al.

D1al's SIP/login IP restriction is a known quantity, not a risk to test for: D1al confirmed they gate access via a manual firewall allowlist with no IP scoring, so once 104.156.244.86 is added (Phase 1, item 6) and confirmed live, D1al access will work. If it doesn't, the allowlist entry is the first thing to check, not the SIP client config.

---

## Phase 6 — Voice-risk decision

The exit node routes nearly all Windows internet traffic, including MicroSIP/D1AL. This may increase VoIP latency. Windows offers no per-application split tunneling, so this is all-or-nothing per agent.

This phase is about **latency, jitter, and call quality** now that D1al's access-control risk is resolved (Phase 1, item 6) — not about whether D1al will accept the connection at all.

### Acceptance targets

| Metric | Target |
|---|---|
| Latency | preferably below 150 ms |
| Jitter | below 30 ms |
| Packet loss | below 1% |
| One-way audio | none |
| SIP registration failures | no repeated failures |
| Dropped calls | no material increase |

Measure from an agent machine with the exit node active:

```powershell
tailscale netcheck
tailscale ping --until-direct mei
```

`netcheck` reports latency to Tailscale's relays; a direct connection to **mei** (rather than a DERP relay) is what keeps added latency low. If `tailscale ping` never reaches "direct," traffic is being relayed and voice quality will suffer — investigate NAT/firewall before rejecting the design.

### If voice performance fails

Do **not** force the full exit node into production. Use one of these alternatives:

1. **Route only Atomicity/browser traffic through a managed proxy.** Configure the U.S. proxy in the browser (or a browser profile) so Atomicity presents the Miami IP while MicroSIP/D1AL continues to use the agent's normal connection. This is the closest match to the original objective at the lowest voice risk.
2. **Run Atomicity in a remote session on the Miami VPS.** Agents connect over RDP; only screen and input traffic crosses the tunnel, and voice stays entirely local. Highest isolation, highest VPS resource cost.
3. **Dedicated browser machine per agent** (small VM or second profile) with Tailscale and the exit node enabled, while the voice softphone runs on the untunneled host.
4. **Partial rollout.** Keep the exit node for non-calling roles where voice quality is not a factor, and use option 1 or 2 for calling agents.

> The alternatives beyond item 1 are completions — the source plan was cut off mid-list. Confirm the intended list with Triesten before treating options 2–4 as approved.

---

## Rollback

If anything breaks, in this order:

```bash
# On an agent machine — restore normal routing immediately
tailscale set --exit-node=

# On the VPS — stop advertising
sudo tailscale set --advertise-exit-node=false

# Revert the policy file via the admin console's version history
```

The policy file keeps a revision history in the admin console; restore the prior revision rather than hand-editing back.

---

## Sources

- [Exit nodes (route all traffic) · Tailscale Docs](https://tailscale.com/docs/features/exit-nodes)
- [Use exit nodes · Tailscale Docs](https://tailscale.com/docs/features/exit-nodes/how-to/setup)
- [Grant examples · Tailscale Docs](https://tailscale.com/docs/reference/examples/grants)
- [Syntax reference for the tailnet policy file · Tailscale Docs](https://tailscale.com/docs/reference/syntax/policy-file)
- [Group devices with tags · Tailscale Docs](https://tailscale.com/docs/features/tags)
- [Performance best practices · Tailscale Docs](https://tailscale.com/docs/reference/best-practices/performance)
- [tailscale up command · Tailscale Docs](https://tailscale.com/docs/reference/tailscale-cli/up)
- [tailscale set man page](https://linuxcommandlibrary.com/man/tailscale-set)
