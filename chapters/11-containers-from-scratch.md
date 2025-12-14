# Chapter 11: Containers from Scratch

> **First Principles Question**: What is a container, really? Not "lightweight VM," not "Docker image," but fundamentally—what Linux primitives make containers possible, and why do they work the way they do?

---

## Chapter Overview

Containers have revolutionized how we deploy software. But "container" has become a marketing term that obscures what's actually happening. Understanding containers from first principles reveals they're not magic—they're a clever combination of Linux kernel features that have existed for years.

This chapter builds a mental model of containers from the ground up, so you understand not just how to use them but why they work.

**What readers will understand after this chapter:**
- The fundamental difference between containers and VMs
- Linux namespaces and what they isolate
- cgroups and resource limits
- Union filesystems and layered images
- How to "build a container" from Linux primitives
- Security boundaries and their limitations

---

## Section 1: The Problem Containers Solve

### 1.1 The Deployment Nightmare

**The Scenario:**
```
Developer's machine:  "It works on my machine!"
- Ubuntu 22.04
- Python 3.11
- OpenSSL 3.0
- Custom compiled libfoo

Production server:    "It's broken!"
- Ubuntu 20.04
- Python 3.8
- OpenSSL 1.1
- No libfoo
```

**Traditional Solutions:**

**Solution 1: Detailed Documentation**
```
DEPLOYMENT_GUIDE.md:
1. Install Python 3.11 from source
2. Compile OpenSSL 3.0
3. Download libfoo from...
4. Set LD_LIBRARY_PATH to...
5. Hope nothing conflicts with other apps on the server
```
*Result: 50-page runbooks, human error, "works on my machine" anyway*

**Solution 2: Virtual Machines**
```
┌─────────────────────────────────────┐
│             Guest OS                │
│  ┌───────────────────────────────┐  │
│  │        Your Application       │  │
│  │        + All Dependencies     │  │
│  └───────────────────────────────┘  │
│                                     │
│  Full OS: kernel, systemd, SSH,    │
│  networking stack, package manager  │
└─────────────────────────────────────┘
                    │
                    │ Hypervisor
                    │
┌─────────────────────────────────────┐
│           Host Machine              │
└─────────────────────────────────────┘
```
*Result: Works, but heavy—multi-GB images, minutes to start, resource overhead*

### 1.2 What If...

**The Key Insight:**
We don't need a full OS. We need isolation and a consistent filesystem. The application just needs to think it has its own environment.

**The Container Approach:**
```
┌─────────────────────────────────────┐
│        Your Application             │
│        + Dependencies               │
│        (just files, not full OS)    │
└─────────────────────────────────────┘
                    │
           Shared Linux Kernel
                    │
┌─────────────────────────────────────┐
│           Host Machine              │
└─────────────────────────────────────┘
```

**No guest OS. Same kernel. Just isolation.**

### 1.3 Containers vs. Virtual Machines

| Aspect | VM | Container |
|--------|-----|-----------|
| Isolation | Hardware-level | Process-level |
| OS | Full guest OS | Shared host kernel |
| Size | GBs | MBs |
| Startup | Minutes | Seconds/milliseconds |
| Resource overhead | High | Minimal |
| Security boundary | Strong (hardware) | Weaker (kernel) |

**When VMs Are Still Better:**
- Need different OS (Windows on Linux host)
- Untrusted workloads (stronger isolation)
- Kernel-level differences required

**When Containers Excel:**
- Same OS workloads
- Fast scaling
- Efficient resource usage
- Development/test environments

---

## Section 2: Linux Namespaces — The Isolation Foundation

### 2.1 What Are Namespaces?

**Definition:**
Namespaces partition kernel resources so that one set of processes sees one set of resources, while another set sees a different set.

**Analogy:**
Imagine a hotel where each room has "Room 1" on the door, but they're all different rooms. Each guest thinks they're in Room 1, but they're isolated from each other.

**The Namespaces:**
| Namespace | Isolates | Flag |
|-----------|----------|------|
| PID | Process IDs | CLONE_NEWPID |
| Network | Network stack | CLONE_NEWNET |
| Mount | Filesystem mounts | CLONE_NEWNS |
| UTS | Hostname | CLONE_NEWUTS |
| IPC | Inter-process communication | CLONE_NEWIPC |
| User | User/group IDs | CLONE_NEWUSER |
| Cgroup | Cgroup root directory | CLONE_NEWCGROUP |

