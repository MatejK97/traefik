# traefik

The single reverse proxy for my homelab: every HTTP/S request to every service,
in Docker or Kubernetes, goes through this Traefik instance on the VPS.

> A published snapshot of a repository that runs on my self-hosted GitLab.
> More about the lab: [homelab](https://github.com/MatejK97/homelab)

## Design

- **Two pairs of entrypoints, each bound to an exact address.**
  `web`/`websecure` listen on the public IP; `internal-web`/`internal`
  listen only on the WireGuard address. Every admin UI (GitLab, ArgoCD,
  Grafana, Homarr, the n8n editor, this dashboard) is routed to `internal`,
  so a routing mistake can't make it public: the same hostname requested from
  the internet gets a 404.
- **Only n8n's webhook paths are public,** on a separate hostname and rate
  limited (`config/dynamic/n8n.yaml`).
- **Host networking.** Docker's published ports bypass `ufw`; with host
  networking Traefik binds the addresses itself, so the firewall applies, and
  it can reach k3s NodePorts that listen on `127.0.0.1` only.
- **TLS for everything, including VPN-only names,** via Let's Encrypt with the
  Cloudflare DNS-01 challenge. The Cloudflare token lives in `.env` on the
  server, not in Git.
- **Header hardening:** HTTP redirects to HTTPS at the entrypoint, and
  `aliasHeadersStrategy: delete` drops look-alike headers (`X_Forwarded_For`)
  that a backend could mistake for ones Traefik sets.
- **Discovery:** Docker services by container labels on the shared
  `frontend` network; Kubernetes services through the file provider
  (`config/dynamic/`), hot-reloaded without a restart.
- **Metrics** on a separate entrypoint on the WireGuard address, scraped by
  Prometheus, with per-router labels.

## Layout

```
docker-compose.yml        host networking, config mounted read-only
config/traefik.yaml       entrypoints, ACME resolver, providers, metrics
config/dynamic/*.yaml     routes to k3s NodePorts (one file per service)
```

`203.0.113.10` in `traefik.yaml` is a documentation address standing in for
the VPS's real public IP.

Deployed by the shared pipeline in [ci-templates](https://github.com/MatejK97/ci-templates).
