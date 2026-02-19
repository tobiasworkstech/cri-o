# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What CRI-O Is

CRI-O is an OCI-based implementation of the Kubernetes Container Runtime Interface (CRI). It provides a lightweight container runtime specifically for Kubernetes — it does NOT build, push, or sign images, and has no stable end-user CLI.

- Language: Go 1.25+
- Repo: <https://github.com/cri-o/cri-o>
- Release cycle: follows Kubernetes (n-2 version skew)

## Build

All builds that touch containers/image or storage code require these build tags to avoid a `pkg-config` dependency on gpgme:

```bash
make BUILDTAGS="containers_image_openpgp containers_image_ostree_stub" all
make BUILDTAGS="containers_image_openpgp containers_image_ostree_stub" test-binaries
```

The Makefile's default `make all` target builds `bin/crio` and `bin/pinns` plus docs. Key targets:

```bash
make all                    # Build crio + pinns + docs
make lint                   # golangci-lint (required before PR)
make prettier               # Format markdown/YAML/JSON
make verify-prettier        # Verify formatting (required before PR)
make verify-mdtoc           # Verify markdown TOC (required before PR)
make mockgen                # Regenerate mocks (commit the output)
make verify-dependencies    # Verify dependencies.yaml is consistent
```

## Testing

**Unit tests** use Ginkgo (`*_test.go`). Run all or a single package:

```bash
make testunit
go test -v ./internal/oci/... -run TestName
```

**Integration tests** use BATS (`test/*.bats`). Always use the runner script with `sudo -E`, and use relative paths (no `test/` prefix):

```bash
sudo make localintegration
sudo -E ./test/test_runner.sh version.bats
sudo -E ./test/test_runner.sh ctr.bats -f 'some pattern'
```

**Mocks** live in `test/mocks/*/` and are committed to git. Regenerate with `make mockgen` after changing an interface.

## Git Workflow

- **Always** `git commit -s` (DCO sign-off required)
- Prefer a **single commit per branch** for simple changes; amend rather than add commits: `git commit --amend -s`
- Use multiple commits only when logical separation aids review
- After amending: `git push --force-with-lease`
- Keep commit message, PR description, and docs synchronized as code evolves
- Do NOT link issues/PRs in commit messages unless asked

Use `.github/ISSUE_TEMPLATE/` for issues and `.github/PULL_REQUEST_TEMPLATE.md` for PRs (includes `/kind` label, what/why, release notes).

## Architecture

### Layer Stack

```
cmd/crio/          CLI + daemon startup (urfave/cli, cmux for gRPC+HTTP)
server/            CRI gRPC server (RuntimeService + ImageService handlers)
internal/lib/      ContainerServer — orchestrates sandboxes and containers
internal/oci/      OCI runtime abstraction + Container/state machine
pkg/config/        Public config types (RootConfig, RuntimeConfig, etc.)
```

`server.Server` embeds `*lib.ContainerServer`, which owns the in-memory stores, the `oci.Runtime` manager, the storage layer, and hook management.

### Runtime Abstraction (`internal/oci/`)

`oci.Runtime` is a strategy router that maps a runtime handler name to a `RuntimeImpl`. Two implementations:

- **`runtimeOCI`** (`runtime_oci.go`) — conmon-based. Spawns `conmon` as a monitor process, which in turn spawns crun/runc. Used for regular containers.
- **`runtimeVM`** (`runtime_vm.go`) — Kata Containers / VM-based. Communicates with a containerd shim v2 process over ttrpc. The `r.task` field holds the shim client; it is `nil` after CRI-O restarts until `updateContainerStatus` or `connectTask` reconnects.

The `RuntimeImpl` interface defines: `CreateContainer`, `StartContainer`, `StopContainer`, `ExecContainer`, `UpdateContainerStatus`, `DeleteContainer`, `PauseContainer`, `UnpauseContainer`, and streaming helpers.

### Container State Machine

States: `created → running → paused → stopped`