### 2.2 PID Namespace

**What It Does:**
Processes in a PID namespace have their own process ID numbering.

**Without PID Namespace:**
```
Host sees:
PID 1: init
PID 2: kernel thread
...
PID 1000: your-app
PID 1001: your-app-child
```

**With PID Namespace:**
```
Host sees:               Container sees:
PID 1: init              PID 1: your-app
...                      PID 2: your-app-child
PID 1000: your-app       (thinks it's alone)
PID 1001: your-app-child
```

**Why It Matters:**
- Process 1 in a container can be killed without affecting real init
- Container can't see or signal host processes
- `ps aux` in container shows only container processes

**Demonstration:**
```bash
# Create new PID namespace
sudo unshare --pid --fork --mount-proc /bin/bash

# Inside the new namespace
ps aux
# Only shows bash and ps (PID 1 and 2!)

echo $$
# Shows 1 (bash is PID 1 in this namespace)
```

### 2.3 Network Namespace

**What It Does:**
Each namespace has its own network interfaces, routing tables, firewall rules.

**Without Network Namespace:**
```
All processes share:
- eth0 (physical interface)
- 192.168.1.100 (IP)
- routing table
- iptables rules
```

**With Network Namespace:**
```
Container namespace:     Host namespace:
- veth0                  - eth0
- 172.17.0.2             - 192.168.1.100
- own routing table      - own routing table
- own iptables           - own iptables
```

**Demonstration:**
```bash
# Create network namespace
sudo ip netns add mycontainer

# Create virtual ethernet pair
sudo ip link add veth0 type veth peer name veth1

# Move one end to the namespace
sudo ip link set veth1 netns mycontainer

# Configure the namespace
sudo ip netns exec mycontainer ip addr add 192.168.1.2/24 dev veth1
sudo ip netns exec mycontainer ip link set veth1 up
sudo ip netns exec mycontainer ip link set lo up

# Run command in namespace
sudo ip netns exec mycontainer ping 192.168.1.1
```

### 2.4 Mount Namespace

**What It Does:**
Each namespace has its own view of the filesystem mount points.

**Without Mount Namespace:**
```
All processes see same mounts:
/        → disk1
/home    → disk2
/tmp     → tmpfs
```

**With Mount Namespace:**
```
Container's view:        Host's view:
/    → container image   /    → disk1
                         /home → disk2
                         /tmp  → tmpfs
```

**Why This Is Key:**
Containers see a completely different root filesystem—their image becomes `/`.

**Demonstration:**
```bash
# Create new mount namespace
sudo unshare --mount /bin/bash

# Mount a different root (in the new namespace only)
mkdir /tmp/newroot
# ... populate newroot with minimal filesystem
mount --bind /tmp/newroot /tmp/newroot
cd /tmp/newroot
pivot_root . old_root
cd /

# Now '/' is the new root, host unaffected
```

### 2.5 UTS Namespace

**What It Does:**
Each namespace has its own hostname and domain name.

```bash
# Create new UTS namespace
sudo unshare --uts /bin/bash

# Change hostname (only in this namespace)
hostname mycontainer

# Verify
hostname
# Shows: mycontainer

# On host (different terminal)
hostname
# Shows: original-hostname
```

**Why It Matters:**
Containers can have their own identity, useful for service discovery and logging.

### 2.6 User Namespace

**What It Does:**
Maps user/group IDs between namespace and host.

**The Magic:**
```
Inside container:    Maps to host:
root (uid 0)    →    user (uid 1000)
user (uid 1000) →    user (uid 2000)
```

**Why It's Powerful:**
- Container can run as "root" without being root on host
- Enables rootless containers
- Limits damage from container escape

```bash
# Create user namespace as non-root
unshare --user --map-root-user /bin/bash

# Check identity
id
# uid=0(root) gid=0(root) (inside namespace)

# Try to access host root files
cat /etc/shadow
# Permission denied (not really root on host)
```

---

## Section 3: cgroups — Resource Limits

### 3.1 What Are cgroups?

**Definition:**
Control groups (cgroups) limit, account for, and isolate resource usage (CPU, memory, disk I/O, network) of a collection of processes.

**The Problem cgroups Solve:**
```
Without cgroups:
Container A: uses 100% CPU
Container B: starved
Container C: starved

With cgroups:
Container A: limited to 50% CPU
Container B: gets its share
Container C: gets its share
```

