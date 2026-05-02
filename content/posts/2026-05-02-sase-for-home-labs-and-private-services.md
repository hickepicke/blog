---
title: "SASE for Home Labs and Private Services: Zero Trust Without the Enterprise Price Tag"
description: "How to apply SASE principles — Zero Trust, secure tunnels, identity-aware access — to your home lab or personal cloud services using Cloudflare and Tailscale free tiers."
date: 2026-05-02
draft: false
tags: ["sase", "zero-trust", "cloudflare", "tailscale", "homelab", "security", "self-hosting"]
categories: ["guides"]
---

SASE — Secure Access Service Edge — is a word invented by a Gartner analyst to sell enterprise contracts. It worked. But the underlying ideas are genuinely useful, and the tools to implement them are now free.

If you run a home lab, self-host services, or have workloads scattered across Cloudflare Workers, Railway, or Fly.io, this is for you.

## What SASE Actually Means

SASE combines two things that traditionally lived in separate products:

- **Network security** — firewalls, traffic inspection, DNS filtering
- **Network access** — VPNs, SD-WAN, getting users to resources

The shift: instead of trusting everything inside a network perimeter, SASE assumes the perimeter doesn't exist. Access decisions happen per-request, based on identity and context. This is Zero Trust.

For enterprise, that's a $200/user/year problem. For a home lab, it's a free tier problem.

## The Problem with Home Labs

Running Proxmox, a NAS, Vaultwarden, or a Kubernetes cluster at home has one uncomfortable default: either it's unreachable from outside (fine until you need it remotely), or you've opened ports on your router. Not fine.

Open ports mean brute force exposure, one misconfigured service away from a full network compromise, and ISP dynamic IP headaches.

Containerized or serverless workloads on PaaS platforms have the opposite problem: they're reachable by design, but you need to control who reaches them without bolting on a separate auth layer.

SASE solves both. Total cost: zero.

## The Stack

### Cloudflare Zero Trust

The free tier includes everything you actually need:

- **Cloudflare Tunnel** (`cloudflared`) — runs on your home server, establishes an outbound-only encrypted connection to Cloudflare's edge. No open ports, no static IP required.
- **Cloudflare Access** — sits in front of your tunneled service and enforces identity checks (email OTP, GitHub, Google) before a request reaches your server.
- **Cloudflare Gateway** — DNS filtering for devices on your network.

Up to 50 users, unlimited applications. For a home lab, effectively unlimited.

### Tailscale

A mesh VPN built on WireGuard. Every device gets a stable private IP on a virtual network (`100.x.x.x`). Devices connect directly using NAT traversal — traffic doesn't route through a central server unless it has to.

Free tier: 100 devices, 3 users, unlimited bandwidth.

### When to Use Which

| Scenario | Tool |
|---|---|
| Expose a home service to a colleague | Cloudflare Tunnel + Access |
| Access your home server from your laptop | Tailscale |
| Protect a public Cloudflare Worker with auth | Cloudflare Access |
| Connect a Fly.io container to your home network | Tailscale subnet router |
| Replace your router VPN entirely | Tailscale |

## Setting Up Cloudflare Tunnel

Install `cloudflared` on whatever runs your service — Linux, Docker, Raspberry Pi:

```bash
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o cloudflared
chmod +x cloudflared

cloudflared tunnel login
cloudflared tunnel create my-homelab
```

Create `~/.cloudflared/config.yml`:

```yaml
tunnel: <tunnel-id>
credentials-file: /root/.cloudflared/<tunnel-id>.json

ingress:
  - hostname: grafana.hicke.se
    service: http://localhost:3000
  - hostname: vault.hicke.se
    service: http://localhost:8080
  - service: http_status:404
```

Run it:

```bash
cloudflared tunnel run my-homelab
```

Add a CNAME in Cloudflare DNS pointing `grafana.hicke.se` to `<tunnel-id>.cfargotunnel.com`. Then add an Access policy requiring email authentication. No firewall rules touched, no ports opened.

For Docker, `cloudflared` runs as a sidecar and requires no changes to existing service configs.

## Setting Up Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

That's it. The device shows up in your Tailscale admin with a stable `100.x.x.x` address.

To reach your entire home network, not just one machine:

```bash
tailscale up --advertise-routes=192.168.1.0/24
```

Approve the route in the admin panel. Every device on your Tailnet can now reach `192.168.1.x` as if it were local.

For Fly.io or Railway containers, Tailscale runs as a process inside the container and joins your Tailnet on startup. Your container gets a private IP and talks directly to your home server. No public exposure.

## What Zero Trust Looks Like in Practice

Both tools enforce Zero Trust without calling it that:

**No implicit trust.** A request from your home network gets the same identity check as one from a coffee shop. Cloudflare Access enforces this explicitly. Tailscale uses WireGuard's mutual authentication — every device has a cryptographic identity, no exceptions.

**Least privilege.** Per-application policies in Cloudflare Access. Grafana requires your email. The admin panel requires your email plus an additional rule. Services that shouldn't be public aren't reachable at all.

**No open inbound ports.** Both tools use outbound connections to establish connectivity. Your router firewall blocks all inbound traffic. Always.

## What You Don't Get for Free

Worth knowing before you depend on this:

- Cloudflare Gateway's HTTP inspection and data loss prevention require a paid plan
- Tailscale's free tier is limited to 6 users — fine for personal use, awkward for a shared lab
- Neither replaces a SIEM for compliance

For a personal home lab, none of these matter.

## The Setup That Works

What I'd run:

1. **Tailscale** on all personal devices and servers — private mesh, zero configuration after setup
2. **Cloudflare Tunnel + Access** for anything shared externally or needing a clean URL — identity enforcement at the edge
3. **No open ports** on the home router — ever

No static IP needed. Survives ISP changes. Costs nothing. And unlike a traditional VPN, access decisions happen before traffic enters your network — not after.