Every state transition that calls into the OCI runtime **must** hold `container.opLock` (write lock). Status reads use the read lock. This prevents races between concurrent CRI calls on the same container.

Other important locks on `oci.Container`:
- `metaLock` — protects spec and resources
- `stopLock` — prevents concurrent stop attempts
- `monitorProcessLock` — guards the conmon process reference

### Sandbox / Pod Lifecycle

`lib.ContainerServer` manages both sandboxes (`internal/lib/sandbox/`) and containers. A sandbox holds the infra container (pause pod) that owns the PID/IPC/network namespace. Containers are stored in a `memorystore.Storer[*oci.Container]`.

On startup, `server.restore(ctx)` walks persistent storage and reconstructs the in-memory state. For each pod: restore or clean up child containers. For each container: restore or release its name and mark the image for GC.

### Container Creation Flow

`server.CreateContainer` →
  1. Look up parent sandbox
  2. Generate OCI spec (seccomp, apparmor, capabilities, mounts, devices)
  3. Create bundle dir + mount rootfs via `storage.RuntimeServer`
  4. `runtime.CreateContainer()` — conmon spawns the OCI runtime (container is paused)
  5. NRI hooks
  6. Return container ID

`server.StartContainer` runs NRI hooks, then `runtime.StartContainer()` (conmon signals the runtime to exec).

`server.StopContainer` runs pre-stop hooks, `runtime.StopContainer(timeout)` (SIGTERM → SIGKILL), then post-stop hooks and unmounts.

### Config System (`pkg/config/`)

Three-level merge: struct defaults → file (`/etc/crio/crio.conf` + drop-ins in `/etc/crio/crio.conf.d/*.conf`) → CLI flags. Call `config.Validate(true)` at startup. Update `dependencies.yaml` whenever tool versions change, then run `make verify-dependencies`.

Public config lives in `pkg/config/`. Internal subsystem configs (seccomp, apparmor, NRI, RDT) live in `internal/config/`.

### Seccomp

`internal/config/seccomp/seccomp.go` handles seccomp profile application. The OCI spec generator (`generate.New()`) auto-populates a default seccomp filter — for **privileged** containers, the server code must explicitly set `g.Config.Linux.Seccomp = nil` to honor the CRI "unconfined" semantics. See `server/sandbox_run_linux.go:setupSandboxSeccomp` and `server/container_create.go:setupSeccomp`.

### Event System

`server.ContainerEventsChan` (capacity 1000) broadcasts lifecycle events when `EnablePodEvents` is set (evented PLEG). A single goroutine (`broadcastEvents`) fans events out to all subscribers via `sync.Map`.

## Key Files for Common Tasks

| Task | Files |
|------|-------|
| Add a config option | `pkg/config/config.go`, then update `docs/crio.conf.5.md` |
| Change container create logic | `server/container_create.go` |
| Change sandbox create logic | `server/sandbox_run_linux.go` |
| Change stop/kill behavior | `server/container_stop.go`, `internal/oci/runtime_oci.go` or `runtime_vm.go` |
| Change seccomp handling | `internal/config/seccomp/seccomp.go` |
| Change OCI spec generation | `internal/factory/container/container.go`, `server/container_create_linux.go` |
| Change image pull logic | `server/image_pull.go` |
| Add a CRI endpoint | `server/server.go` + new handler file |

## Common Pitfalls

- **Build on macOS**: Many files have `_linux.go` suffixes and use Linux-only packages (e.g., `go.podman.io/common/pkg/timezone`). The code only compiles on Linux — use `gofmt -e` for syntax checks on macOS.
- **Man pages**: Edit `.md` source files in `docs/`, not the generated output.
- **`dependencies.yaml`**: Must be kept in sync when any tool version changes. It is the authoritative version reference, not this file.
- **`r.task` in runtimeVM**: This field is `nil` after CRI-O restarts until the shim reconnects. Any function that calls `r.task.*` without a nil check will panic. Use `connectTask()` to safely reconnect.
- **opLock ordering**: Always acquire `container.opLock` before calling into the OCI runtime. Never call runtime methods without it.