### 3.2 cgroup Controllers

| Controller | Controls |
|------------|----------|
| cpu | CPU time allocation |
| cpuset | Which CPUs can be used |
| memory | Memory limits |
| blkio | Block device I/O |
| devices | Device access |
| freezer | Suspend/resume processes |
| pids | Number of processes |

### 3.3 Memory Limits

**Setting Memory Limit:**
```bash
# Create cgroup
mkdir /sys/fs/cgroup/memory/mycontainer

# Set limit (100MB)
echo 104857600 > /sys/fs/cgroup/memory/mycontainer/memory.limit_in_bytes

# Add process to cgroup
echo $PID > /sys/fs/cgroup/memory/mycontainer/cgroup.procs
```

**What Happens at Limit:**
- Process allocates beyond limit
- Kernel triggers OOM (Out of Memory) killer
- Process is killed

**Soft vs Hard Limits:**
```bash
# Hard limit (OOM kill if exceeded)
echo 100M > memory.limit_in_bytes

# Soft limit (try to reclaim, but allow temporarily)
echo 80M > memory.soft_limit_in_bytes
```

### 3.4 CPU Limits

**CPU Shares (Relative):**
```bash
# Container A: 1024 shares
echo 1024 > /sys/fs/cgroup/cpu/containerA/cpu.shares

# Container B: 512 shares
echo 512 > /sys/fs/cgroup/cpu/containerB/cpu.shares

# Result: A gets 2x CPU time of B when both are busy
# If only B is running, it gets 100%
```

**CPU Quota (Absolute):**
```bash
# Period: 100ms
echo 100000 > /sys/fs/cgroup/cpu/mycontainer/cpu.cfs_period_us

# Quota: 50ms out of every 100ms (50% of one CPU)
echo 50000 > /sys/fs/cgroup/cpu/mycontainer/cpu.cfs_quota_us
```

### 3.5 cgroups v2

**Modern Unified Hierarchy:**
```
cgroups v1:               cgroups v2:
/sys/fs/cgroup/           /sys/fs/cgroup/
├── cpu/                  └── mycontainer/
│   └── container1/           ├── cgroup.controllers
├── memory/                   ├── cpu.max
│   └── container1/           ├── memory.max
└── pids/                     └── pids.max
    └── container1/

Multiple hierarchies      Single unified hierarchy
```

**cgroups v2 Example:**
```bash
# Create cgroup
mkdir /sys/fs/cgroup/mycontainer

# Set limits (unified interface)
echo "50000 100000" > /sys/fs/cgroup/mycontainer/cpu.max  # 50% CPU
echo 100M > /sys/fs/cgroup/mycontainer/memory.max        # 100MB RAM
echo 100 > /sys/fs/cgroup/mycontainer/pids.max           # 100 processes
```

---

## Section 4: Union Filesystems — Layered Images

### 4.1 The Problem

**Traditional Approach:**
```
App 1 needs: Ubuntu + Python + Flask + App1 code = 1GB
App 2 needs: Ubuntu + Python + Django + App2 code = 1.1GB
App 3 needs: Ubuntu + Python + FastAPI + App3 code = 1GB

Total: 3.1GB (but Ubuntu + Python repeated 3x!)
```

**What We Want:**
Share common layers, store only differences.

### 4.2 Union Filesystems

**The Concept:**
Stack multiple directories (layers) and present them as one unified filesystem. Changes go to a writable top layer.

```
        ┌─────────────────────────────┐
        │    Writable Layer           │  ← Container writes here
        │    (thin, container-specific)│
        ├─────────────────────────────┤
        │    App Code Layer (RO)      │  ← From image
        ├─────────────────────────────┤
        │    Flask Layer (RO)         │  ← Shared!
        ├─────────────────────────────┤
        │    Python Layer (RO)        │  ← Shared!
        ├─────────────────────────────┤
        │    Ubuntu Layer (RO)        │  ← Shared!
        └─────────────────────────────┘

Multiple containers share read-only layers
Each gets own writable layer
```

### 4.3 Copy-on-Write

**How Changes Work:**
```
Read file: Look up through layers, return first match
Write file:
  1. Copy file to writable layer
  2. Modify the copy
  3. Lower layers unchanged

Delete file:
  1. Create "whiteout" marker in writable layer
  2. File appears deleted but still exists in lower layers
```

