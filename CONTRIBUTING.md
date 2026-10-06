# Contributing

The branch flow, the hooks, the releases and the house style are in
`home-server/CONTRIBUTING.md`.

## Rules a service follows

- Each rootless user has its own container network. Talk to other services
  through the host, on the published port, never by container name. Use a
  literal address, not a name. `home-server-bunker/README.md`, "How the proxy
  reaches the other pods", has the reasoning and the traps.
- To reach a port that another pod published on the host loopback, put
  `Network=pasta:-T,<port>` on this pod and use `127.0.0.1:<port>`; repeat
  `-T,<port>` for each port. `home-server-monitoring/README.md`, "Specifics",
  explains it. The proxy is the exception: it reaches its upstreams through
  `--map-host-loopback`, as `home-server-bunker/README.md` says.
- Inside a pod use `127.0.0.1:<port>`. A rootless pod binds IPv4 only, and
  `localhost` resolves to `::1` first.
- Order the Quadlet sections `[Unit] [Container] [Service] [Install]`.
  `quadlet_service` in `home-server` adds the drop-ins that `README.md`, "Role
  contract", lists. `quadlets/container.d/` of the service adds its own or
  replaces one of them by name, except `pod.conf`. A container carries only what
  is its own. systemd applies a drop-in after the unit file, so you cannot
  override a key that a drop-in sets. Pick another key instead.
- A bind mount of a file this repository owns carries `z` (shared with the
  other containers of the pod) or `Z` (this container alone). Podman labels it
  for SELinux on each start, and nothing else does. A host path such as
  `/var/log/journal` or `/dev/log` carries neither: relabelling it breaks the
  service that owns it.
- Pin each image to a tag. `AutoUpdate=registry` follows the tag, and Renovate
  moves it.
- Every container gets a ceiling in `<name>_service_memory_defaults`. If the
  entrypoint runs as root and switches user or fixes ownership, add
  `AddCapability=SETUID SETGID` (`CHOWN`, ...) to that container. Try without
  the capability first: an image with `USER` set needs nothing.
- Add `HealthCmd` + `HealthOnFailure=kill` only after you ran the check against
  the image. A wrong check plus `kill` restarts a healthy container forever.
- Logs go to stdout. `quadlet_service` sets `LogDriver=passthrough`. A program
  that opens `/dev/stdout` by path fails under passthrough. Make that program
  log through syslog to a mounted `/dev/log`, or keep journald for that pod
  with a `quadlets/container.d/log.conf` that sets `LogDriver=journald`, as
  home-server-bunker does.

## Checks in this repository

- Hooks: the basics, ansible-lint and commitizen.
- Ansible variables are `<role>_*`.
- Renovate updates the container image tags in the role defaults, through the
  preset that `.github/renovate.json` extends.
- `home-server` checks out a service made from this template in
  `services/<name>`. `home-server/CONTRIBUTING.md` says how a change reaches the
  pin.
