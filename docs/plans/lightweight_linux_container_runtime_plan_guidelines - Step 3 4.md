# Lightweight Linux Container Runtime — Plan & Guidelines (Step 3 Expanded)

> Continuing from Step 2 — we now define the **Planning & MVP Roadmap**, focusing on implementation phases, deliverables, testing milestones, and architecture alignment.

---

## Step 3 — Planning & MVP Roadmap

### 🎯 Primary Goal
Deliver a working **lightweight container runtime** prototype in Go, capable of running isolated environments with rootless (unprivileged) support, optional privileged mode, and minimal dependencies — similar in concept to Docker but simpler and educationally transparent.

---

### 🧭 Phase Overview

| Phase | Title | Objective | Key Deliverables |
|-------|--------|------------|------------------|
| 0 | **Environment Setup & Research** | Validate host kernel and environment compatibility | Verified user namespaces, cgroups v2, mount and unshare syscall access on Arch + Ubuntu |
| 1 | **Proof-of-Concept Namespace Runner** | Demonstrate isolated shell with Go-based unshare sequence | Run `/bin/sh` inside isolated PID/NET/MNT/USER namespaces |
| 2 | **Container Runtime Core** | Build `myctr run` CLI with minimal config and rootfs mounting | Create `myctr run <rootfs>` → executes container process in isolation |
| 3 | **Image Builder & Management** | Implement custom image format and `Containerfile` parsing | `myctr build -t <name>` generates `.tar` + metadata.json images |
| 4 | **Networking Support** | Add slirp4netns for rootless mode, veth + bridge for privileged | Full container connectivity with host or isolated network |
| 5 | **Resource & Lifecycle Management** | Introduce cgroup v2 limits, logs, container cleanup, and lifecycle commands | `myctr ps`, `myctr stop`, `myctr rm`, logging and cleanup support |
| 6 | **Stabilization & Polish** | Refine UX, error handling, and security | MVP ready for testing and documentation |

---

### 🧱 Detailed Milestones

#### Phase 0 — Setup & Validation (1 week)
- Verify kernel support for namespaces (`cat /proc/self/status` → `CapEff` includes `setuid`)
- Test userns unprivileged creation (`unshare -Urn bash`)
- Confirm OverlayFS availability and permissions
- Prepare minimal rootfs from Arch/Ubuntu via `debootstrap` or `pacstrap`

Deliverables:
- Compatibility checklist for Arch and Ubuntu
- Dev environment with Go 1.22+, slirp4netns installed

---

#### Phase 1 — Namespace Runner (2–3 weeks)
- Implement low-level namespace setup in Go:
  - Clone new namespaces (PID, UTS, NET, MNT, IPC, USER)
  - Configure UID/GID mappings using `/etc/subuid` and `/etc/subgid`
  - Mount `/proc` and minimal root filesystem
- Execute `/bin/sh` in isolated environment

Deliverables:
- Run isolated process from Go binary
- Output showing different PID space and hostname

Testing:
- Compare `ps aux` inside and outside container
- Validate user mapping (`whoami` inside → root)

---

#### Phase 2 — Runtime CLI & RootFS Handling (3–4 weeks)
- Add CLI (Cobra-based): `myctr run`, `myctr list`, `myctr delete`
- Manage state under `/var/lib/myctr/containers/<id>/`
- Support mounting of rootfs (OverlayFS if possible)
- Handle cleanup and mount teardown

Deliverables:
- Functional CLI binary
- Container lifecycle management (start, list, remove)

Testing:
- Start multiple containers simultaneously
- Verify isolation between containers

---

#### Phase 3 — Image Building & Metadata (3 weeks)
- Implement basic `Containerfile` parser (FROM, COPY, RUN, CMD)
- Build rootfs from instructions
- Save metadata.json: image name, layer info, commands, env vars
- Export/import via `myctr image save/load`

Deliverables:
- Functional image build and import/export
- Local image registry under `/var/lib/myctr/images/`

Testing:
- Build and run sample image (e.g., alpine with busybox)

---

#### Phase 4 — Networking Support (3 weeks)
- Integrate **slirp4netns** for unprivileged networking
- Add optional **veth + bridge** setup for privileged containers
- Handle port forwarding and DNS resolution