**Example:**
```
Original (read-only layer):
/etc/config.txt = "original content"

Container modifies:
1. Copy /etc/config.txt to writable layer
2. Writable layer: /etc/config.txt = "modified content"
3. Container sees: "modified content"
4. Original layer: still "original content"

Another container from same image sees: "original content"
```

### 4.4 OverlayFS

**Modern Union Filesystem:**
```
OverlayFS structure:
┌─────────────────────────────────────────────┐
│  merged/                                    │ ← What container sees
│  (unified view)                             │
├─────────────────────────────────────────────┤
│  upper/                                     │ ← Writable layer
│  (container changes)                        │
├─────────────────────────────────────────────┤
│  lower/                                     │ ← Read-only (image layers)
│  (can be multiple stacked)                  │
├─────────────────────────────────────────────┤
│  work/                                      │ ← Internal bookkeeping
└─────────────────────────────────────────────┘
```

**Demonstration:**
```bash
# Create directories
mkdir lower upper work merged

# Populate lower (read-only base)
echo "base file" > lower/file1.txt
echo "to be modified" > lower/file2.txt

# Mount overlay
mount -t overlay overlay \
  -o lowerdir=lower,upperdir=upper,workdir=work \
  merged/

# Read from merged
cat merged/file1.txt  # "base file"

# Modify in merged
echo "modified" > merged/file2.txt

# Check layers
cat lower/file2.txt   # "to be modified" (unchanged!)
cat upper/file2.txt   # "modified" (copy-on-write)
cat merged/file2.txt  # "modified" (sees upper)
```

---

## Section 5: Building a Container from Scratch

### 5.1 The Ingredients

To create a container, we need:
1. **Namespaces**: Isolation from host
2. **cgroups**: Resource limits
3. **Root filesystem**: What the container sees
4. **Proper startup**: PID 1 handling

### 5.2 A Minimal Container Runtime

```c
// simplified_container.c
#define _GNU_SOURCE
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/mount.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <sys/wait.h>

#define STACK_SIZE (1024 * 1024)

char container_stack[STACK_SIZE];

char* container_args[] = {"/bin/bash", NULL};

int container_main(void* arg) {
    char* rootfs = (char*)arg;

    printf("Container PID: %d\n", getpid());

    // Set hostname
    sethostname("container", 9);

    // Setup new root filesystem
    chroot(rootfs);
    chdir("/");

    // Mount proc filesystem
    mount("proc", "/proc", "proc", 0, NULL);

    // Execute shell
    execv(container_args[0], container_args);

    return 1;
}

int main(int argc, char** argv) {
    if (argc < 2) {
        printf("Usage: %s <rootfs>\n", argv[0]);
        return 1;
    }

    printf("Parent PID: %d\n", getpid());

    // Create new namespaces and fork
    int container_pid = clone(
        container_main,
        container_stack + STACK_SIZE,
        CLONE_NEWPID |   // New PID namespace
        CLONE_NEWNS |    // New mount namespace
        CLONE_NEWUTS |   // New UTS namespace
        CLONE_NEWIPC |   // New IPC namespace
        CLONE_NEWNET |   // New network namespace
        SIGCHLD,
        argv[1]
    );

    if (container_pid == -1) {
        perror("clone");
        return 1;
    }

    // Wait for container to exit
    waitpid(container_pid, NULL, 0);

    return 0;
}
```

### 5.3 Preparing a Root Filesystem

```bash
# Create minimal rootfs using debootstrap
sudo debootstrap --variant=minbase focal /container/rootfs

# Or extract from existing image
docker export $(docker create alpine) | tar -xf - -C /container/rootfs

# Verify structure
ls /container/rootfs
# bin  dev  etc  home  lib  proc  root  sys  tmp  usr  var
```

### 5.4 Running Our Container

```bash
# Compile
gcc -o container container.c

# Run (requires root)
sudo ./container /container/rootfs

# Inside container:
hostname        # container
ps aux          # Only shows container processes
ls /            # Shows rootfs contents
```

### 5.5 Adding Resource Limits

