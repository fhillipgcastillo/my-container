# Project: Lightweight Linux Container System (Docker-like MVP)

---

## Step 1: Clarification & Exploration

**Goal Recap:**  
Create a minimal Linux container system similar to Docker, designed for isolated environments that share the same host OS kernel. It should enable creating, managing, and running isolated spaces (containers) where users can install and manage software independently. The system leverages existing OS resources but enforces isolation using namespaces, cgroups, and filesystem layers.

### Key Questions for Clarity

#### 1. Environment & Scope
- Should this system run **only on Linux** (e.g., Arch, Ubuntu, Alpine), or do you plan cross-platform support (via WSL2, etc.) later?
- Will the MVP target **CLI-based management** first (like Docker CLI), or include a **daemon + API model** (like `dockerd`)?

#### 2. Isolation Goals
- How strict should isolation be? (Process & network only, or full filesystem, user, and IPC isolation?)
- Do you need **security-focused isolation** (like Docker’s seccomp/apparmor integration) in the MVP, or just namespaces + cgroups for now?

#### 3. Resource Management
- Should resource limits (CPU, memory, disk) be configurable per container in the MVP?
- Do you want automatic cleanup (stopped containers, temporary images, etc.)?

#### 4. Storage Model
- Should container layers be **copy-on-write** (e.g., overlayfs)?
- Do you want persistent volumes or just ephemeral filesystems for MVP?

#### 5. Networking
- Should networking be implemented via **network namespaces** only, or include a **bridge model** like Docker’s `docker0`?
- Will containers communicate with the host or each other?

#### 6. Distribution & Portability
- Should this system support **image export/import** for portability between systems?
- Do you want a **custom lightweight image format**, or to reuse existing tarballs/filesystems?

#### 7. OS Base
- MVP will include support for **Arch Linux**, **Alpine**, and **Ubuntu**.
- Should each distro be supported as a **base container image**, or will the host OS dynamically serve as the base (shared rootfs)?

#### 8. Developer Experience
- CLI style: Should it resemble Docker commands (`run`, `exec`, `ps`, etc.) or be simpler for now?
- Do you want a YAML-based definition file (like Docker Compose) later?

#### 9. Long-term Vision
- Is the goal to **compete with Docker/Podman** or to **build a lightweight educational/research system** focused on modular architecture and kernel features?

---

## Step 2: Architectural & Technical Considerations

### 1. Core Components Overview
- **Namespace Management Layer**: Controls process, mount, PID, IPC, UTS, and network isolation.
- **Cgroup Controller**: Handles resource limits (CPU, memory, IO, pids).
- **Filesystem Layer**: Uses OverlayFS or UnionFS to create isolated root filesystems.
- **Image & Container Manager**: Handles lifecycle — creation, start, stop, and deletion.
- **Networking Module**: Configures virtual interfaces, bridges, and DNS.
- **CLI / Command Handler**: Provides user interface for container operations.

---

### 2. Namespace Isolation Strategy
Namespaces form the foundation for process isolation. You’ll implement wrappers or helper scripts to create isolated environments using `unshare` and/or `clone()` syscalls.

- **PID namespace** → Isolates process trees.
- **Mount namespace** → Enables each container to have its own filesystem view.
- **UTS namespace** → Lets containers have custom hostnames.
- **Network namespace** → Allows per-container network stacks.
- **IPC namespace** → Separates shared memory and message queues.
- **User namespace** → (Optional for MVP) Provides UID/GID mapping for security without requiring root.

✅ **Decision for MVP:**  
Start with PID, Mount, UTS, and Network namespaces. Add User namespace later for unprivileged containers.

---

### 3. Cgroups Resource Management
Cgroups (Control Groups) allow resource limitation per container.

- Implement a wrapper around `/sys/fs/cgroup`.
- Focus on `cpu`, `memory`, and `pids` subsystems first.
- Future: add `blkio` and `devices`.

✅ **Decision for MVP:**  
Add simple CLI flags for CPU and memory limits (e.g., `--cpu=50%`, `--mem=512M`).

---

### 4. Filesystem Architecture
Containers will require a layered filesystem setup:

- **Base RootFS**: Each supported distro (Arch, Alpine, Ubuntu) has a base image.
- **OverlayFS Layers**: Combine base + writable layers.
- **Container Volumes** (optional): Bind-mount host directories into containers.

✅ **Decision for MVP:**  
Use OverlayFS for layering. Mount structure example:
```
container/
  upperdir/
  workdir/
  merged/  ← active mountpoint
```

---

### 5. Networking System
Networking will be implemented with Linux network namespaces and virtual Ethernet pairs (`veth`).

- Each container gets a veth interface connected to a host bridge (like Docker’s `docker0`).
- The bridge provides internet access via NAT (using iptables/`nftables`).

✅ **Decision for MVP:**  
Implement single-bridge NAT setup with optional isolated networks in later versions.

---

### 6. Process Execution
The container’s init process is launched using `clone()` or `unshare()` syscalls with namespace flags. The CLI command will wrap these operations.

Example flow:
1. Prepare rootfs (mount overlay).
2. Create namespaces.
3. Apply cgroup limits.
4. Launch process (`/bin/bash` or user command).
5. Attach to container TTY.

---

### 7. Security & Permissions
For simplicity, the MVP will run as root, but will plan for user namespace support in later versions.

Future additions:
- Drop capabilities (`capsh --drop`)
- Add AppArmor/SELinux profiles

---

### 8. CLI & API Design Overview
Commands (examples):
```
myct run --name test --image arch /bin/bash
myct ps
myct stop test
myct rm test
```

- CLI interacts directly with the container manager.
- Future: add a REST API or daemon.

---

### 9. Base OS Integration (Arch, Alpine, Ubuntu)
Each supported base image provides:
- Minimal rootfs archive (tarball or squashfs).
- Image metadata (name, version, layer info).

✅ **Decision for MVP:**  
Ship with local tarball images:
- `arch-base.tar.gz`
- `alpine-base.tar.gz`
- `ubuntu-base.tar.gz`

---

### 10. System Architecture Summary (High-Level)
```
┌──────────────────────────────┐
│         CLI Interface        │
│  (myct run/start/ps/rm)      │
└──────────────┬───────────────┘
               │
┌──────────────▼───────────────┐
│     Container Manager        │
│  - Lifecycle handling        │
│  - Metadata registry         │
└──────────────┬───────────────┘
               │
   ┌───────────┼─────────────┐
   │           │             │
┌──▼──┐    ┌───▼──┐     ┌────▼────┐
│FS   │    │NS/Cg │     │Network │
│Layer│    │Layer │     │Module  │
└─────┘    └──────┘     └────────┘
```

---

Next step → **Step 3: Planning & MVP Roadmap**
Will define milestones, MVP goals, and implementation roadmap with time estimates and dependencies.

