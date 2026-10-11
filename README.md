docker volume create recflare-data
docker run --rm -p 10000:10000 \
-e JWT_SECRET='YOUR_RANDOM_KEY_WITH_AT_LEAST_32_CHARACTERS' \
-e PUBLIC_BASE_URL=http://localhost:10000 \
-v recflare-data:/var/data recflare-node
```

## Deployment on Render via Dockerfile

No deployment was performed on the user's Render account. The Dockerfile is ready; tests were run locally using Node 24.8.0, not as a Docker build in this environment.

1. Extract this ZIP and publish **the contents of the `server-node` folder at the root of a repository**. Do not upload the data folder or secrets.
2. Create a **Web Service**, connect the repository, and choose **Docker**.
3. Dockerfile Path: `./Dockerfile`; Docker Context: `.`. The container executes `node src/docker-entrypoint.js`, which adjusts disk directory permissions, drops root privileges, and starts `src/server.js` as UID/GID 1000. Files brought from another server must be readable/writable by this user; the entrypoint does not recursively change the entire disk.
4. Choose a **paid** plan, as this configuration requires a Persistent Disk. The Blueprint uses the identifier `0.5c-512mb`.
5. Add a persistent disk mounted at **`/var/data`** (initially 1 GB) and keep **one instance**.
6. Configure:

| Variable | Value |
|---|---|
| `JWT_SECRET` | stable random key, minimum 32 characters |
| `PUBLIC_BASE_URL` | `https://YOUR-SERVICE.onrender.com`, no trailing slash |
| `DATA_DIR` | `/var/data` |
| `ROUTING_MODE` | `path` |
| `NODE_ENV` | `production` |
| `PORT` | `10000`, or the port injected by Render |
| `TRUST_PROXY` | `1` only when behind Render's managed proxy |

7. Set the Health Check Path to `/healthz` and deploy.
8. Test `/healthz`, `/accounts/`, `/api/api/config/v2`, and `/www/` on the service domain.

Alternative: use **Blueprint** with `render.yaml`. The JWT key is generated during the initial provisioning; fill in `PUBLIC_BASE_URL` once you know the actual URL. If it changes, update the variable and redeploy. If the files are located in a repository subdirectory, adjust the Root Directory and Docker context accordingly.

### Render persistence and limits

Only data written to the persistent mount survives the container lifecycle. The project writes to:

```text
/var/data/recflare.sqlite
/var/data/recflare.sqlite-wal
/var/data/recflare.sqlite-shm
/var/data/notifications.sqlite
/var/data/cdn/
/var/data/images/
```

Do not use the ephemeral filesystem or the free plan for the required persistence. A disk is associated with a specific instance; this design does not allow for horizontal scaling. Deployments involving a disk may interrupt connections; the client must reconnect, including WebSockets. Perform backups on storage other than the attached disk. Upload endpoints have a default global limit of 16 MiB, configurable via `MAX_BODY_BYTES`. Size your disk and memory according to usage; do not assume 512 MB is sufficient for a production load. The guidelines were verified on October 6, 2026, against the official documents:
- https://render.com/docs/disks
- https://render.com/docs/docker
- https://render.com/docs/blueprint-spec

## Routing and game client

On a single Render domain, the first segment selects the service and is stripped before the handler executes:

```text
/accounts/account/me -> accounts service, path /account/me
/auth/connect/token  -> auth service, path /connect/token
/api/api/config/v2   -> api service, path /api/config/v2
/notify/hub/v1       -> notify service, path /hub/v1
```

The `/` discovery endpoint uses `PUBLIC_BASE_URL` and advertises the prefixed URLs. The original name-server also advertises services that lack a specific implementation in the ZIP (e.g., auxiliary media services); this port does not create implementations for those hosts. Configure `SUBDOMAINS` redirects when your client requires them, in accordance with the supported protocol.

The client must accept these prefixed URLs and be modified to point to your name-server. If it requires subdomains, configure DNS, TLS, and Render custom domains for each service, and use `ROUTING_MODE=subdomain` and `DOMAIN=your-domain`. The static site in this package has been adapted for **path** mode; do not assume its multi-subdomain navigation is validated. The official version has not been tested by connecting to the actual game.

## Authentication and integrations

- JWT HS256 uses HMAC and constant-time signature comparison; expiration and `nbf` claims are verified.
- Passwords, verifiable platforms, sign-up limits, refresh tokens, and permissions retain the original logic.
- Notifications use the original hub with SQLite storage and native WebSocket transport. SignalR JSON handshake and player subscription were tested; recovery under load and unstable network conditions were not validated.
- Expired presence entries are cleared every five minutes while the process is running.
- Photon, Meta, Discord, Turnstile, and Tachyon are not replaced by real services in this package; please configure the credentials.