```bash
# Create cgroup for our container
mkdir /sys/fs/cgroup/memory/mycontainer
mkdir /sys/fs/cgroup/cpu/mycontainer
mkdir /sys/fs/cgroup/pids/mycontainer

# Set limits
echo 50M > /sys/fs/cgroup/memory/mycontainer/memory.limit_in_bytes
echo 50000 > /sys/fs/cgroup/cpu/mycontainer/cpu.cfs_quota_us
echo 100000 > /sys/fs/cgroup/cpu/mycontainer/cpu.cfs_period_us
echo 50 > /sys/fs/cgroup/pids/mycontainer/pids.max

# Add container process to cgroups (in the C code or manually)
echo $CONTAINER_PID > /sys/fs/cgroup/memory/mycontainer/cgroup.procs
echo $CONTAINER_PID > /sys/fs/cgroup/cpu/mycontainer/cgroup.procs
echo $CONTAINER_PID > /sys/fs/cgroup/pids/mycontainer/cgroup.procs
```

---

## Section 6: Security Considerations

### 6.1 What Containers DON'T Isolate

**Shared Kernel:**
```
Host kernel is shared. Container can:
- Exploit kernel vulnerabilities
- Access kernel interfaces (/proc, /sys)
- Potentially escape namespace isolation
```

**Contrast with VMs:**
```
VM has own kernel:
- Host kernel protected by hypervisor
- VM kernel compromise doesn't affect host
- Hardware-level isolation
```

### 6.2 Container Escape Vectors

**Privileged Containers:**
```bash
# DANGEROUS: Full host access
docker run --privileged ...

# Can access host devices, mount host filesystems
# Almost no isolation
```

**Sensitive Mounts:**
```bash
# DANGEROUS: Mounting Docker socket
docker run -v /var/run/docker.sock:/var/run/docker.sock ...

# Container can control host's Docker
# Can create privileged containers
# Full host compromise
```

**Capability Abuse:**
```bash
# Capabilities give specific root powers
# Some are dangerous: CAP_SYS_ADMIN, CAP_NET_ADMIN
docker run --cap-add SYS_ADMIN ...
```

### 6.3 Hardening Containers

**Drop Capabilities:**
```bash
# Run with minimal capabilities
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE ...
```

**Read-Only Root:**
```bash
# Prevent filesystem modifications
docker run --read-only ...
```

**No New Privileges:**
```bash
# Prevent privilege escalation
docker run --security-opt no-new-privileges ...
```

**Seccomp Profiles:**
```bash
# Limit system calls
docker run --security-opt seccomp=profile.json ...
```

**AppArmor/SELinux:**
```bash
# Mandatory access control
docker run --security-opt apparmor=docker-default ...
```

### 6.4 Rootless Containers

**Run Without Root:**
```bash
# User namespace maps container root to unprivileged user
# Even if container is compromised, no host root access

# Podman supports rootless by default
podman run ...

# Docker requires configuration for rootless mode
dockerd-rootless.sh
```

**Limitations:**
- Can't bind to ports below 1024
- Some networking features limited
- Requires user namespace support

---

## Section 7: Container Standards

### 7.1 OCI (Open Container Initiative)

**Standardization:**
- **Image Spec**: How images are structured
- **Runtime Spec**: How containers are created and run
- **Distribution Spec**: How images are distributed

**Why It Matters:**
Docker images work with Podman, containerd, CRI-O, etc.

### 7.2 Container Runtimes

**High-Level:**
- Docker Engine
- containerd
- CRI-O

**Low-Level:**
- runc (OCI reference implementation)
- crun (faster, C implementation)
- gVisor (sandbox)
- Kata Containers (VM-based)

**The Stack:**
```
┌─────────────────────────────────────────────────────┐
│  docker CLI / kubectl                               │
├─────────────────────────────────────────────────────┤
│  Docker Engine / containerd / CRI-O                │
├─────────────────────────────────────────────────────┤
│  runc / crun / gVisor / kata                       │
├─────────────────────────────────────────────────────┤
│  Linux Kernel (namespaces, cgroups, etc.)          │
└─────────────────────────────────────────────────────┘
```

---

## Section 8: Now You Understand Why...

Armed with this foundation, you can now understand:

- **Why containers start so fast**: No OS to boot, just process isolation setup

- **Why images are layered**: Union filesystems enable sharing and efficiency

- **Why containers share the host kernel**: Namespaces provide isolation without virtualization

- **Why "container escape" is possible**: Shared kernel means kernel vulnerabilities affect containers

