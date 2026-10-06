# home-server-__NAME__

A skeleton to copy for a new service: an Ansible role and a rootless Podman
Quadlet pod. `__NAME__` marks the service name.

Copy it beside this repository with the commands below. The name is the Linux
user, the pod, the `service` label, the role `<name>_service` and the variable
prefix `<name>_service_`, so it takes only `a-z`, `0-9` and `_`, starts with
a letter, and is not `core` or `root`.

```bash
name=<name>
git -C home-server-template archive --prefix="home-server-$name/" HEAD | tar -x
cd "home-server-$name"
find . -depth -name '*__NAME__*' -execdir sh -c 'mv "$1" "$(printf %s "$1" | sed "s/__NAME__/$2/g")"' _ {} "$name" \;
grep -rl __NAME__ . | xargs sed -i "s/__NAME__/$name/g"
grep -rl 'noqa: var-naming' . | xargs sed -i 's/ *# noqa: var-naming\[pattern\]//'
```

Then delete the placeholder rule in `.github/renovate.json`. Replace the
sections "Configuration interface", "Role contract" and "Monitoring" of this
README, and "Rules a service follows" of `CONTRIBUTING.md`, with a line that
points here. `home-server/README.md`, "Adding a service", has the steps
outside this repository.

| Container | Job | Default memory ceiling |
|---|---|---|
| `__NAME__-main` | | 256M |

```
home-server-__NAME__/
├── ansible-role/__NAME___service/
│   ├── defaults/main.yml        What a user sets: images, credentials, overrides
│   ├── vars/main.yml            What the role owns: config, memory and CPU defaults
│   └── tasks/main.yml           Data directories, then import quadlet_service
├── quadlets/
│   ├── __NAME__.pod.j2          Pod and published port
│   ├── __NAME__-*.container.j2  Containers
│   ├── container.d/             (optional) drop-ins for every container
│   └── configs/                 Env and config files
├── monitoring/                  Rules, dashboards and log filters
└── containers/                  (optional) build of an own image
```

## Configuration

| Variable | Default | Controls |
|---|---|---|
| `__NAME___service_password` | required | A Podman secret |
| `__NAME___service_hostname` | empty | The public hostname |
| `__NAME___service_config` | `{}` | The app's settings, merged over `__NAME___service_config_defaults` |
| `__NAME___service_memory` | `{}` | Memory ceilings per container |
| `__NAME___service_cpu` | `{}` | CPU quotas per container, such as `{main: 50%}` |
| `__NAME___service_main_image` | `docker.io/example/__NAME__:1` | Image and tag |

## Configuration interface

Every service offers the same kinds of variables, so a user configures each one
the same way. A service has only the ones that apply to it.

| Variable | In | Meaning |
|---|---|---|
| `<name>_service_<credential>` | `defaults/main.yml`, empty | A credential; `quadlet_service` stores it, or a value made from it, as a Podman secret named in `quadlet_service_secrets` |
| `<name>_service_hostname` | `defaults/main.yml`, empty | The public hostname; with an entry in `bunker_service_sites`, the proxy puts the service on it |
| `<name>_service_config` | `defaults/main.yml`, `{}` | The user's settings for the app, in the app's own keys |
| `<name>_service_config_defaults` | `vars/main.yml` | The role's generic settings; `<name>_service_config` merges over them |
| `<name>_service_memory` | `defaults/main.yml`, `{}` | Memory ceilings per container, keyed by the container name without `<name>-` |
| `<name>_service_memory_defaults` | `vars/main.yml` | The ceilings the role ships |
| `<name>_service_cpu` | `defaults/main.yml`, `{}` | CPU quotas per container, in the same keys; a container without one has no CPU limit |
| `<name>_service_cpu_defaults` | `vars/main.yml` | The quotas the role ships, often `{}` |
| `<name>_service_*_image` | `defaults/main.yml` | The images |
| `port` of the `base_setup_services` entry | the deployment directory | The pod's loopback port, `service_port` in the role |

- The defaults are generic: empty means off. A setting of one deployment, such
  as a country, belongs in its deployment directory.
- The config merges recursively: a nested dict merges, a list replaces the
  default list. The memory and CPU dicts merge key by key. Set such a dict in one
  inventory file only, because a second file replaces it.
- A service with a variable number of accounts takes one dict keyed by account
  name, such as `ntfy_service_users`, instead of one credential variable each.
- A config file carries no credential. A container reads a secret through
  `Secret=`, as a file in `/run/secrets/` or as an environment variable.
- A credential that the service reads at its first start only says so in the
  service's README, with the steps to change it.
- A setting that the role needs to work stays in the Quadlet or wins the
  merge. Such a setting is a port, a path or an access rule. Its README says
  which, or that the config can change every key.
- `true` and `false` become the app's own words, such as `true` or `yes`.

## Role contract

`site.yml` includes `<service_repo>/ansible-role/<name>_service` once for each
entry of `base_setup_services`, and passes:

| Var | Value |
|---|---|
| `service_name` | `__NAME__` |
| `service_home` | `/var/services/__NAME__` |
| `service_repo` | `<playbook dir>/services/__NAME__` |
| `service_port` | the `port` of its `base_setup_services` entry, or none |

Before that, `base_setup` creates the user with its `uid` and subuid range.
It makes the home a Btrfs subvolume with mode `0750`, with a snapshot timer.
It also enables linger and the user's `podman-auto-update.timer`. The user
and its subuids reach no private network (`home-server/README.md`,
"Security").

`/etc/containers/policy.json` admits the image repositories that the services
declare, and the signed ones that the OS image's own policy admits.
`base_setup` reads them from the `*_image` variables in `defaults/main.yml` and
from the `<name>_service_*_image` host variables. A reference names
`registry/namespace/name`, such as `docker.io/library/nginx`; a shorter one
fails the deploy. An image under `ghcr.io/marpogaus` needs this project's
cosign signature.

A role may also read `base_setup_services`, notify the `Reload systemd` handler
of `base_setup` and write metrics into `base_setup_textfile_dir`.

The role creates its data directories, then imports `quadlet_service` from
`home-server`. `quadlet_service`:

- checks `quadlet_service_required`, that `quadlet_service_memory` names
  every container and nothing else, and that `quadlet_service_cpu` names only
  containers.
- with `quadlet_service_db_dump` set, runs `pg_dumpall` into the service
  subvolume before each snapshot and prunes old dumps.
- stores `quadlet_service_secrets` as Podman secrets. It replaces a secret
  whose value changed and then restarts the pod.
- packs `quadlets/`, its own `container.d/` drop-ins and the extra files into
  one reproducible archive. It renders each `.j2` file without the suffix and
  copies the other files. It adds a `Memory=` drop-in per container and a
  `CPUQuota=` drop-in per CPU quota. Its own drop-ins put every container into
  `<name>.pod` and set the restart policy, `AutoUpdate=registry`,
  `DropCapability=ALL`, `NoNewPrivileges=true`, `PidsLimit=512` and
  `LogDriver=passthrough`. A file of the same name in the service's
  `container.d/` replaces one of them, except `pod.conf`.
- compares the archive with the one it last unpacked on the host. When they
  differ, it deletes `~/.config/containers/systemd/` of the service user,
  unpacks the archive there, reloads the user manager and restarts
  `<name>-pod.service`, from `quadlets/<name>.pod.j2`. Otherwise it only
  starts the pod if it is stopped.

Nothing else writes into the Quadlet directory: the next change deletes it.

| Var | Default | Use |
|---|---|---|
| `quadlet_service_required` | `{}` | Variable name to a regex it must match; no regex means not empty |
| `quadlet_service_secrets` | `{}` | Secret name to its value; an empty value is an optional secret that is not set |
| `quadlet_service_memory` | `{}` | Container name without `<name>-` to its ceiling |
| `quadlet_service_cpu` | `{}` | Container name without `<name>-` to its CPU quota, such as `50%` |
| `quadlet_service_db_dump` | `{}` | A Postgres container to dump before each snapshot: `container`, `user` and `retention_days` |
| `quadlet_service_restart` | `false` | `true` restarts the pod for a reason of the role |
| `quadlet_service_extra_files` | `[]` | More files, each a `dest` and its `content` |

`vars/main.yml` of this skeleton sets the first four.

## Monitoring

All files are optional. `home-server-monitoring/README.md`, "Monitoring files of
a repository", says how they reach the host.

| File | Holds |
|---|---|
| `monitoring/prometheus-rules.yaml` | Prometheus rule groups |
| `monitoring/loki-rules.yaml` | Loki ruler groups |
| `monitoring/dashboards/*.json` | Grafana dashboards |
| `monitoring/alloy-drop.txt` | One RE2 regex per line; Alloy drops a line of this service that matches |
| `monitoring/alloy-redact.txt` | One RE2 regex per line; its capture groups become `<redacted>` in every line |

- The files are not templates. In the Alloy files, `#` lines and blank lines
  do not count.
- A service's own alert name starts with its service name. The generic alerts
  of `home-server` and `home-server-monitoring` do not. `labels.severity` is
  `critical`, `warning` or `info`. `annotations.summary` is one line.
- A Loki rule selects only on the labels in `home-server-monitoring/README.md`,
  "Labels".
- A dashboard carries the tag `home-server` and no `links`, and uses the
  datasource uids `prometheus` and `loki`. The role adds the link bar.

A new service gets the generic alerts without a rule of its own:
`ContainerRestartLoop`, `UserUnitFailed`, the snapshot `JobStale` and the host
memory alerts.

## LLM coding tools

LLM-based coding tools write most of the code and documentation of this
project. The maintainer sets the goals and the design, reviews every change and
is responsible for it. Each change runs on a VM before it reaches a host.

## License

MIT
