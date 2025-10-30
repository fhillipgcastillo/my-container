# Comprehensive Container Runtime Implementation Guide

## Table of Contents
1. [Project Overview](#project-overview)
2. [Core Design & Architecture](#core-design--architecture)
3. [Implementation Roadmap](#implementation-roadmap)
4. [Technical Implementation Details](#technical-implementation-details)
5. [Development Guidelines](#development-guidelines)
6. [Testing & Security](#testing--security)
7. [Future Enhancements](#future-enhancements)

## Project Overview

### Goal
Create a lightweight container runtime (`myctr`) that allows isolated environments to share the same OS kernel, using unprivileged containers by default with optional privileged execution. Built for learning, experimentation, and extensibility.

### Core Characteristics
- **Language**: Go (primary) with minimal dependencies
- **Target Distros**: Ubuntu 22.04+/Debian 12+, Arch Linux
- **Runtime Model**: CLI-based tool (`myctr`) controlling isolated processes
- **Default Mode**: Unprivileged (rootless) via user namespaces
- **Optional Mode**: Privileged execution for advanced features

### Isolation Features
- **Namespaces**: user, pid, net, ipc, uts, mount
- **Cgroups v2**: CPU, memory, I/O control
- **Storage**: OverlayFS + tmpfs for rootfs layering
- **Network**: slirp4netns (unprivileged), virtual bridge (privileged)
- **Security**: Optional seccomp/AppArmor enhancements

### MVP Commands
- `myctr run`: Spawn process in new namespaces with mounted rootfs
- `myctr build`: Parse Containerfile and create layered images
- `myctr list/ps`: Show running containers
- `myctr stop/rm`: Manage container lifecycle
- `myctr logs`: View container output

## Core Design & Architecture

### Privilege Model
- **Unprivileged (Default)**:
  - Uses user namespaces
  - Root inside container ≠ root on host
  - UID/GID mapping via `/etc/subuid` and `/etc/subgid`
  - slirp4netns for networking
  - Possible OverlayFS limitations

- **Privileged (Optional)**:
  - Full control over namespaces
  - Direct network setup (veth + bridge)
  - Unrestricted OverlayFS usage
  - Host resource access

### Storage Architecture
- **Image Format**: Custom tar + metadata.json (MVP)
- **Layout**:
  ```
  /var/lib/myctr/
  ├── containers/
  │   └── <id>/
  │       ├── config.json
  │       ├── rootfs/
  │       └── state.json
  ├── images/
  │   └── <name>/
  │       ├── manifest.json
  │       └── layers/
  └── logs/
  ```

### System Architecture
```
┌──────────────────────────────┐
│         CLI Interface        │
│  (myctr run/start/ps/rm)    │
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
┌──▼──┐    ┌───▼──┐     ┌───▼───┐
│FS   │    │NS/Cg │     │Network│
│Layer│    │Layer │     │Module │
└─────┘    └──────┘     └───────┘
```

## Implementation Roadmap

### Phase 0: Environment Setup (1 week)
- Validate kernel features (userns, cgroup v2)
- Test unprivileged container creation
- Verify OverlayFS functionality
- Set up development environment

### Phase 1: Core Runtime (2-3 weeks)
- Implement namespace creation
- Configure UID/GID mapping
- Mount minimal root filesystem
- Execute processes in containers

### Phase 2: Image & Build (3-4 weeks)
- Implement Containerfile parser
- Create layered filesystem support
- Add image build pipeline
- Handle container state management

### Phase 3: Networking (3 weeks)
- Integrate slirp4netns for unprivileged
- Implement bridge networking for privileged
- Add port forwarding support
- Configure DNS resolution

### Phase 4: Resource Control (3 weeks)
- Add cgroup v2 resource limits
- Implement logging system
- Add container lifecycle commands
- Create cleanup utilities

### Phase 5: Polish & Security (2-3 weeks)
- Improve error handling
- Add seccomp profiles
- Write documentation
- Create integration tests

## Technical Implementation Details

### Project Structure
```
myctr/
├── cmd/
│   └── myctr/               # CLI entrypoint
├── pkg/
│   ├── runtime/             # Core container runtime
│   ├── builder/             # Image builder
│   ├── image/              # Image management
│   ├── net/                # Network setup
│   └── utils/              # Shared utilities
├── internal/
│   ├── syscalls/           # Syscall wrappers
│   └── config/             # Configuration
└── tests/
    ├── integration/
    └── unit/
```

### Containerfile Specification
```dockerfile
FROM arch:latest
COPY ./app /app
RUN pacman -S --noconfirm python
ENV PORT=8080
CMD ["/app/start.sh"]
```

### Container Runtime Flow
1. **Setup**:
   - Create container directories
   - Prepare rootfs (overlay/bind mount)
   - Configure networking

2. **Namespace Creation**:
   ```go
   cmd := exec.Command("/proc/self/exe", "--child", id)
   cmd.SysProcAttr = &syscall.SysProcAttr{
       Cloneflags: syscall.CLONE_NEWUSER | 
                  syscall.CLONE_NEWNS | 
                  syscall.CLONE_NEWPID |
                  syscall.CLONE_NEWUTS |
                  syscall.CLONE_NEWNET,
   }
   ```

3. **Resource Configuration**:
   - Write UID/GID mappings
   - Configure cgroups
   - Setup networking

4. **Process Execution**:
   - Mount proc, sys, dev
   - Pivot root/chroot
   - Execute container command

### Network Implementation
- **Unprivileged Mode**:
  - Use slirp4netns for user-space networking
  - Configure DNS via host resolv.conf
  - Handle port forwarding

- **Privileged Mode**:
  - Create veth pair
  - Configure bridge interface
  - Setup NAT rules

## Development Guidelines

### Go-Specific Considerations
- Use Go 1.22+ for latest features
- Handle Go runtime threading issues
- Implement proper error handling
- Follow standard Go project layout

### Best Practices
- Use meaningful error messages
- Implement proper cleanup handlers
- Follow security best practices
- Write comprehensive tests

### Dependencies
- **Required**:
  - golang.org/x/sys/unix
  - github.com/spf13/cobra
  - slirp4netns (external)

- **Optional**:
  - github.com/containerd/cgroups
  - github.com/opencontainers/runc/libcontainer

## Testing & Security

### Test Coverage
- Unit tests for core components
- Integration tests for container lifecycle
- Network connectivity tests
- Resource limit verification

### Security Measures
- Proper capability dropping
- Secure mount options
- Resource isolation verification
- User namespace validation

## Future Enhancements

### Post-MVP Features
- OCI image format support
- Advanced networking features
- Improved image layer caching
- REST API and daemon mode
- Additional security profiles

### Optimization Areas
- Resource usage efficiency
- Startup time improvement
- Image size reduction
- Cache mechanism enhancement

---

This guide serves as the primary reference for implementing the container runtime. It should be updated as the project evolves and new features are added.