- **Why resources need limits**: Without cgroups, one container could starve others

- **Why rootless containers exist**: User namespaces allow unprivileged container operation

- **Why Docker isn't the only option**: OCI standards mean containers are portable

---

## Practical Exercises

### Exercise 1: Explore Namespaces
```bash
# See your process's namespaces
ls -la /proc/$$/ns/

# Create a new network namespace
sudo ip netns add test
sudo ip netns exec test ip addr
```

### Exercise 2: Create a cgroup
```bash
# Create memory-limited cgroup
sudo mkdir /sys/fs/cgroup/memory/test
echo 50M | sudo tee /sys/fs/cgroup/memory/test/memory.limit_in_bytes

# Run a process in it
echo $$ | sudo tee /sys/fs/cgroup/memory/test/cgroup.procs

# Try to allocate more than 50MB
```

### Exercise 3: Build a Container Image Manually
```bash
# Create minimal rootfs
mkdir rootfs
# Add busybox binary
cp /bin/busybox rootfs/bin/
# Create symlinks for commands

# Package as tarball (poor man's image)
tar -cvf myimage.tar -C rootfs .
```

### Exercise 4: Overlay Filesystem
Follow the OverlayFS demonstration in Section 4.4. Observe copy-on-write behavior.

### Exercise 5: Container Security
```bash
# Compare capabilities
docker run --rm alpine cat /proc/1/status | grep Cap
docker run --rm --cap-drop ALL alpine cat /proc/1/status | grep Cap
```

---

## Key Takeaways

1. **Containers are Linux primitives, not magic**. Namespaces for isolation, cgroups for resources, union filesystems for images.

2. **Containers share the host kernel**. This is their power (efficiency) and weakness (security boundary).

3. **Union filesystems enable efficient images**. Layers are shared, writes go to a thin writable layer.

4. **cgroups prevent resource abuse**. Without limits, containers could starve each other.

5. **Security requires conscious hardening**. Default containers are not secure; drop capabilities, use read-only roots, consider rootless.

6. **OCI standards mean portability**. Docker images work everywhere because of standardization.

---

## Looking Ahead

Now that you understand what containers actually are, Chapter 12 explores Docker—the tool that made containers accessible—and how to use it effectively in practice.

---

## Chapter 11 Summary Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONTAINERS FROM SCRATCH                                   │
│                                                                             │
│  WHAT IS A CONTAINER?                                                       │
│  ────────────────────                                                       │
│  NOT a VM. A process with:                                                  │
│  • Isolated view of system (namespaces)                                    │
│  • Resource limits (cgroups)                                               │
│  • Different root filesystem (union fs)                                    │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  NAMESPACES (Isolation)                                                     │
│  ──────────────────────                                                     │
│  PID:     Process IDs       (container sees own PIDs)                      │
│  NET:     Network stack     (own interfaces, IPs, routes)                  │
│  MNT:     Mount points      (own filesystem view)                          │
│  UTS:     Hostname          (own hostname)                                 │
│  IPC:     IPC resources     (own semaphores, shared memory)                │
│  USER:    User/Group IDs    (uid mapping, rootless)                        │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  CGROUPS (Resources)                                                        │
│  ───────────────────                                                        │
│  Memory:  memory.max = 100M                                                │
│  CPU:     cpu.max = "50000 100000"  (50% of one CPU)                      │
│  PIDs:    pids.max = 100                                                   │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  UNION FILESYSTEM (Images)                                                  │
│  ─────────────────────────                                                  │
│                                                                             │
│  ┌─────────────────┐                                                       │
│  │ Writable Layer  │ ← Container changes (copy-on-write)                   │
│  ├─────────────────┤                                                       │
│  │ App Layer (RO)  │ ← From image                                          │
│  ├─────────────────┤                                                       │
│  │ Base Layer (RO) │ ← Shared across containers                            │
│  └─────────────────┘                                                       │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  SECURITY CONSIDERATIONS                                                    │
│  ───────────────────────                                                    │
│                                                                             │
│  Shared kernel = Weaker boundary than VMs                                  │
│  Harden with:                                                              │
│  • Drop capabilities (--cap-drop ALL)                                      │
│  • Read-only root (--read-only)                                            │
│  • Seccomp profiles                                                        │
│  • Rootless mode                                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*Next Chapter: Docker in Practice — Using the tool that made containers mainstream*
