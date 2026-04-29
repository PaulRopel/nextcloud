# Setup

This project uses Docker Compose and reads its runtime variables from [`.env`](./.env).

## Data root

The Compose file uses `DATA_ROOT` to place persistent volumes:

- `${DATA_ROOT}/db` for MariaDB
- `${DATA_ROOT}/html` for Nextcloud

Current default in this repo:

```env
DATA_ROOT=/mnt/data/nextcloud
```

Notes:

- `DATA_ROOT` is not inferred from the location of `docker-compose.yml`.
- If `DATA_ROOT` is not set, Compose expands the paths literally and you will not get an automatic "same folder as the compose file" fallback.
- Keep the persistent data outside the git checkout. That keeps the repository clean and avoids mixing mutable data with deployment files.

## Redis warning

On startup, Redis may print this warning:

```text
WARNING Memory overcommit must be enabled!
```

This is a host kernel setting, not a Compose setting.

Fix it on the Docker host:

```bash
sudo sysctl vm.overcommit_memory=1
```

To make it persistent across reboots, add this line to a sysctl config file on the host:

```conf
vm.overcommit_memory = 1
```

Then reload sysctl settings:

```bash
sudo sysctl --system
```

If you are using Docker Desktop, apply the setting inside the Linux VM that backs Docker, not on macOS itself.

## Nextcloud behind Caddy

This deployment uses Caddy on the VPS as the public reverse proxy and SSH reverse forwarding from the home server to the VPS.

Required pieces:

1. Caddy should proxy the public hostname to the forwarded local port on the VPS.
2. Nextcloud should trust the public domain.
3. Nextcloud should know it is behind a reverse proxy so it generates the correct URLs.

Example Caddyfile on the VPS:

```caddy
nextcloud.onspry.com {
    reverse_proxy 127.0.0.1:8040
}
```

Notes:

- Use `127.0.0.1:8040` on the VPS if the SSH reverse tunnel binds the port to loopback.
- Do not point Caddy at `192.168.1.241:8040` from the VPS; that is the home LAN address and will not be the public path.
- If Caddy is running in Docker instead of as a host service, `localhost` inside the container is not the VPS host. In that case, the proxy target must be adjusted accordingly.

Nextcloud env vars for the proxy setup:

```env
NEXTCLOUD_TRUSTED_DOMAINS=nextcloud.onspry.com
OVERWRITEHOST=nextcloud.onspry.com
OVERWRITEPROTOCOL=https
TRUSTED_PROXIES=127.0.0.1
```

Notes:

- `NEXTCLOUD_TRUSTED_DOMAINS` fixes the "Access through untrusted domain" error.
- `OVERWRITEHOST` and `OVERWRITEPROTOCOL` help Nextcloud generate correct external URLs behind Caddy.
- `TRUSTED_PROXIES` should include the proxy address that Nextcloud sees. If Caddy is on the same VPS and talks to the app through the SSH tunnel, `127.0.0.1` is usually the right first value.

Important:

- These environment variables need to be passed into the `app` service in `docker-compose.yml`.
- On an existing Nextcloud install, env vars may not retroactively update `config.php`. If the error persists after restart, set the values with `occ` inside the container.

Example:

```bash
docker compose exec app php occ config:system:set trusted_domains 1 --value=nextcloud.onspry.com
```
