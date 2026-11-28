# Deploying with systemd

SoroVista ships as a single self-contained Go binary (embedded SQL migrations and HTML templates included), which makes running it directly under `systemd` on a Linux host straightforward when you do not want a container runtime on the machine.

## 1. Create a dedicated system user and install the binary

Never run SoroVista as `root`. Create a dedicated, unprivileged system account with no login shell and no home directory:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin --user-group sorovista
```

Copy the compiled `sorovista` binary into `/usr/local/bin` owned by `root:root`:

```bash
CGO_ENABLED=0 go build -trimpath -o sorovista ./cmd/sorovista
sudo install -o root -g root -m 0755 sorovista /usr/local/bin/sorovista
```

Because SoroVista listens on `:8080` by default — an unprivileged port above `1024` — the service needs **no capability grants** (`AmbientCapabilities=CAP_NET_BIND_SERVICE` is unnecessary and should be omitted). Terminate TLS on `:443` in a reverse proxy (Caddy, nginx, or Traefik) and proxy to `127.0.0.1:8080`.

## 2. Configure `/etc/sorovista/sorovista.env`

Keep configuration in `/etc/sorovista/sorovista.env` rather than inline in the unit file. In standalone mode, `DATABASE_URL` contains your Postgres password (and `RPC_URL` may contain an API key), so a world-readable environment file (`0644`) leaks credentials to every local user on the machine. Restrict `/etc/sorovista/sorovista.env` to `0640` owned by `root:sorovista`:

```bash
sudo install -d -o root -g sorovista -m 0750 /etc/sorovista
sudo install -o root -g sorovista -m 0640 /dev/null /etc/sorovista/sorovista.env
```

### Standalone mode (`SOURCE_MODE=rpc`)

Edit the two required connection settings (`DATABASE_URL` and `NETWORK` / `RPC_URL`):

```ini
# /etc/sorovista/sorovista.env (standalone mode)
SOURCE_MODE=rpc
NETWORK=testnet
DATABASE_URL=postgres://sorovista:replace-with-strong-password@127.0.0.1:5432/sorovista?sslmode=disable
HTTP_ADDR=127.0.0.1:8080
LOG_LEVEL=info
# Optional: bound storage. Unset keeps events forever.
# EVENT_RETENTION=2160h
```

### Upstream mode (`SOURCE_MODE=sorotrail`)

Upstream mode is stateless and opens no database connection — only `SOURCE_MODE` and `SOROTRAIL_URL` are required:

```ini
# /etc/sorovista/sorovista.env (upstream mode)
SOURCE_MODE=sorotrail
SOROTRAIL_URL=http://127.0.0.1:8081
HTTP_ADDR=127.0.0.1:8080
LOG_LEVEL=info
```

## 3. Install the `systemd` unit file

Write `/etc/systemd/system/sorovista.service`:

```ini
[Unit]
Description=SoroVista Soroban contract event explorer
Documentation=https://github.com/sorotrail/SoroVista
After=network-online.target
Wants=network-online.target
# Standalone mode with a local Postgres also needs the database started first.
# Add this line only when Postgres runs on the same host; a remote database
# needs nothing beyond the network dependency above. Upstream mode never
# needs Postgres at all.
# After=postgresql.service

[Service]
Type=simple
User=sorovista
Group=sorovista
EnvironmentFile=/etc/sorovista/sorovista.env
ExecStart=/usr/local/bin/sorovista

# SoroVista exits non-zero on a configuration error, so a restart loop means
# the environment file is wrong, not that a transient failure should be
# waited out. The backoff still avoids hammering a genuinely transient fault.
Restart=on-failure
RestartSec=5s

# Must exceed the 15s graceful-shutdown timeout in cmd/sorovista/main.go,
# otherwise systemd SIGKILLs in-flight requests after the HTTP server has
# already begun draining them.
TimeoutStopSec=30s

# No ambient capabilities are needed: the default listen address is :8080,
# which is above 1024, so the process can bind it as an ordinary user.
# Resist the urge to add AmbientCapabilities=CAP_NET_BIND_SERVICE.

# Logging: SoroVista writes structured JSON logs (log/slog) to stdout.
StandardOutput=journal
StandardError=journal
SyslogIdentifier=sorovista

# --- Hardening -------------------------------------------------------------
# NoNewPrivileges: a process can never gain privileges through setuid/setgid
# binaries or file capabilities, so a compromise cannot escalate inside the
# service.
NoNewPrivileges=yes
# ProtectSystem: mounts /usr, /boot and /etc read-only for this service. The
# binary and its environment file stay readable but cannot be modified.
ProtectSystem=strict
# ProtectHome: hides /home, /root and /run/user. SoroVista stores nothing
# there, so there is no reason to expose it.
ProtectHome=yes
# PrivateTmp: gives the service its own /tmp and /var/tmp namespaces, so it
# cannot read or race another process's temporary files (and vice versa).
PrivateTmp=yes
# ReadWritePaths: deliberately empty. SoroVista keeps no local state — events
# live in Postgres, logs go to stdout/journald — so ProtectSystem=strict can
# make the rest of the filesystem read-only. If you later add a local write
# path, list only that exact directory here rather than weakening the whole
# filesystem.
ReadWritePaths=

