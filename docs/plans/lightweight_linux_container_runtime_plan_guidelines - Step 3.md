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

