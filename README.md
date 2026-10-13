# Keyherd releases

Signed Linux builds of the Keyherd agent for `amd64` and `arm64`. Each release has:

- `keyherd_<version>_linux_<arch>.tar.gz`: the `keyherd` binary and its five systemd units;
- `install.sh`;
- `checksums.txt`: the sha256 of the two archives and `install.sh`;
- `checksums.txt.sigstore.json`: the cosign bundle that signs `checksums.txt`.

## Install

You need `curl` from apt (not the snap: a snap has its own `/tmp`), `sha256sum`, `tar`, and [cosign](https://github.com/sigstore/cosign) v3 or later. cosign v2 cannot read the bundle.

1. Download `install.sh`, `checksums.txt` and its bundle for a tag into a new private directory:

   ```sh
   tag=v0.1.0
   base=https://github.com/meanii/keyherd-releases/releases/download/$tag
   cd "$(mktemp -d)"
   curl -fsSL --proto =https -O "$base/install.sh" -O "$base/checksums.txt" -O "$base/checksums.txt.sigstore.json"
   ```

2. Check the signature, then the checksum of `install.sh`. Run both as yourself, not with `sudo`: `sudo`'s `secure_path` hides a `cosign` in your own `PATH`.

   ```sh
   cosign verify-blob --bundle checksums.txt.sigstore.json \
     --certificate-identity "https://github.com/meanii/keyherd/.github/workflows/release.yml@refs/tags/$tag" \
     --certificate-oidc-issuer https://token.actions.githubusercontent.com \
     checksums.txt
   sha256sum -c --ignore-missing checksums.txt
   ```

   The first prints `Verified OK`, the second `install.sh: OK`. Stop if either does not.

3. Install from that directory. `install.sh` downloads the archive for this machine and checks it against the `checksums.txt` you just verified:

   ```sh
   sudo KEYHERD_VERSION=$tag sh install.sh
   ```

`install.sh` creates the `keyherd` user and installs the binary and the units. It never overwrites an env file and never enables or starts a unit. For the next steps, follow Agent setup below. Running your own Keyherd server is not supported before v0.4.

**Upgrade.** Repeat the three steps with the new tag. Running agents keep the old binary until they restart:

```sh
sudo systemctl try-restart keyherd.service 'keyherd-agent@*.service'
```

## Agent setup

One agent instance per certificate, `keyherd-agent@<name>`, on the host that serves it.

1. Ask the Keyherd operator for the agent's token and the server URL.
2. Fill the `KEYHERD_TELEGRAM_*` lines in `/etc/keyherd/server.env`; the agent unit reads them from there.
3. Copy `/etc/keyherd/agent/example.env.sample` to `/etc/keyherd/agent/<name>.env` and fill it as its comments say, taking the paths, `FILE_GROUP`, `TEST_CMD` and `RELOAD_CMD` from your server's row below.
4. Create the certificate directory with `sudo install -d -m 0750 -o root -g <FILE_GROUP> <dir>`, and point your server's configuration at the two file paths without reloading it. For nginx or HAProxy, add the row's drop-in with `sudo systemctl edit keyherd-agent@<name>`.
5. `sudo systemctl enable --now keyherd-agent@<name>`. Its first pull writes the files, runs `TEST_CMD` and `RELOAD_CMD`, then checks that the server serves the new certificate. Check `journalctl -u keyherd-agent@<name>` for it. Until a pull succeeds, your server's configuration names files that do not exist, so revert it if the pull failed: otherwise the server's next restart fails.

The agent refuses a key that does not match its certificate before it writes anything. The check after the reload catches a server that did not take the new certificate.

## Presets

| Server | Files | `FILE_GROUP` | `TEST_CMD` | `RELOAD_CMD` |
|---|---|---|---|---|
| Caddy | `tls <chain> <key>` | `caddy` | `caddy validate --config /etc/caddy/Caddyfile` | `caddy reload --config /etc/caddy/Caddyfile --force` |
| nginx | `ssl_certificate <chain>;` `ssl_certificate_key <key>;` | `root` | `nginx -t` | `nginx -s reload` |
| HAProxy | `crt <dir>/<name>.pem`, key at `<dir>/<name>.pem.key` | `haproxy` | `haproxy -c -f /etc/haproxy/haproxy.cfg` | `systemctl reload haproxy` |

**Caddy.** The unit already writes under `/etc/caddy/certs`, so keep the files in a directory there, such as `/etc/caddy/certs/<name>/`. No drop-in.

**nginx.** With the files in `/etc/nginx/certs/<name>/`:

```ini
[Unit]
After=nginx.service

[Service]
ReadWritePaths=/etc/nginx/certs/<name> /run /var/log/nginx
CapabilityBoundingSet=CAP_DAC_OVERRIDE
```

`nginx -t` opens `/run/nginx.pid` and the log files for writing, and Ubuntu's log files belong to `www-data`, which the agent's root cannot open without `CAP_DAC_OVERRIDE`. If your configuration names a log or pid file elsewhere, add its directory to `ReadWritePaths`. The agent runs as root and can ask systemd to run commands outside its sandbox, so the sandbox guards against the agent's mistakes, not against someone who controls it. This drop-in also lets it read any file it can see, whoever owns it (all but `/home`, `/root`, `/run/user` and its private `/tmp` and `/var/tmp`), and write any file under `/run` and `/var/log/nginx`. Do not use it on a host that also runs a Keyherd server.

**HAProxy.** With the files in `/etc/haproxy/certs/<name>/`:

```ini
[Unit]
After=haproxy.service

[Service]
ReadWritePaths=/etc/haproxy/certs/<name>
```

Name the certificate file in `crt`, never the directory: the agent keeps `.prev` and `.new` files beside it, and HAProxy would load those too.

The nginx and HAProxy rows were proven with Ubuntu 24.04's packages, under the agent's own unit.
