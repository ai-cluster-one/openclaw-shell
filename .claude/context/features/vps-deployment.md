# VPS Deployment (Direct-to-Server)

## What It Does

Provides a one-command installation of OpenClaw Shell on a bare Linux VPS (Hetzner, DigitalOcean, etc.) with automatic HTTPS via Caddy. The container runs inside Docker on the host; Caddy runs natively on the host for TLS termination and domain routing.

## Why

Railway and Coolify handle TLS/domain management automatically, but deploying to a bare VPS has no built-in mechanism for HTTPS or domain assignment. This feature bridges that gap: `install-vps.sh` installs Docker + Caddy on the host, configures automatic Let's Encrypt certificates, and starts the OpenClaw Shell container — all in one step.

## Architecture

```
Internet (HTTPS :443)
       ↓
   Caddy (host-level, auto TLS via Let's Encrypt)
       ↓
   localhost:8080
       ↓
   Docker container (openclaw-shell)
   └── auth-proxy → openclaw gateway / pages-api / filebrowser
       ↓
   /data (host directory mounted as volume)
```

- **Caddy** runs on the host (not inside Docker). It reverse-proxies HTTPS traffic to the container's auth-proxy on `localhost:8080`. TLS certificates are provisioned and renewed automatically.
- **Docker** runs the same OpenClaw Shell image used on Railway/Coolify. No Dockerfile changes needed.
- **Data** persists in a host directory (default `/data/openclaw`), mounted into the container as `/data`.

## Why Caddy on the Host (Not in Docker)

1. Caddy's cert storage survives container rebuilds without extra volume config
2. The Dockerfile stays unchanged for Railway/Coolify where Caddy isn't needed
3. Simpler networking — no Docker network bridging for TLS termination
4. Caddy can front multiple OpenClaw containers on different domains from the same host

## Multi-Instance Support

Multiple OpenClaw instances can run on the same host. Each instance needs:
- Its own `docker-compose.vps.yml` with a unique container name and host port
- A Caddyfile entry routing its domain to its port

```
instance1.example.com { reverse_proxy localhost:8080 }
instance2.example.com { reverse_proxy localhost:8081 }
```

All instances share the same Caddy installation and TLS management.

## Remote Management

The host is fully accessible via SSH. Management options:
- `openclaw` CLI command (installed by the script) for common operations
- `ssh root@host 'docker exec openclaw-shell <command>'` for container-level access
- `ssh root@host 'openclaw update'` to pull latest code and rebuild
- Claude Code can manage the box remotely via SSH

## Implementation

- **Install script**: `install-vps.sh` (project root)
- **Generated compose file**: `docker-compose.vps.yml` (written by install script, not committed)
- **Caddyfile**: `/etc/caddy/Caddyfile` (written by install script on the host)
- **Management CLI**: `/usr/local/bin/openclaw` (written by install script on the host)
- **Deployment guide**: `.claude/context/deployment/vps.md`