Deliverables:
- Working network in both unprivileged and privileged containers
- CLI flag: `--net host|none|bridge`

Testing:
- `ping`, `curl`, or small HTTP server in containers

---

#### Phase 5 — Lifecycle Management & cgroups (3 weeks)
- Add basic CPU/memory limits using cgroup v2
- Introduce container logging (`/var/log/myctr/<id>.log`)
- Implement cleanup and `myctr ps` listing

Deliverables:
- Resource-managed container runtime
- Persistent log and lifecycle management

Testing:
- Verify container process isolation post-stop
- Simulate resource stress and validate cgroup enforcement

---

#### Phase 6 — Stabilization & Documentation (2–3 weeks)
- Improve CLI UX and error messages
- Document code structure and build flow
- Prepare developer docs (GoDoc, README)
- Write simple integration tests for CI

Deliverables:
- MVP complete, ready for alpha testing
- Developer documentation + usage examples

---

### 🧩 Cross-Phase Considerations
- **Security:** Drop all non-essential capabilities, enforce mount flags (`noexec`, `nodev`, `nosuid`)
- **Performance:** Optimize namespace creation and rootfs caching
- **Extensibility:** Design internal APIs to later integrate with an HTTP API daemon
- **Testing:** Local test suite using Go + Bash integration scripts

---

### 🚀 MVP Completion Criteria
✅ Run multiple isolated processes concurrently  
✅ Support both rootless and privileged containers  
✅ Build and run basic images  
✅ Provide minimal logging and cleanup  
✅ Network connectivity for rootless (via slirp4netns)  
✅ Stable CLI UX for basic management

---

### 🗂️ Suggested Project Structure
```
myctr/
├── cmd/
│   └── myctr/               # CLI entrypoint (cobra commands)
├── pkg/
│   ├── runtime/             # Namespace, process, mount logic
│   ├── builder/             # Containerfile parsing and image creation
│   ├── image/               # Image metadata and management
│   ├── net/                 # Networking setup (slirp4netns, bridge)
│   └── utils/               # Helpers (logging, fs, error handling)
├── internal/
│   ├── syscalls/            # Low-level Go syscall wrappers
│   └── config/              # Constants, defaults, paths
├── tests/
│   ├── integration/
│   └── unit/
└── go.mod
```

---

### 📘 Next Steps
Step 4 → **Full Canvas Detailing & Implementation Guidelines**  
This will cover code-level guidelines, syscall sequences, folder-by-folder responsibility breakdown, and example Go scaffolding to start coding the MVP.

---

## Step 4 — Full Implementation Guidelines (Go-focused)

This section provides detailed, actionable instructions to implement the MVP in Go. It includes syscall flows, package responsibilities, Containerfile spec, CLI behaviour, and sample code scaffolding to get a working `myctr run` prototype quickly.

### 1. Project Layout (refined)
```
myctr/
├── cmd/myctr/                # CLI (cobra)
│   └── main.go
├── pkg/runtime/              # Core runtime logic
│   ├── run.go                # main run flow
│   ├── namespaces.go         # namespace creation & mapping
│   ├── mounts.go             # mount helpers (overlay, proc, sysfs, dev)
│   └── exec.go               # exec into container (TTY handling)
├── pkg/builder/              # Containerfile parsing, build pipeline
│   └── builder.go
├── pkg/image/                # image metadata, storage, import/export
├── pkg/net/                  # networking helpers (slirp4netns wrapper, veth/bridge)
├── pkg/cgroup/               # cgroup v2 management
├── pkg/seccomp/              # optional seccomp profiles (json)
├── pkg/log/                  # logging & log file rotation
├── internal/sys/             # low-level syscall wrappers
└── tests/
```

---

### 2. Syscall & Runtime Sequence (myctr run)

Below is the sequence the runtime must perform to start a container. Implement most steps in `pkg/runtime/run.go`.

1. **Validate environment**
   - Ensure `/proc/self/uid_map` writable for userns mapping if unprivileged.
   - Check cgroup v2 mount (`/sys/fs/cgroup`) availability.

2. **Create working dirs**
   - `/var/lib/myctr/containers/<id>/{upper,work,merged,proc,sys,dev}`

