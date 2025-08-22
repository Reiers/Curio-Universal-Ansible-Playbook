# Curio Build & Deploy – Universal Ansible Playbook

This repository contains a *universal* Ansible playbook that builds and deploys **Curio** from Git across one or many hosts.  


> Playbook file: **`curio_update_universal.yml`**

---

## TL;DR

```bash
ansible-playbook -i inventory curio_update_universal.yml \
  -l curio_nodes \
  -e curio_path=/opt/curio \
  -e build_user=mybuilder \
  -e curio_service_name=curio.service \
  -e build_target=build \
  -e use_cuda=true
```

- Performs rolling updates (`serial: 1`) host-by-host.
- Installs build dependencies, checks out Curio, ensures Rust, builds, stops the service, installs, reloads systemd, restarts, and verifies `curio --version`.

---

## What the Playbook Does

1. **Prereqs**: Installs common build packages (Debian/Ubuntu + RHEL/CentOS/Fedora).
2. **Source**: Clones/updates Curio from Git at the branch you specify.
3. **Toolchain**: Ensures Rust (installs with rustup if missing) and sources Cargo env.
4. **Build**: Runs `make clean <target>` with optional CUDA flags.
5. **Deploy**: Stops the systemd unit (if present), runs `make install` as root.
6. **Service**: Reloads systemd if a unit exists, enables & starts it.
7. **Verify**: Prints `curio --version` for each host.

> The play is designed to be **idempotent** where possible. The `make` build will run each time you invoke the playbook.

---

## Requirements

- **Ansible**: 2.12+ recommended.
- **SSH access** to each target with privilege escalation (sudo).
- **Python 3** on target hosts (Ansible default).
- **Outbound HTTPS** to fetch Git & Rustup.
- **Supported OS families**: Debian/Ubuntu and RHEL/CentOS/Fedora.
- **CUDA / NVIDIA drivers** are **not** installed by this playbook. If you set `use_cuda=true`, ensure drivers and CUDA runtime are preinstalled on the host(s).

---

## Inventory Example

```ini
[curio_nodes]
node1 ansible_host=10.0.0.11
node2 ansible_host=10.0.0.12
```

Limit to a subset during a rollout:
```bash
ansible-playbook -i inventory curio_update_universal.yml -l node1
```

---

## Variables (with defaults)

These can be overridden via `-e key=value` or vars files.

| Variable | Default | Description |
|---|---|---|
| `curio_repo` | `https://github.com/filecoin-project/curio.git` | Git repo to clone |
| `curio_branch` | `main` | Git branch/commit/tag to checkout |
| `curio_path` | `/opt/curio` | Local checkout directory on targets |
| `build_user` | `ansible_user` or `root` | User that owns source and runs the build |
| `curio_service_name` | `curio.service` | systemd unit to manage (if present) |
| `build_target` | `build` | Make target: e.g. `build`, `build-calibnet` |
| `use_cuda` | `true` | Enable CUDA seal (`FFI_USE_CUDA_SUPRASEAL=1`) |
| `build_env` | see below | Build environment passed to `make` |

**Default `build_env` map**
```yaml
build_env:
  RUSTFLAGS: "-C target-cpu=native -g"
  FFI_BUILD_FROM_SOURCE: "1"
  FFI_USE_CUDA_SUPRASEAL: "{{ '1' if use_cuda | bool else omit }}"
```
> To customize, you can override the entire `build_env` mapping with `-e 'build_env={...}'`.

---

## Tags

Speed up runs by targeting phases:
- `deps` – OS packages, rust/cargo toolchain
- `code` – git checkout/update
- `build` – compile Curio
- `deploy` – stop → install → daemon-reload → start

Examples:
```bash
ansible-playbook -i inventory curio_update_universal.yml --tags deps,code
ansible-playbook -i inventory curio_update_universal.yml --tags build,deploy
```

---

## Common Scenarios

### 1) Match an older single-host layout
```bash
-e curio_path=/home/user/curio -e build_user=user
```

### 2) CPU-only hosts
```bash
-e use_cuda=false
```

### 3) Build for Calibnet (example)
```bash
-e build_target=build-calibnet
```
> Ensure the target exists in Curio’s `Makefile`.

### 4) Pin to a specific branch or commit
```bash
-e curio_branch=vX.Y.Z
# or a commit hash
-e curio_branch=abcdef1234567890
```

### 5) Custom repo fork
```bash
-e curio_repo=https://github.com/yourorg/curio.git -e curio_branch=my-feature
```

---

## Systemd Expectations

The playbook manages a service **only if a matching unit file exists**, e.g.:

`/etc/systemd/system/curio.service`
```ini
[Unit]
Description=Curio
After=network.target

[Service]
ExecStart=/usr/local/bin/curio run
Restart=always
User=curio
Group=curio
WorkingDirectory=/var/lib/curio
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

If no unit is present, the build/install still completes. Create your unit, then rerun the play to start/enable it.

---

## Rolling Updates

The play uses `serial: 1` so only one host is upgraded at a time. This minimizes risk. If you want parallelism, edit the playbook or run separate batches via `-l` limits.

---

## Dry Runs (`--check`)

Most shell build steps are **not check-mode safe**. Use `--check` only for dependency & template validation; expect build/deploy tasks to be skipped or report changed incorrectly in check-mode.

---

## Troubleshooting

- **Rust not found**: The play auto-installs Rust with `rustup` for the `build_user` and sources `~/.cargo/env`. If your environment sets a custom toolchain, pass it via env:
  ```bash
  -e 'build_env={"RUSTUP_TOOLCHAIN":"stable"}'
  ```

- **CUDA issues**: Ensure NVIDIA drivers and a compatible CUDA runtime are present on the host(s). The play only sets `FFI_USE_CUDA_SUPRASEAL` when `use_cuda=true`.

- **Unit not started**: If the systemd unit file is missing or named differently, set `-e curio_service_name=...` and/or create the unit file.

- **Permissions**: Make sure `build_user` owns `curio_path` and can write to it. The play ensures directory ownership but cannot fix arbitrary parent perms.

- **Corporate proxies**: Configure Git and environment on the host (or extend `build_env`) to reach external resources.

---

## Security Notes

- `make install` runs as root. Inspect Curio’s `Makefile` before deploying to production to understand what is installed and where.
- Keep your inventory and group_vars private; they may contain hostnames and credentials.

---

## Outputs & Verification

- Binaries are typically installed to `/usr/local/bin` (per Curio’s `Makefile`).
- Final step runs:
  ```bash
  curio --version
  ```
  and prints the version in the Ansible summary per host.

---

## Contributing / Customizing

- Extend the OS package lists for your distro variants.
- Add handlers if you prefer handler-driven service restarts.
- Split the play into a role if you want reusable defaults and molecule tests.
- PRs welcome for additional distro support and CUDA helpers.

---

## License

MIT (or your project’s license).

