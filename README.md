# bento.rs

A rootless, daemonless, low-level container runtime for Linux written in Rust, targeting
compliance with the [OCI Runtime Spec](https://github.com/opencontainers/runtime-spec).

> Originally started as a collaborative project at
> [`homebrew-ec-foss/bento.rs`](https://github.com/homebrew-ec-foss/bento.rs). After the
> upstream maintainers became unresponsive for 1+ year on open PRs, this fork continues
> development. See [Attribution](#attribution).

---

## Features

| Component | Status |
|---|---|
| Namespace isolation (PID, mount, UTS, user, IPC) | ✅ |
| Rootless rootfs setup via `pivot_root` | ✅ |
| OverlayFS union filesystem (lower/upper/work/merged) | ✅ |
| Seccomp syscall filtering (`libseccomp`, BPF) | ✅ |
| `create` / `start` / `list` | ✅ |
| `state` / `kill` / `delete` / `spec` | 🚧 stubbed |
| PTY support | 🚧 in progress |
| Cgroups | 📋 planned |

---

## Architecture

```
crates/
├── bento-cli/          # CLI (clap)
│   └── src/main.rs
└── libbento/
    └── src/
        ├── process.rs   # container lifecycle, fork/clone orchestration
        ├── fs.rs         # rootfs setup, pivot_root, mounts
        ├── overlayfs.rs  # OverlayFS layer management
        ├── seccomp.rs    # libseccomp filter construction/loading
        └── syscalls.rs   # low-level syscall wrappers
```

- **Namespaces**: created via `unshare`/`clone` (`nix::sched`). PID 1 inside the new PID
  namespace is obtained by cloning directly with `CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET`
  after user namespace/UID-GID mapping setup — unsharing first and cloning second does not
  yield PID 1.
- **Rootfs isolation**: mount propagation is reset to `MS_REC | MS_PRIVATE`, then
  `pivot_root` swaps in the container rootfs and unmounts the old root, removing host
  filesystem access.
- **OverlayFS**: standard 4-layer mount (read-only lower, writable upper, work dir for
  kernel bookkeeping, merged view used as container root), toggled via `--overlayfs`.
- **Seccomp**: config-driven BPF filter (default action + per-syscall allow/deny rules,
  OCI `linux.seccomp`-compatible) validated before being loaded via
  `prctl(PR_SET_SECCOMP, ...)`.

---

## Usage

```bash
cargo build --release

mkdir -p test-bundle/rootfs/bin
cp /bin/sh test-bundle/rootfs/bin/
# add config.json in test-bundle/

cargo run -- create --bundle test-bundle my-container
cargo run -- create --bundle test-bundle my-container --overlayfs
cargo run -- start my-container
cargo run -- list
```

---

## Roadmap

- [ ] `state`, `kill`, `delete`, `spec`
- [ ] Cgroups (CPU, memory limits)
- [ ] PTY-based interactive containers
- [ ] OCI conformance test suite

---

## Attribution

- **Rootless filesystem isolation** — initially scaffolded with @Alex-Hunterz, rewritten
  and completed here.
- **Seccomp** — implemented upstream (pending PR), completed and maintained here.
- **OverlayFS** — implemented independently, not present upstream.

---

## License

Licensed under the MIT License.
