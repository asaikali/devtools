# dev-tools

A mise-managed installer for developer tools on macOS and Linux. Once installed,
tools and background services run independently of this repository.

## Zot container image cache

Zot runs as your user at **http://127.0.0.1:5000**, starts at login, and restarts
after unexpected failure. Docker Hub images are cached on demand. Search and the
web interface are enabled.

```text
~/.zot/
├── config.json   # installed copy of zot/config.json
├── data/         # cached images and registry metadata
└── zot.log
```

The service directly runs `~/.local/share/mise/installs/zot/latest/zot`, with
`~/.zot` as its working directory. It uses no repository paths or mise tasks at
runtime. You can move or remove the checkout after setup.

## Install

Requires [mise](https://mise.jdx.dev/) 2026.10.8 or newer, a macOS desktop login or
Linux systemd user session, and curl. The setup uses mise's standard user tool
directory. Zot needs no root privileges or Homebrew.

```sh
mise trust
mise install
mise run setup
```

`mise.toml` pins Zot **2.1.22** and selects the official full GitHub release binary
for macOS or Linux, on ARM64 or x86-64. Setup validates and copies the config into
`~/.zot`, registers the native user service, and restarts it. Reapplying preserves
cached data and replaces the installed config with the version in Git.

Mise generates the macOS LaunchAgent or Linux systemd user unit. No custom service
script or generated repository-local config is needed. See
[mise user services](https://mise.jdx.dev/bootstrap/services.html).

## Update

Change `tools.zot.version` in `mise.toml`, then:

```sh
mise install
mise run zot:restart
```

`latest` means the newest **installed** Zot version, rather than downloading new
releases automatically. Installing a newer version updates the link; restart
makes the service use it. To deploy changes to `zot/config.json`, run
`mise run setup` again.

## Operate

| Command | Behavior |
| --- | --- |
| `mise run zot:start` | Register/start the service. |
| `mise run zot:stop` | Stop it for this session; login startup stays enabled. |
| `mise run zot:restart` | Restart using the latest installed binary and installed config. |
| `mise run zot:status` | Inspect whether the service is running. |
| `mise run zot:ping` | Wait briefly for the OCI endpoint. |
| `mise run zot:ui` | Open the web interface. |
| `mise run zot:configure` | Validate and copy the source config into `~/.zot`. |
| `mise run zot:validate` | Validate the source config. |
| `mise run zot:plan` | Preview native service registration. |

You can also restart without this repository:

```sh
# macOS
launchctl kickstart -k "gui/$(id -u)/dev.mise.zot"

# Linux
systemctl --user restart dev.mise.zot.service
```

Logs are in `~/.zot/zot.log`. No task deletes cached data. Linux startup before
login is outside the initial scope.

## Use the cache

On native Linux, with Docker on the same host:

```sh
docker pull 127.0.0.1:5000/library/busybox:1.37.0
```

Docker Desktop runs its daemon inside a VM. On this Mac, container access to Zot
through `host.docker.internal:5000` was verified; daemon pulls through localhost
failed. To route ordinary Docker Hub pulls through Zot, merge this setting into
Docker Desktop's **Settings → Docker Engine**, then restart Docker:

```json
{
  "registry-mirrors": ["http://host.docker.internal:5000"]
}
```

For native Linux, use `http://127.0.0.1:5000` as the mirror address. Preserve other
engine settings. Docker restart can interrupt running containers; setup does not
change or restart Docker. Docker's mirror setting covers Docker Hub. Other upstream
registries need explicit Zot routing and corresponding client configuration.

Cached tags are checked upstream at most once every five minutes. Docker media
types retain their digests through `http.compat = ["docker2s2"]`; Zot 2.1.22 ignores
the older `preserveDigest` flag, so it is omitted. See
[Zot mirroring](https://zotregistry.dev/v2.1.21/articles/mirroring/).

## Validation

Validated Zot configuration and all mise tasks on this Mac. Existing cached images
and metadata were moved into `~/.zot/data`. The service was restarted from outside
the checkout while both repository config files were temporarily unavailable; the
cached image remained accessible and unchanged.

The Linux unit was rendered in an isolated Linux container. Actual Linux execution
and login startup need testing on a systemd host. Reboot/login behavior on this Mac
has not been exercised. Automatic Docker Desktop mirror routing is still pending
its engine setting and restart.
