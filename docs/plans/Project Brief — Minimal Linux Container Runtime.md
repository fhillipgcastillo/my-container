## 🧩 Project Brief — Minimal Linux Container Runtime (Go-based)
### 🎯 Goal

Create a Docker-like lightweight container runtime that allows isolated environments to share the same OS kernel, using unprivileged containers by default with optional privileged execution.
Built for learning, experimentation, and extensibility — not to replace Docker, but to deeply understand how containerization works.

---

### ⚙️ Core Characteristics

- Language: Go (primary) — alternatives: Rust, C, or Zig.
- Target Distros (MVP): Alpine (base), Arch Linux, Ubuntu.
- Runtime Model: CLI-based tool (myctr) controlling isolated processes using Linux kernel primitives (no daemon at MVP stage).
- Isolation Features:
  - Namespaces: user, pid, net, ipc, uts, mount.
  - Cgroups v2: CPU, memory, I/O control.
  - OverlayFS + tmpfs: for rootfs layering.
  - Network: optional virtual bridge or host networking.
  - Seccomp/AppArmor (later): sandbox enhancements.

### 🧱 MVP Scope

1. `myctr run`
- Spawns a process in new namespaces.
- Mounts a minimal rootfs.
- Runs /bin/sh or a user-specified command.

2. `myctr build`
- Parses a Containerfile.
- Creates a rootfs by layering instructions (FROM, COPY, RUN).

3. `myctr list / stop / rm`
- Manages running containers via stored PIDs & state.

4. Basic Networking (bridge mode)
- Optional root-level setup for veth pairs and namespace linking.

### 🧩 Architecture Overview

- CLI → Runtime API Layer → Kernel Interface
  - CLI: parses commands, calls runtime API.
  - Runtime: handles namespaces, mounts, cgroups.
  - Kernel interface: uses syscall or unix Go package.

- Config storage: /var/lib/myctr/containers/<id> or $HOME/.myctr.
- Images: Built via Containerfile, stored as tarballs/layers.

### 🔍 Key Technical Choices

- Unprivileged containers first: rely on userns + newuidmap/newgidmap.
- Privileged mode (optional): direct namespace + networking creation.
- Single binary design: self-contained, minimal dependencies.
- Custom Containerfile syntax: Docker-like, simpler parsing.
- Cross-distro strategy: test rootfs compatibility and minimal dependency use.

### 🧠 Learning Emphasis
- This runtime teaches:
- Linux kernel isolation (namespaces & cgroups)
- Root filesystem layering and chroot mechanics
- Process re-parenting & PID namespace control
- How Docker, Podman, and LXC work under the hood

### 🧩 Roadmap Highlights
| Phase | Focus                     | Deliverable               |
| ----- | ------------------------- | ------------------------- |
| 1     | Namespace setup           | Run isolated `/bin/sh`    |
| 2     | Rootfs + chroot           | Execute inside container  |
| 3     | Cgroups integration       | Basic resource control    |
| 4     | Containerfile parsing     | Build from minimal recipe |
| 5     | CLI UX                    | `run`, `list`, `stop`     |
| 6     | Privileged network bridge | Optional                  |
| 7     | Image cache & persistence | Local registry            |

### 🧰 MVP Essentials
- Namespaces: user, pid, mount
- Rootfs mounting
- Minimal Containerfile support
- Optional networking via bridge
- No daemon — direct CLI calls
- Container lifecycle management

