# WSL2 + Docker + Wazuh Deployment

Wazuh's single-node stack (Manager, Indexer, Dashboard) runs in Docker inside WSL2, on the physical Windows 11 host — not as a dedicated VM.

## Prerequisites

- Windows 11, WSL2 enabled
- Docker Desktop, WSL 2 based engine selected during install
- WSL Integration enabled for the target distro (Docker Desktop → Settings → Resources → WSL Integration)

## 1. Get the Wazuh Docker deployment files

```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.6
cd wazuh-docker/single-node
```

## 2. Generate SSL certificates

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

## 3. Pull images

Pulling individually (rather than letting `compose` pull all three in parallel) gives cleaner progress feedback and avoids bandwidth contention on slower connections:

```bash
docker pull wazuh/wazuh-indexer:4.14.6
docker pull wazuh/wazuh-manager:4.14.6
docker pull wazuh/wazuh-dashboard:4.14.6
```

## 4. Start the stack

```bash
cd single-node/config
docker compose up -d
```

Give the Indexer 1–2 minutes to reach cluster health `green` before relying on the Manager or Dashboard:

```bash
curl -k -u admin:<password> https://localhost:9200/_cluster/health?pretty
```

## 5. Verify

```bash
docker compose ps
```

All three containers (`wazuh.manager`, `wazuh.indexer`, `wazuh.dashboard`) should show `Up`.
Dashboard: `https://localhost:443` (mapped from container port 5601).

## Known Operational Notes

- `vm.max_map_count` may need raising for the Indexer (OpenSearch) to start cleanly on some WSL2 kernels:

  ```bash
  sudo sysctl -w vm.max_map_count=262144
  ```

- WSL2 clock drift after host sleep/idle can silently break TLS/DNS-dependent operations (image pulls, dashboard sessions). If pulls stall or the dashboard login-loops, check `date` against actual time before troubleshooting further:

  ```bash
  sudo timedatectl set-ntp true
  # or, if chrony reports "running in a container" and refuses to step the clock:
  sudo chronyd -q 'server time.windows.com iburst'
  ```

- Docker Desktop / WSL distro corruption can occur after repeated `wsl --shutdown` cycles, surfacing as `docker: command not found` or a failed `docker-desktop` distro proxy. Recovery pattern:

  ```powershell
  wsl --shutdown
  wsl --unregister docker-desktop
  ```

  Relaunch Docker Desktop, then re-enable the WSL Integration toggle (Settings → Resources → WSL Integration) — this resets on distro rebuild and must be turned back on manually.

## Startup Order After a Host Restart

1. Start Docker Desktop, wait for a steady "Engine running" state.
2. `docker compose up -d` from `wazuh-docker/single-node/config`.
3. Wait for Indexer `green` before relying on the Manager.
4. Power on lab VMs — Domain Controller first, then clients, then Kali last.
5. Confirm all agents show Active in the Dashboard before running any attack scenario.
