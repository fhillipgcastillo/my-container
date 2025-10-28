# Lightweight Linux Container Runtime — Plan & Guidelines (Updated Clarified Direction)

> Refined with clarified project vision: unprivileged by default, Go-based implementation, supporting Ubuntu/Debian and Arch Linux for MVP.

---

## 1 — Project Goal

Build a lightweight Docker-like container system for Linux that:
- Uses the host kernel (no hypervisor)
- Provides isolated environments using Linux namespaces
- Works **unprivileged by default**, with optional privileged mode for advanced features
- Allows installation and management of separate environments (rootfs/images)
- Prioritizes simplicity and clarity of design (educational + practical use)

---

## 2 — Core Design Choices

### Privilege Model
- **Default:** unprivileged mode via user namespaces (root inside container ≠ root on host)
- **Optional:** privileged mode for full control (veth networking, overlay mounts, etc.)

### Language
- **Primary:** Go (best mix of power and simplicity)
- **Alternatives:** Rust (secure but complex), C (low-level), Python (for testing or tooling)
- Go ecosystem is mature for containers (`runc`, `containerd`, etc.) and perfect for a self-contained CLI runtime.

### Target Distros
- **Recommended:** Ubuntu 22.04+ / Debian 12+ (stable defaults)
- **Also supported for MVP:** Arch Linux (rolling kernel, good namespace and cgroup support)
- Avoid very old distros (missing cgroup v2 or userns features)

### Image Format
- **Custom tar + metadata.json format** for MVP (simpler, fully controlled)
- Add optional Docker/OCI image import support later

### Architecture
- **Single CLI binary** (`myctr`) for MVP
  - Handles running containers, building images, managing state
  - Daemon/API can be added later for persistent background management

---

## 3 — MVP Essentials

| Priority | Feature | Description |
|-----------|----------|--------------|
| ⭐⭐⭐ | `run` with rootfs + userns | Core function: run isolated process in namespaces |
| ⭐⭐ | Build from Containerfile | Define and build container images from recipes |
| ⭐⭐ | Basic FS handling | OverlayFS or direct rootfs mount (depending on privilege) |
| ⭐⭐ | User-mode networking | slirp4netns for unprivileged containers |
| ⭐ | Privileged mode networking | veth + bridge setup for root execution |
| ⭐ | cgroup v2 resource limits | CPU/memory limits, add later in Phase 2 |
| ⭐ | seccomp/capabilities | Optional security hardening after MVP |

---

## 4 — Early Key Considerations

### Logging / Output
- Capture container stdout/stderr to a log file under `/var/log/myctr/<container-id>.log`.
- Allow `myctr logs <container>` to tail these logs.

### UID/GID Mapping
- Use `/etc/subuid` and `/etc/subgid` to map container root → host unprivileged user.
- Support fallback mapping if unavailable.

### Testing & Safety
- Implement integration tests in isolated namespaces.
- Avoid system-wide mounts or cgroup persistence during tests.
- Provide cleanup utilities for failed containers.

---

## 5 — Recommended Development Path

### Phase 0 — Preparation (1–2 weeks)
- Validate environment on Ubuntu/Debian and Arch.
- Create proof-of-concept `unshare + chroot` shell script.

### Phase 1 — Core Runtime (2–4 weeks)
- Implement `myctr run` with userns, mount `/proc`, `/sys`, `/dev`, and exec target binary.
- Manage container state via JSON in `/var/lib/myctr/containers/`.

### Phase 2 — Image Build System (3 weeks)
- Parse minimal `Containerfile` (FROM, COPY, RUN, CMD)
- Generate tar image with metadata.json
- Implement `myctr build -t <name> .`

### Phase 3 — Networking & Privileged Support (3 weeks)
- Add slirp4netns integration for unprivileged networking
- Add veth + bridge for privileged mode

### Phase 4 — Polish & Hardening (3–4 weeks)
- Logging system, UID mapping refinements, cleanup commands
- Add optional cgroup v2 limits
- Start planning daemon API structure

---

## 6 — Tools & Libraries

- **Language:** Go 1.22+
- **Core Libraries:**
  - `golang.org/x/sys/unix` — namespace syscalls
  - `containers/storage` or inspiration from `runc/libcontainer`
  - `github.com/spf13/cobra` — CLI framework
  - `slirp4netns` (external binary) for user-mode networking
- **Testing:** Go test + integration scripts using `unshare` in CI

---

## 7 — Example MVP Command Flow

```
# Build image
myctr build -t myapp:local .

# Run container (unprivileged)
myctr run --name myapp myapp:local

# Run container (privileged)
sudo myctr run --privileged --name myapp myapp:local

# View logs
myctr logs myapp

# Stop & remove
myctr stop myapp && myctr rm myapp
```

---

## 8 — Key MVP Challenges

1. **User namespaces permissions** — verify `unprivileged_userns_clone=1` on target kernel.
2. **OverlayFS with userns** — may need privileged helper; fallback to simple rootfs copy for MVP.
3. **Network setup** — slirp4netns integration must handle file descriptors cleanly.
4. **Error isolation** — failed mounts or syscalls must rollback cleanly to avoid orphaned namespaces.

---

## 9 — Immediate Next Step

Generate next-level plan:
- Define Go project folder structure (`cmd/`, `pkg/runtime/`, `pkg/builder/`, etc.)
- Outline syscall sequence for `myctr run`
- Start minimal proof-of-concept code to spawn a container shell using namespaces

---

_End of updated plan._

