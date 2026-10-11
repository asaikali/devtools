# dev-tools

A mise-managed installer for developer tools on macOS and Linux. Once installed,
tools and background services run independently of this repository.

## Zot container image cache

Zot runs as your user at **http://127.0.0.1:5051**, starts at login, and restarts
after unexpected failure. Public container images from Docker Hub, GHCR, Quay,
Kubernetes, ECR Public, Microsoft, Google, and GitLab are cached on demand. Search
and the web interface are enabled.

Port 5051 avoids macOS AirPlay's use of port 5000 and keeps this service separate
from the OCI playground on port 5050. See [Apple's port reference](https://support.apple.com/en-us/103229).

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
mise trust --all
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
docker pull 127.0.0.1:5051/library/busybox:1.37.0
```

Docker Desktop runs its daemon inside a VM. On this Mac, container access to Zot
through `host.docker.internal:5051` was verified; daemon pulls through localhost
failed. With Docker Desktop installed, configure ordinary Docker Hub pulls from
a fresh checkout:

```sh
mise run docker:plan
mise run docker:configure
docker desktop restart
docker pull busybox:1.37.0
```

`docker/daemon.json` declares the mirror and permits HTTP only for the local
`host.docker.internal:5051` registry, so explicit pulls from Zot work too. See
[Docker registry transport](https://docs.docker.com/reference/cli/dockerd/#insecure-registries).
The task uses mise's
[native JSON merge](https://mise.jdx.dev/dotfiles/edits.html#set-keys-in-a-config-file)
to update `~/.docker/daemon.json`, preserving other engine settings. Before a
change, an existing file is backed up beside it as `daemon.json.before-zot.*`.
Repeated application leaves a matching file unchanged. Invalid JSON is refused.
The repository owns the `registry-mirrors` and `insecure-registries` arrays; add
any other desired entries to the source file. The installed config is a regular
file and does not depend on this checkout.

`mise.docker.toml` keeps Docker configuration opt-in, separate from Zot setup.
The configure task targets macOS Docker Desktop and never restarts it; the
restart command can interrupt running containers. For native Linux, merge
`"registry-mirrors": ["http://127.0.0.1:5051"]` into Docker's daemon configuration
(normally `/etc/docker/daemon.json`) and restart Docker separately. Docker's
mirror setting covers Docker Hub. An ordinary `docker pull ghcr.io/...` still goes
directly to GHCR; use the Zot address below to cache other registries.

### Other public registries

Each upstream has its own Zot repository prefix to separate image names:

| Upstream | Path after the local Zot address |
| --- | --- |
| Docker Hub (`docker.io`) | `library/<image>` or `<owner>/<image>` |
| GitHub (`ghcr.io`) | `ghcr.io/<owner>/<image>` |
| Quay (`quay.io`) | `quay.io/<owner>/<image>` |
| Kubernetes (`registry.k8s.io`) | `registry.k8s.io/<image>` |
| ECR Public (`public.ecr.aws`) | `public.ecr.aws/<alias>/<image>` |
| Microsoft (`mcr.microsoft.com`) | `mcr.microsoft.com/<image>` |
| Google (`gcr.io`) | `gcr.io/<project>/<image>` |
| GitLab (`registry.gitlab.com`) | `registry.gitlab.com/<group>/<project>[/<image>]` |

On Docker Desktop, for example:

```sh
docker pull host.docker.internal:5051/ghcr.io/stefanprodan/podinfo:6.9.1
docker pull host.docker.internal:5051/registry.k8s.io/pause:3.10
```

On native Linux, replace `host.docker.internal` with `127.0.0.1`. Keep the upstream
repository path and tag (or digest) after its registry prefix. Docker Hub official
images need the `library/` prefix when pulled explicitly from Zot.

All upstreams use TLS verification and on-demand fetching, with no periodic
full-registry sync. No upstream credentials are configured; private images need
additional authentication. Google's entry covers `gcr.io` URLs backed by Artifact
Registry, not every regional `*-docker.pkg.dev` host. See
[Google's gcr.io repositories](https://docs.cloud.google.com/artifact-registry/docs/transition/gcr-repositories).

To add or change upstreams, edit `zot/config.json` in this repository and run
`mise run setup`. The `zot:configure` dependency validates and installs the source;
the setup task then restarts Zot. Runtime configuration is deployed through mise.

Cached tags are checked upstream at most once every five minutes. Docker media
types retain their digests through `http.compat = ["docker2s2"]`;
[Zot 2.1.22 ignores the older `preserveDigest` flag](https://github.com/project-zot/zot/blob/v2.1.22/pkg/extensions/config/sync/config.go#L74),
so it is omitted. See
[Zot mirroring](https://zotregistry.dev/v2.1.21/articles/mirroring/).

## Validation

Validated the configuration with Zot 2.1.22 and all 12 mise tasks on this Mac.
Deployed Zot and Docker settings through mise. One public image manifest from
each of the eight upstreams was cached under its expected path, matched the
original bytes/digest, and remained unchanged on repeated requests. Complete
Docker pulls succeeded for GHCR Podinfo and Kubernetes Pause; automatic Docker
Hub routing still worked for BusyBox. All six existing containers resumed after
Docker Desktop restarted.

Docker configuration tasks were tested with temporary files for preview without
writes, preservation of unrelated settings, backups, repeated application without
rewrites, missing config creation, and refusal of malformed JSON or unrelated
symlinks. The service also passed a restart test from outside the checkout while
repository config files were unavailable; a cached image remained unchanged.

The Linux unit was rendered in an isolated container. Actual Linux execution and
login startup need testing on a systemd host. Reboot/login behavior on this Mac
has not been exercised. Private upstream authentication has not been configured
or tested.
