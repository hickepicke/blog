---
title: "SASE for Home Labs and Private Services: Zero Trust Without the Enterprise Price Tag"
description: "How to apply SASE principles — Zero Trust, secure tunnels, identity-aware access — to your home lab or personal cloud services using Cloudflare and Tailscale free tiers."
date: 2026-05-02
draft: true
tags: ["sase", "zero-trust", "cloudflare", "tailscale", "homelab", "security", "self-hosting"]
categories: ["guides"]
---

SASE — Secure Access Service Edge — sounds like a word invented by a Gartner analyst to sell enterprise contracts. And it was. But the underlying ideas are genuinely useful, and the tools to implement them are now available for free.

If you run a home lab, self-host services, or have workloads scattered across Cloudflare Workers, Railway, Fly.io, or similar platforms, this post is for you.

## What SASE Actually Means

SASE combines two things that traditionally lived in separate boxes:

- **Network security** — firewalls, secure web gateways, traffic inspection
- **Network access** — VPNs, SD-WAN, connecting users to resources

The key shift: instead of trusting everything inside a network perimeter and blocking everything outside it, SASE assumes the perimeter doesn't exist. Access decisions are made per-request, based on identity, device state, and context. This is Zero Trust.

For enterprise, this is a $200/user/year problem. For home lab users with a few services, it's a free tier problem.

## The Threat Model for Home Labs

Running a Proxmox node, a NAS, a self-hosted Vaultwarden, or a Kubernetes cluster at home has one uncomfortable default: either it's unreachable from the outside (fine until you need it remotely), or you've opened ports on your router (not fine).

Open ports mean:
- Exposure to brute force and scanners
- One misconfigured service away from a full network compromise
- Your ISP's dynamic IP causing DNS headaches

Containerized or serverless workloads on PaaS platforms (Cloudflare Workers, Fly.io, Railway) have a different problem: they're reachable by design, but you need to control *who* can reach them without bolting on a separate auth layer.

SASE principles solve both.

## The Free Tier Stack

### Cloudflare Zero Trust

Cloudflare's free tier includes:

- **Cloudflare Tunnel** (`cloudflared`) — runs on your home server or container, establishes an outbound-only encrypted connection to Cloudflare's edge. No open ports, no static IP needed.
- **Cloudflare Access** — sits in front of your tunneled service and enforces identity checks (email OTP, GitHub, Google, etc.) before a request reaches your server.
- **Cloudflare Gateway** — DNS filtering and traffic inspection for devices on your network.

Free tier limits: up to 50 users, unlimited applications. For a home lab, this is effectively unlimited.

**What it's good for:** exposing home services (Vaultwarden, Grafana, Home Assistant, internal dashboards) securely to yourself or a small group over the internet, without opening firewall ports.

### Tailscale

Tailscale is a mesh VPN built on WireGuard. Every device gets a stable private IP on a virtual network (100.x.x.x range). Devices connect directly to each other using NAT traversal — no traffic routes through a central server unless it has to.

Free tier: 100 devices, 3 users, unlimited bandwidth.

**What it's good for:** private connectivity between your own devices — laptop to home server, phone to NAS, cloud VM to home network. No public exposure at all.

### Combining Them

These two tools cover different access patterns:

| Scenario | Tool |
|---|---|
| Expose home service to a colleague | Cloudflare Tunnel + Access |
| Access home server from your own laptop | Tailscale |
| Protect a public Cloudflare Worker with auth | Cloudflare Access |
| Connect a Fly.io container to home network | Tailscale subnet router |
| Replace your router VPN | Tailscale |

## Setting Up Cloudflare Tunnel

Install `cloudflared` on the machine running your service (works on Linux, Docker, Raspberry Pi, anywhere):

```bash
# Install cloudflared
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o cloudflared
chmod +x cloudflared

# Authenticate and create a tunnel
cloudflared tunnel login
cloudflared tunnel create my-homelab
```

Create a config file at `~/.cloudflared/config.yml`:

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

Run the tunnel:

```bash
cloudflared tunnel run my-homelab
```

Add DNS records via the Cloudflare dashboard — CNAME `grafana.hicke.se` to `<tunnel-id>.cfargotunnel.com`. Then add an Access policy requiring email authentication before anyone reaches `grafana.hicke.se`.

For Docker environments, `cloudflared` runs as a sidecar container and requires no changes to existing service configs.

## Setting Up Tailscale

```bash
# Install on any Linux machine
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

That's it. The device appears in your Tailscale admin panel with a stable 100.x.x.x address. Every other device you connect joins the same private mesh.

For accessing your entire home network (not just one machine), enable subnet routing:

```bash
# On your home server/router
tailscale up --advertise-routes=192.168.1.0/24
```

Approve the route in the Tailscale admin panel. Now any device on your Tailnet can reach `192.168.1.x` addresses as if it were on your home network.

For Fly.io or Railway containers, Tailscale runs as a process inside the container and joins your Tailnet on startup — your container gets a private IP and can talk directly to your home server without any public exposure.

## Zero Trust Principles in Practice

Both tools implement Zero Trust by default, even if they don't call it that:

**No implicit trust based on network location.** A request from inside your home network gets the same identity check as one from a coffee shop. Cloudflare Access enforces this explicitly. Tailscale uses mutual authentication via WireGuard — every device has a cryptographic identity.

**Least privilege.** Cloudflare Access lets you write policies per-application. Your Grafana dashboard requires your email. Your admin panel requires your email plus a specific rule. Services that shouldn't be public aren't reachable at all.

**No open inbound ports.** Cloudflare Tunnel and Tailscale both use outbound connections to establish connectivity. Your router firewall can block all inbound traffic.

## What You Don't Get for Free

The free tiers are genuinely useful but have limits worth knowing:

- **Cloudflare Gateway** data loss prevention and HTTP inspection require a paid plan
- **Tailscale** free tier is limited to 3 users — fine for personal use, not for a shared lab with colleagues
- **Cloudflare Access** device posture checks (is this device managed? is it patched?) require the Teams plan
- Neither tool replaces a proper SIEM or audit log system for compliance purposes

For a home lab or small personal project, none of these gaps matter. For anything approaching production or multi-user, they're worth factoring in.

## The Practical Setup for a Home Lab

The combination that covers most home lab needs:

1. **Tailscale** on all your own devices and servers — private mesh for personal access
2. **Cloudflare Tunnel + Access** for anything you want to share with others or access via a clean URL — with identity enforcement in front of it
3. **No open ports** on the home router — ever

This costs nothing, requires no static IP, survives ISP changes, and is more secure than a traditional VPN setup because access decisions happen at the edge, not after traffic has already entered your network.