3. **Set up overlayfs or bind-mount rootfs**
   - If privileged and overlay available: mount overlay using `mount(2)` with `lowerdir`, `upperdir`, `workdir`.
   - If unprivileged and overlay not permitted: bind-mount a prepared rootfs or copy minimal files into upperdir.

4. **unshare / clone**
   - Use `clone()` via `syscall.RawSyscall` or `unix.Clone` in Go with flags: `CLONE_NEWUSER | CLONE_NEWNS | CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNET | CLONE_NEWIPC` (use only required ones).
   - For Go, create a child process running a small helper binary to enter namespaces (see notes below about Go runtime threads and `unshare`).

5. **Set UID/GID mapping (for userns)**
   - Write to `/proc/<pid>/uid_map` and `/proc/<pid>/gid_map` with ranges derived from `/etc/subuid` and `/etc/subgid` or fallback to mapped user.
   - Use `/proc/<pid>/setgroups` write `deny` before writing gid_map if required.

6. **Mount pseudo filesystems** (inside mount namespace)
   - `mount -t proc proc <merged>/proc -o nosuid,noexec,nodev
   - mount sysfs and tmpfs for /dev
   - setup devpts for pty handling

7. **pivot_root / chroot**
   - Use `pivot_root` to switch root to merged dir, falling back to chroot when pivot_root unavailable.
   - After pivot, make old root private and unmount.

8. **Setup cgroups**
   - Create a cgroup under `/sys/fs/cgroup/myctr/<id>` and set `memory.max`, `cpu.max`, and `pids.max` as requested.
   - Add the container pid to the `cgroup.procs` file.

9. **Networking**
   - For unprivileged: launch `slirp4netns` with the child netns fd and configure DNS/port forwarding.
   - For privileged: create veth pair, add the host side to a bridge, set IP and bring up link.

10. **Drop capabilities & apply seccomp**
    - Use libseccomp or `prctl` with `PR_CAPBSET_DROP` to remove unwanted capabilities.

11. **Exec the user command**
    - `execve` the container's `CMD`, attach stdio to host TTY (or run detached).

12. **Teardown on exit**
    - Remove mounts, cgroups, and any created network interfaces. Rotate logs.

---

### 3. Implementation Notes & Go Specifics

- **Go & unshare caveat**: The Go runtime creates multiple threads; unsharing namespaces (e.g., userns) must be done in a new process rather than the current multi-threaded process. Use `syscall.ForkExec`/`unix.Clone` or implement a small helper binary (e.g., `nsenter-helper`) that performs `unshare()` and then `exec` the runtime child.

- **Using setns**: For some operations it's simpler to `fork` a child, then from the parent do UID/GID map writes targeting the child's `/proc/<pid>/uid_map`.

- **Privileges**: For operations requiring privileges (creating bridge, mounting overlay, setting capabilities), perform them only if running as root or via a setuid helper designed carefully.

- **slirp4netns**: Communicate via file descriptors and use `--disable-host-loopback` if you want isolation. Launch as a child process and monitor its lifecycle.

- **Seccomp**: Start with a permissive profile and iteratively tighten. Keep profiles as JSON files under `/etc/myctr/seccomp/`.

---

### 4. Sample Go Pseudocode Flow (runtime/run.go)
```go
func RunContainer(cfg RunConfig) error {
    id := newID()
    // 1. Prepare directories
    prepareContainerDirs(id)

    // 2. Prepare rootfs (overlay or rootfs bind)
    setupRootfs(cfg.ImagePath, id)

    // 3. Fork/exec child to create namespaces
    cmd := exec.Command("/proc/self/exe", "--child", id)
    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWUSER | syscall.CLONE_NEWNS | syscall.CLONE_NEWPID | syscall.CLONE_NEWUTS | syscall.CLONE_NEWNET,
        // Credential mapping handled after fork
    }
    // wire stdio
    cmd.Stdin = os.Stdin
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    if err := cmd.Start(); err != nil { return err }

    // Parent: write uid_map/gid_map for child
    writeUIDGIDMap(cmd.Process.Pid, cfg.UIDMap)

    // Parent: setup cgroups and add pid
    createCgroup(id, cmd.Process.Pid, cfg.Resources)

    // Parent: if using slirp4netns, launch it and connect to child's netns

    // Wait for child
    return cmd.Wait()
}
```

**Note:** The `--child` mode inside the same binary should implement the mount, pivot_root, mount proc, and finally `execve` the target process.

---

### 5. Containerfile Spec (Minimal)
```
FROM arch:latest
COPY ./app /app
RUN pacman -S --noconfirm python
ENV PORT=8080
CMD ["/app/start.sh"]
```

- **FROM**: base image name (maps to local tarball or remote import later)
- **COPY**: copy files from build context into image layer
- **RUN**: run command in temporary build namespace to mutate layer
- **CMD**: default command at runtime

---

### 6. Builder Strategy
- Implement `myctr build` as a sequence of ephemeral namespace runs:
  1. Extract base image to a temporary dir
  2. For each `RUN`, `exec` a command inside an ephemeral namespace using chroot/pivot_root to the build dir, commit changes to a new layer (tar)
  3. For `COPY`, add files directly into the build dir
  4. Produce final image tarball and metadata.json

- Keep builder simple: no advanced caching or layer optimization in MVP.

---

### 7. Networking Implementation Details
- **Unprivileged (slirp4netns)**:
  - Start child netns, then run `slirp4netns -c childpid tapname` and read the assigned IP.
  - Configure `/etc/resolv.conf` inside container root to point to resolv.conf provided by host or slirp.

- **Privileged (veth + bridge)**:
  - Host creates `veth_host` and `veth_cont`, add host end to `myctr0` bridge, set IP on container side, then move `veth_cont` to container netns.
  - Use `ip link add`, `ip link set`, `ip addr add`, and `ip netns exec` as needed.

---

### 8. Cgroup v2 Placement Example
- Create cgroup: `/sys/fs/cgroup/myctr/<id>/`
- Write limits: `echo "500M" > memory.max`
- Add pid: `echo <pid> > cgroup.procs`

Implement operations in Go by writing to these files using `os.WriteFile` with proper permissions.

---

### 9. Testing & CI Recommendations
- Use QEMU or nested VM in CI to test privileged operations safely.
- For unprivileged tests, run in GitHub Actions with `--privileged` runner when required.
- Provide a test matrix for: Arch Linux, Ubuntu 22.04, Go versions.

---

### 10. Security Checklist for MVP
- Validate UID/GID mappings are correct and non-overlapping
- Ensure `setgroups` is set to deny before writing `gid_map` when required
- Mount `/proc` with `nosuid,noexec,nodev`
- Drop capabilities not required by running processes
- Avoid suid helpers; if required, minimize code area and audit thoroughly

---

### 11. Example Development Tasks (first 10 tickets)
1. Create repository structure and CLI skeleton (Cobra) — 1 day
2. Implement `prepareContainerDirs` and rootfs mounting helper — 2 days
3. Implement child process spawn with clone flags and `--child` mode — 3 days
4. Implement uid/gid mapping writer helper — 1 day
5. Implement `mountProc`, `mountSys`, and `setupDev` helpers — 2 days
6. Implement basic cgroup v2 create and assign — 2 days
7. Integrate slirp4netns for test network — 2 days
8. Implement `myctr ps` and container state listing — 1 day
9. Add logging and basic cleanup commands — 1 day
10. Write integration test that starts a shell and runs `whoami` and `ps` — 1 day

---

### 12. Deliverables for This Step
- Detailed syscall sequence (above) and Go pseudocode
- Full folder layout and file ownership mapping
- Containerfile spec and builder algorithm
- First-10-ticket list to start implementation

---

_End of Step 4 — Full Implementation Guidelines_

(If you'd like, I can now generate the starter Go prototype for `myctr run --rootfs <path>` that performs: directory setup, namespace creation via `exec` trick, basic mounts and exec into /bin/sh. Say **yes** and I'll produce the runnable Go code.)
Step 4 → **Full Canvas Detailing & Implementation Guidelines**  
This will cover code-level guidelines, syscall sequences, folder-by-folder responsibility breakdown, and example Go scaffolding to start coding the MVP.