# Additional low-cost restrictions: no device nodes, no kernel module or
# sysctl changes, no control-group access, and only the socket families a
# client/server process actually needs.
PrivateDevices=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
RestrictSUIDSGID=yes
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
LockPersonality=yes

[Install]
WantedBy=multi-user.target
```

### Ordering for standalone vs upstream mode

- **Standalone mode with local Postgres:** Uncomment `# After=postgresql.service` so SoroVista starts after the local database is ready to accept connections and run startup migrations. (If Postgres runs on another host, `After=network-online.target` alone is sufficient.)
- **Upstream mode (`SOURCE_MODE=sorotrail`):** No Postgres instance is used. Leave the `postgresql.service` line commented out, or optionally order after `sorotrail.service` if SoroTrail runs on the same machine.

### Why each hardening directive is there

- **`NoNewPrivileges=yes`** — Prevents the process (or any child) from gaining privileges via `setuid`/`setgid` binaries or file capabilities.
- **`ProtectSystem=strict`** — Mounts the entire filesystem hierarchy read-only for the service, except for `/dev`, `/proc`, and `/sys`.
- **`PrivateTmp=yes`** — Gives the unit its own isolated `/tmp` and `/var/tmp` namespace that disappears when the service stops, preventing temp-file collisions or symlink attacks.
- **`ProtectHome=yes`** — Makes `/home`, `/root`, and `/run/user` inaccessible to the process.
- **`ReadWritePaths=` (empty)** — SoroVista writes nothing to local disk in either mode (standalone mode writes only to Postgres over a socket; upstream mode is completely stateless). Keeping `ReadWritePaths=` empty alongside `ProtectSystem=strict` leaves zero writable directories on the host filesystem.
- **`PrivateDevices=yes`**, **`ProtectKernelTunables=yes`**, **`ProtectControlGroups=yes`**, **`RestrictSUIDSGID=yes`** — Blocks access to physical device nodes, `/proc/sys` kernel tunables, cgroup hierarchies, and creation of setuid/setgid files.
- **`ProtectKernelModules=yes`** — Prevents the service from loading or unloading kernel modules; a compromised process cannot extend the kernel's attack surface.
- **`RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX`** — Limits socket creation to the three families SoroVista actually uses (TCP/UDP over IPv4 and IPv6, and Unix-domain sockets for Postgres). All other address families (Netlink, Bluetooth, etc.) are denied.
- **`LockPersonality=yes`** — Prevents the process from changing its execution domain (ABI), closing a Linux personality-based exploitation vector.

## 4. Enable, start, and verify

Reload `systemd`, enable the unit at boot, and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now sorovista.service
sudo systemctl status sorovista.service
```

Confirm the HTTP probes respond:

```bash
curl -s http://127.0.0.1:8080/livez
curl -s http://127.0.0.1:8080/readyz
curl -s http://127.0.0.1:8080/health
```

## 5. Reading logs and diagnosing restart loops

SoroVista writes structured JSON records (`time`, `level`, `msg`, `request_id`, etc.) to standard output via Go's `log/slog`, which `systemd-journald` captures automatically.

Tail live logs or filter recent errors with `journalctl`:

```bash
# Follow live logs
sudo journalctl -u sorovista.service -f

# Show only raw JSON log lines from the current boot
sudo journalctl -u sorovista.service -b -o cat

# Filter for errors only
journalctl -u sorovista.service --since "1 hour ago" -o cat | jq 'select(.level=="ERROR")'
```

> **Startup validation & restart loops:** At startup, `cmd/sorovista/main.go` validates all environment variables for the selected `SOURCE_MODE` and (in standalone mode) pings Postgres, runs migrations, and verifies that the Stellar RPC's network passphrase matches `NETWORK` / `NETWORK_PASSPHRASE`. If any check fails, SoroVista logs a `"startup failed"` JSON entry and exits with status `1`. With `Restart=on-failure`, a persistent restart loop every 5 seconds almost always indicates a configuration error in `/etc/sorovista/sorovista.env` (such as a missing `DATABASE_URL`, a wrong `NETWORK` preset, or an unreachable database) rather than a runtime crash.

## 6. Upgrading

```bash
CGO_ENABLED=0 go build -trimpath -o sorovista ./cmd/sorovista
sudo install -m 0755 sorovista /usr/local/bin/sorovista
sudo systemctl restart sorovista.service
```

Standalone mode applies pending migrations at startup; back up the database first. Upstream mode has nothing local to migrate.
