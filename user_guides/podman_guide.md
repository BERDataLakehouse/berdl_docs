# Running Containers in Your Notebook with Podman

Your BERDL notebook ships [rootless Podman](https://podman.io/), so tools that are
painful to install from source but published as container images — ODK / ROBOT,
format converters, small CLI utilities — can be run directly from a terminal or a
notebook cell, with no admin registration step.

This is for **lightweight, single-container work** inside your own notebook pod. For
heavy pipelines (InterProScan, GTDB-Tk, CheckM2, anything that wants many cores or a
reference-data mount) use the [CDM Task Service](https://github.com/kbase/cdm-task-service),
which schedules containers on the compute cluster.

| | Podman in the notebook | CDM Task Service |
|---|---|---|
| Use case | Quick CLI tools, hard-to-install software | Heavy compute, batch jobs, pipelines |
| Image control | Your choice | Admin-registered, digest-pinned |
| Parallelism | One container at a time, in your pod | Multi-container, cluster-scheduled |
| Resources | Your notebook pod's CPU/memory limits | High CPU/RAM, refdata mounts |

## Quick start

```bash
# Is it enabled on this pod?  (see "If podman is unavailable" below)
echo $BERDL_PODMAN_STATUS        # -> available

# Run a tool against files in your home directory
mkdir -p ~/work && cp my.owl ~/work/
podman run --rm -v ~/work:/work docker.io/obolibrary/odkfull \
    robot convert --input /work/my.owl --output /work/my.obo

# Everyday commands work as on a laptop
podman pull docker.io/library/python:3.12-slim
podman images
podman run --rm -it docker.io/library/python:3.12-slim python
podman build -t mytool ~/mytool      # from a Containerfile / Dockerfile
podman rmi mytool                    # reclaim space
```

From a notebook cell, prefix with `!`:

```python
!podman run --rm -v ~/work:/work docker.io/library/alpine:3.20 sh -c 'wc -l /work/*.tsv'
```

## How it works here

- **Rootless and daemon-less.** Containers run as *you*, inside your notebook pod,
  in a user namespace. There is no Docker daemon and nothing to start.
- **Images live in your home.** Pulled images and build layers are stored under
  `~/.local/share/containers/storage`, so they survive notebook restarts — and count
  against your home directory space. `podman images`, `podman rmi <image>` and
  `podman system prune -a` keep it tidy; `podman system reset` wipes everything.
- **Images must be `linux/amd64`.** The notebook nodes are x86_64; there is no
  emulation.
- The first `podman` command of a session prints
  `no subuid ranges found ... Using rootless single mapping` — that is expected (see
  below) and does not repeat. A `"/" is not a shared mount` warning is also harmless.

## What is different from Docker on a laptop

**Single user ID.** Only your own user ID is mapped into containers: `root` inside a
container is you outside. Files a container writes into a bind mount are owned by you,
which is what you want. The flip side is that a container cannot *switch* to another
user — an image whose Dockerfile ends with `USER app` fails with
`cannot setresgid ... Invalid argument`. Override the image's user:

```bash
podman run --rm --user 0 docker.io/some/image-that-drops-privileges ...
```

`--user <anything but 0>` is not supported.

**Networking is the pod's.** Containers share the notebook pod's network namespace.
A server inside a container is reachable at `localhost:<container port>` from your
notebook and at the pod IP from elsewhere; `-p host:container` is a no-op. Use
`--network none` for a fully offline run. Registry pulls go out through the pod's
normal egress.

**No resource flags.** `--memory`, `--cpus` and friends are ignored (the nodes run
cgroup v1, which rootless containers cannot delegate). The notebook pod's own CPU
and memory limits bound everything you run, containers included — a container that
exhausts the pod's memory is OOM-killed like any other process in it.

**`/proc` is the pod's.** Containers still get their own PID namespace, but `ps`
inside one lists the notebook pod's processes too. Nothing else leaks in: the
container's filesystem view is just the image plus what you `-v` mount.

**Not available:** `--privileged`, `--device` (no `/dev/fuse`, no GPUs), nested
podman / Docker-in-Docker, `podman-compose`, and Podman pods with port mapping.

## If podman is unavailable

```bash
$ echo $BERDL_PODMAN_STATUS
unavailable
```

Rootless containers need the kernel to allow `mount` inside a user namespace, and
the cluster's default container policy (AppArmor) forbids it. Notebook pods must be
admitted with a profile that allows it; that is a platform setting
(`BERDL_NOTEBOOK_APPARMOR_PROFILE` on the JupyterHub deployment), not something a
notebook can change. If you see `unavailable`, ask a BERDL administrator whether
container support is enabled for your environment.

Typical errors and what they mean:

| Error | Cause / fix |
|---|---|
| `crun: ... mount ... Operation not permitted` (any mount) | Pod not admitted with the required AppArmor profile — `BERDL_PODMAN_STATUS` will be `unavailable`. |
| `cannot setresgid to 65534: Invalid argument` | Image switches to a non-root user; add `--user 0`. |
| `exec format error` | Image is not `linux/amd64`. |
| `'overlay' is not supported over overlayfs` | Home directory storage is not on a regular filesystem (should not happen on BERDL — report it). |
| `no space left on device` | Clean up: `podman system prune -a`. |

## Reference

- Podman defaults for BERDL live in `/etc/containers/containers.conf`,
  `/etc/containers/storage.conf` and `/etc/containers/containers.conf.d/`; per-user
  overrides go in `~/.config/containers/containers.conf`.
- Setup happens at pod boot in `before-notebook.d/15-podman-setup.sh`
  (subordinate IDs, `XDG_RUNTIME_DIR`, the `/proc` bind-mount default, the
  feasibility check that sets `BERDL_PODMAN_STATUS`).
- Original request: [cdm-task-service#739](https://github.com/kbase/cdm-task-service/issues/739)
  / [spark_notebook#262](https://github.com/KBaseDataLakehouse/spark_notebook/issues/262).
