---
tags:
  - cks
  - kubernetes
---
# Least Privilege Principle
- Least privilege means: give every user, service account, process, node, and workload only the permissions it needs to do its job nothing extra. 
- In Kubernetes, this mainly reduces the risk of privilege escalation, node compromise, lateral movement, and unnecessary attack surface. 

**Least Privilege  Implementation**

| Security Measure                 | Description                                                                          |
| -------------------------------- | ------------------------------------------------------------------------------------ |
| Limit Node Access                | Ensure nodes have restricted user permissions to prevent unauthorized modifications. |
| Role-Based Access Control (RBAC) | Define precise access rights for users and services within the cluster.              |
| Remove Obsolete Packages         | Keep systems updated by removing software that is no longer required.                |
| Restrict Network Access          | Limit network communication between components to reduce attack surfaces.            |
| Restrict Kernel Modules          | Load only essential kernel modules and block unnecessary ones.                       |
| Fix Open Ports                   | Identify and secure any open ports to prevent unauthorized entry points.             |

---

## 1. Limiting Node Access

```sh
# Shows the current user ID, group ID, and groups.
id
# Example: uid=1000(omar) gid=1000(omar) groups=1000(omar),27(sudo)

# Shows users currently logged in to the system.
who

# Shows login history: who logged in, from where, and when.
last

# Searches for a user named omar in /etc/passwd , -i means case-insensitive , ^omar means the line must start with omar
grep -i ^omar /etc/passwd

# /etc/shadow contains password hashes
grep -i ^omar /etc/shadow

# Searches for groups named omar
grep -i ^omar /etc/group

# Changes omar’s login shell to: /bin/nologin
usermod -s /bin/nologin omar

# To remove the user and home directory
userdel -r bob

# Removes user omar from the group admin.
deluser omar admin
```

---

## 2. SSH Hardening
SSH Commands
```sh
ssh <hostname OR IP Address>
ssh <user>@<hostname OR IP Address>
# or Without @
ssh -l <user> <hostname OR IP Address>
# -l: login name
```
### Using SSH Key Pairs for Authentication
A more secure authentication method involves using a cryptographic key-pair composed of a private key on the client and a public key installed on the remote server. With this setup, you can log in without repeatedly entering a password.

```sh
# Generate the key pair on your client
ssh-keygen -t rsa
```
**To enable passwordless login**, copy the public key to the remote server using the `ssh-copy-id` command. For example, if your username is `mark` and the remote server is `node01`, run:

```sh
ssh-copy-id mark@node01
```
After providing your password for the remote system, the public key is appended to the `authorized_keys` file inside the .ssh directory of your remote home folder. You can verify its content with:
```sh
cat /home/mark/.ssh/authorized_keys
```

### Hardening SSH Service
#### Disabling Root Login
Disabling remote logins for the root account is a best security practice
```sh
vi /etc/ssh/sshd_config
# Locate the line for PermitRootLogin and change it to:
# PermitRootLogin no
```

#### Disabling Password-Based Authentication
Since key-based authentication is now in place, you can disable password-based authentication to further protect your server
```sh
# /etc/ssh/sshd_config
PasswordAuthentication no
```
After making these changes, save the file and restart the SSH service:
```sh
systemctl restart sshd
```

---

## 3. User Privilege Escalation
- **Privilege Escalation** in Linux means a user or process gains higher permissions than it originally had `normal user → root user`
- Privilege Escalation: `sudo su -`

The default configuration for sudo is maintained in the `/etc/sudoers` file. This file governs policies for executing commands with elevated privileges and can only be modified by users who have been explicitly granted access. Only users listed in the `/etc/sudoers` file can use **sudo**, thereby preventing unauthorized root logins.

```sh
cat /etc/sudoers
```

```text
# The root user can run any command as any user and any group on all hosts.
root    ALL=(ALL:ALL) ALL

# Members of the admin group (%admin) may gain root privileges
# Any user inside the admin group can run any command as any user.
%admin ALL=(ALL) ALL

# Allow members of group sudo to execute any command
# Any user in the sudo group can run any command as any user and any group.
%sudo   ALL=(ALL:ALL) ALL

# Allow mark to run any command
# User mark can run any command as any user and any group.
mark    ALL=(ALL:ALL) ALL

# Sarah can run the reboot command as root on localhost.
sarah localhost=/usr/bin/shutdown -r now

# See sudoers(5) for more information on "#include" directives
#include /etc/sudoers.d
```

Each line in the **sudoers file** is structured as follows:
- **User or Group**: The first field specifies the user or group (groups are prefixed with %) that receives the privileges.
- **Host Specification**: The second field, typically set to ALL, indicates that the privileges apply to all hosts (commonly confined to the localhost).
- **Run-as Specification**: The third field, enclosed in parentheses, indicates the user(s) as whom the commands will be executed. "ALL" means that commands can be run as any user.
- **Command Specification**: The fourth field specifies the allowed commands. Using "ALL" permits any command, though you can restrict users to specific commands, as demonstrated in the entry for Sarah.

```text
USER_OR_GROUP  HOST=(RUN_AS_USER:RUN_AS_GROUP)  COMMANDS

Example:
omar ALL=(ALL:ALL) ALL
omar ALL => (Host)= (ALL => user:ALL => Group) ALL => (Command)

# Meaning:
omar can run any command,
on any host,
as any user,
as any group.
```

> [!important] 
> Avoid modifying the `/etc/sudoers` file improperly—always use the `visudo` command to edit and check syntax before saving

> [!note]

```sh
grep -i ^root /etc/passwd
# root:x:0:0:root:/root:/usr/sbin/nologin
```
- username: root
- password field: x
- UID: 0
- GID: 0
- comment: root
- home directory: /root
- login shell: /usr/sbin/nologin

```text
UID 0 = root/admin user
/usr/sbin/nologin = interactive login is disabled
```
So this means the `root` account exists, but direct login as `root` is disabled because its shell is set to `/usr/sbin/nologin`

Pass in the private key while connecting to a server via SSH:
```sh
ssh -i <private.key>
```

---

## 4. Removing Unwanted Packages and Services
- If you do not need it, remove it or disable it.
- To view all services installed on your system, use:

```sh
systemctl list-units --type service
```

```sh
# If you determine that a service file is not needed, you can disable and stop it. For example, to disable Apache:
systemctl stop apache2
systemctl disable apache2

# If you want to prevent a service from being started manually or by another service:
sudo systemctl mask <service-name>
# To unmask:
sudo systemctl unmask <service-name>

# After stopping the service, remove the corresponding package
apt remove apache2
```

Unwanted packages may:
1. Contain vulnerabilities
2. Open network ports
3. Run background services
4. Consume resources
5. Allow attackers more tools to abuse

---

## 5. Restricting Kernel Modules
The Linux kernel follows a modular design, making it easy to extend its capabilities dynamically. For example, when new hardware is connected, the kernel automatically or manually loads the necessary module using tools such as `modprobe` or `insmod` to enable device support.

```sh
# to list all modules loaded into the kernel
lsmod

# to load the PC Speaker module manually 
modprobe pcspkr

# vi /etc/modprobe.d/blacklist.conf
# You can use any file name ending with a `.conf` and it is located in the `/etc/modprobe.d/`.
# blacklist sctp

# Then reboot and test
lsmod | grep sctp
```
you should blacklist `sctp` and `dccp` on Kubernetes nodes if your workloads do not need them: 
```sh
# Create a blacklist file on each Kubernetes node:
vi /etc/modprobe.d/blacklist.conf

blacklist sctp
blacklist dccp
```
Check: 
```sh
lsmod | grep -E 'sctp|dccp'
```
Unload if they are loaded:
```sh
sudo modprobe -r sctp
sudo modprobe -r dccp
```
This is a common Linux/Kubernetes hardening step to reduce kernel attack surface. The SCTP module is seldom used in Kubernetes clusters. DCCP is generally not used.

---

## 6. Disabling Open Ports
Shows all network connections and listening ports:
```sh
netstat -an | grep -w LISTEN
# -a  → show all sockets/connections
# -n  → show numbers instead of resolving names
# -w the exact word LISTEN
```

```text
tcp        0      0 0.0.0.0:22       0.0.0.0:*       LISTEN
tcp        0      0 127.0.0.1:53     0.0.0.0:*       LISTEN
tcp6       0      0 :::80            :::*            LISTEN
```
- `0.0.0.0:22` Service is listening on port `22` on all `IPv4 interfaces`
- `127.0.0.1:53` Service is listening on port `53` only `locally`
- `:::80` Service is listening on port `80` on `IPv6/all interfaces`

```sh
cat /etc/services | grep -w 53
# /etc/services is a file that maps common port numbers to service names.
# or
grep -w 53 /etc/services
```

---

## 7. Minimizing IAM Policies and Roles
Bad example:
- EKS node role has AdministratorAccess

Better:
- Node role has only required permissions
- Pod-specific access uses [[#^irsa|^IRSA]] / workload identity
- CI/CD role can deploy only to required cluster/namespace

### IRSA ^irsa
- It is an AWS EKS feature that lets a Kubernetes ServiceAccount assume an AWS IAM Role
- So Pods can access AWS services without using static AWS access keys and **without using the node IAM role.**
- AWS describes IRSA as associating an IAM role with a Kubernetes service account so Pods using that service account can call AWS APIs with that role’s permissions
- **Pod → Kubernetes ServiceAccount → AWS IAM Role → AWS permissions**

---

## 8. Restricting Network Access
**UFW Firewall**: Uncomplicated Firewall, its is as front-end for IPTables.

```sh
netstat -an | grep -w LISTEN
```

```text
tcp        0      0 0.0.0.0:22          0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:80           0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:8080         0.0.0.0:*               LISTEN
```
Installing UFW:
```sh
apt-get update
# ... additional update output ...
apt-get install ufw

# After installation, check the current status of UFW:
ufw status
# The expected output should state:
# Status: inactive
```
Default Firewall Rules: 
```sh
ufw default allow outgoing
# Next, set the default rule to deny all inbound connections:
ufw default deny incoming
```
Permit `SSH`, `HTTP` from the jump server with the ip add `172.16.238.5`:
```sh
ufw allow from 172.16.238.5 to any port 22 proto tcp
ufw allow from 172.16.238.5 to any port 80 proto tcp
# from 172.16.238.5   = source ip
# to any              = Any IP address/interface on this machine
# port 22             = destination port
# proto tcp           = protocol

# allow from my local subnet to samba app
sudo ufw allow from 192.168.1.0/24 to any app Samba
```
Deny access to `8080` from everywhere:
```sh
ufw deny 8080
```
Enable UFW: 
```sh
ufw enable
ufw status
sudo ufw status numbered
ufw delete 5
```

---

## 9. Linux syscalls
- A normal application cannot directly access hardware, files, network, memory, processes, etc. It must ask the kernel through a syscall.
- Simple meaning: `Syscall = request from user-space program to Linux kernel`

```text
Application
   ↓
System call
   ↓
Linux kernel
   ↓
Hardware / filesystem / network / process management
```

- System calls enable applications running in user space to request services from the kernel. For instance, when an application needs to open a file stored on disk, it cannot access the hardware directly; instead, it must instruct the kernel to perform the necessary operations. Consider the task of creating an empty file named `error.log` in the `/tmp` directory. This process involves a series of system calls, beginning with the `execve` call to execute the binary (such as the `touch` command).
- The Linux kernel architecture involves many layers: user space, kernel space, system calls, and the interactions with memory, CPU, and devices. The image below reinforces this conceptual framework:

![syscall](https://kodekloud.com/kk-media/image/upload/v1752871739/notes-assets/images/Certified-Kubernetes-Security-Specialist-CKS-Linux-Syscalls/frame_80.jpg)

### Tracing syscalls
Tracing Syscalls with `strace` command
Example:
```sh
cat file.txt
```
Behind the scenes, cat uses syscalls like:
```text
open()   → open the file
read()   → read file content
write()  → print content to terminal
close()  → close the file
```
You can see syscalls using:
```sh
strace <command>
strace ls
strace cat file.txt
```
Show me the Linux system calls made by the etcd process:
```sh
pidof etcd
# output: 3596
strace -p 3596
```
Summary of all syscalls:
```sh
strace -c touch error.log
```

### AquaSec Tracee
- `tracee` is an open source tool created by Aqua Security used to trace syscalls on a container.
- it used eBPF to trace the system at runtime
- **eBPF** is a Linux technology that lets you run small, safe programs inside the kernel without modifying or interfering the kernel source code or loading traditional kernel modules.

Tracee as Docker Container:
```sh
# 1. Tracing Syscalls for a Single Command
# To capture system calls generated by a single command ls
docker run --name tracee --rm --privileged --pid=host \
  -v /lib/modules/:/lib/modules:ro \
  -v /usr/src:/usr/src:ro \
  -v /tmp/tracee:/tmp/tracee \
  aquasec/tracee:0.4.0 --trace comm=ls
  
# Kernel Header: /lib/modules and its dependencies and source files under /usr/src
# Output directory: /tmp/tracee
# --privileged --pid=host : gives Tracee enough access to observe host processes and kernel events. 
# --trace comm=ls : Trace only events where the process command name is ls.

# This command outputs a list of syscalls invoked by the ls command. A sample output might include:
# TIME(s)      UID    COMM    PID    TID    RET
# 1263.457188  0      ls      27461  27461  -2
# 1263.457218  0      ls      27461  27461  -2
# 1263.457238  0      ls      27461  27461  0
# ...
# [output truncated]

# 2. Tracing Syscalls for All New Processes
sudo docker run --name tracee --rm --privileged --pid=host \
  -v /lib/modules/:/lib/modules:ro \
  -v /usr/src:/usr/src:ro \
  -v /tmp/tracee:/tmp/tracee \
  aquasec/tracee:0.4.0 --trace pid=new

# This setup produces extensive output as Tracee collects syscall data for every new process initiated on the host. An excerpt from the output may resemble:
# 1613.769845 0  wc      1619 1619  -2   openat
# 1613.846148 0  kubectl 1617 1621  -2   openat
# ...
  
# 3. Tracing Syscalls for New Containers
sudo docker run --name tracee --rm --privileged --pid=host \
  -v /lib/modules/:/lib/modules:ro \
  -v /usr/src:/usr/src:ro \
  -v /tmp/tracee:/tmp/tracee \
  aquasec/tracee:0.4.0 --trace container=new
  
# In another terminal window, launch an Ubuntu container that prints a message and exits:
docker run ubuntu echo hi
```

### Restricting syscalls With seccomp
- Restricting syscalls means controlling which Linux kernel actions a process/container is allowed to request.
- simple commands, such as using the touch command, trigger multiple syscalls. For example, running:

```sh
strace -c touch /tmp/error.log
# % time     seconds  usecs/call     calls    errors  syscall
# -------  -----------  -----------  --------  -------  ----------------
#   0.00    0.000000        0     1              1    read
#   0.00    0.000000        0     6              0    close
#   0.00    0.000000        0     2              0    fstat
#   0.00    0.000000        0     5              0    mmap
#   0.00    0.000000        0     4              0    mprotect
#   0.00    0.000000        0     1              0    munmap
#   0.00    0.000000        0     3              0    brk
#   0.00    0.000000        0     3              3    access
#   0.00    0.000000        0     1              0    dup2
#   0.00    0.000000        0     1              0    execve
#   0.00    0.000000        0     1              0    arch_prctl
#   0.00    0.000000        0     3              0    openat
#   0.00    0.000000        0     1              0    utimensat
# -------  -----------  -----------  --------  -------  ----------------
# 100.00    0.000000      32     3    total
```
By default, the Linux kernel permits all user-space programs to invoke any syscall. `Seccomp` (Secure Computing) is a kernel-level feature that allows you to sandbox applications by filtering their allowed syscalls.

To verify if your kernel supports Seccomp, run:
```sh
grep -i seccomp /boot/config-$(uname -r)
# or 
zgrep SECCOMP /proc/config.gz # zgrep searches inside compressed .gz files without manually decompressing them.
# Output:
CONFIG_HAVE_ARCH_SECCOMP_FILTER=y
CONFIG_SECCOMP_FILTER=y
CONFIG_SECCOMP=y
# If you see an output like this with CONFIG_SECCOMP=yes, that means that seccomp is supported by the kernel.
```
#### Demonstrating Seccomp in Action
```sh
~ > docker run -it --rm ubuntu bash
root@9403d420a3a2:/# date -s '19 APR 2012 22:00:00'
date: cannot set date: Permission denied
root@9403d420a3a2:/# 
root@9403d420a3a2:/# ps aux 
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   4776  4324 pts/0    S<s  18:25   0:00 bash
root           9  0.0  0.0   6760  4000 pts/0    R<+  18:26   0:00 ps aux
root@9403d420a3a2:/#
```
Note that the `Bash` runs as `PID 1`
**Why this** `date: cannot set date: Permission denied` ? 
```sh
grep Seccomp /proc/1/status
# /proc/1/status contains information about process ID 1.
# This command searches for the word Seccomp inside that status file.
# Output: 
# Seccomp:	0
```
means: **Process 1 is running with disabled seccomp mode - Seccomp: 0**

**Seccomp Modes**
Seccomp operates in three distinct modes:
- Mode 0: Seccomp is **disabled**
- Mode 1: **Strict** mode, permitting only **four syscalls**: `read`, `write`, `exit`, and `sigreturn`
- Mode 2: **Filter** mode, allowing a defined subset of syscalls based on a filtering profile.

> [!important]
> Docker automatically applies a default Seccomp filter if your host supports Seccomp. This default filter is defined via a JSON document that whitelists approximately 60 syscalls.

#### Default Docker Seccomp Profile
The default Docker profile is designed to block dangerous syscalls such as `ptrace`, which was exploited in the Dirty COW vulnerability. Here is an example snippet of a default Seccomp JSON profile used by Docker:
```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  "syscalls": [
    {
      "names": [
        "arch_prctl",
        "brk",
        "capget",
        "capset",
        "mkdir",
        "close",
        "execve",
        "...",
        "clone"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```
The key elements of any Seccomp JSON profile are:
1. **Architectures** – Defines the supported CPU architectures (e.g., x86_64, x86, x32).
2. **Syscalls** – An array listing syscall names and their permitted actions
3. **Default Action** – Determines how to handle syscalls not explicitly listed. Whitelist profiles typically deny undeclared syscalls.

**A whitelist profile** explicitly allows certain syscalls while denying all others:
```json
{
  // 3. Default Action: reject all other syscalls by default
  "defaultAction": "SCMP_ACT_ERRNO",
  // 1. Architectures
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  // 2. Allowed Syscalls
  "syscalls": [
    {
      "names": [
        "<syscall-1>",
        "<syscall-2>",
        "<syscall-3>"
      ],
	  // Whitelist Profile: Allow these syscalls to run - allows the syscalls declared within it, and blocks everything else by default.
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```
- this Whitelist profile is highly secure, but it is very restrictive.
- If a syscall needed by the application is not added to the whitelist, it can cause the application to fail.
- Before using a whitelist type seccomp profile, you would have to identify every single syscall that will be made by the application.

**A blacklist profile** allows all syscalls by default and only denies those specifically listed:
```json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  "syscalls": [
    {
      "names": [
        "<syscall-1>",
        "<syscall-2>",
        "<syscall-3>"
      ],
	  // block these syscalls and return error
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
```

The default Docker Seccomp profile on x86 blocks around 60 syscalls related to functions such as system time adjustments, file system mounts, and kernel module loading. This is why changing the system time in our earlier container failed:
```sh
root@9403d420a3a2:/# date -s '19 APR 2012 22:00:00'
date: cannot set date: Permission denied
```
- The syscalls for `clock_adjtime` , and `clock_settime` are disabled by Docker by default.
- [The Complete List of all the Blocked syscalls by default](https://docs.docker.com/engine/security/seccomp/)

#### Custom Seccomp Profiles
Although Docker’s default profile enhances security by restricting many dangerous syscalls, you can further harden your container by using a custom Seccomp profile. For example, to block the `mkdir` syscall, and save it as custom.json:
```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  "syscalls": [
    {
      "names": [
        "arch_prctl",
        "brk",
        "capget",
        "capset",
        "close",
        "execve",
        "clone"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

```sh
docker run -it --rm --security-opt seccomp=/root/custom.json docker/whalesay /bin/sh
```

Example:
```sh
# From the running container
root@9403d420a3a2:/# mkdir test
root@9403d420a3a2:/# 
root@9403d420a3a2:/# ll | grep test
drwxr-xr-x   2 root root 4096 Jun  9 19:16 test/
root@9403d420a3a2:/# exit 

# From the host , block mkdir command using blacklist profile:
$ touch custom.json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  "syscalls": [
    {
      "names": ["mkdir", "mkdirat"],
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}

$ docker run -it --rm --security-opt seccomp=custom.json ubuntu /bin/sh

# From the running container
# mkdir test
mkdir: Permission denied
```
Now Can't Create test directory as the custom profile restricts it.

It is also possible to disable Seccomp entirely using the **unconfined flag**:
```sh
docker run -it --rm --security-opt seccomp=unconfined ubuntu /bin/sh
#
```
Even without a Seccomp profile, certain syscalls (like those used to change the system time) may remain blocked by additional Docker security measures:
```sh
docker run -it --rm --security-opt seccomp=unconfined ubuntu /bin/sh
# date -s '19 APR 2012 22:00:00'
date: cannot set date: Operation not permitted
```
Additional security layers are discussed in further lessons.

### Implement Seccomp in Kubernetes
Docker blocks around `60` Syscalls with the **default profile**, to test that using `amicontainerd`
```sh
docker run r.j3ss.co/amicontained amicontained
# new command:
docker run --rm -it --pid host jess/amicontained:v0.4.9 -d
```
The command output indicates that `64` syscalls are blocked due to Docker’s default Seccomp profile. Notice that the Seccomp mode is set to `filtering (mode 2)`:
```text
Container Runtime: docker
Has Namespaces:
    pid: true
    user: false
AppArmor Profile: docker-default (enforce)
Capabilities:
    BOUNDING -> chown dac_override fowner fsetid kill setgid setuid setpcap net_bind_service net_raw sys_chroot mknod audit_write setcap
Seccomp: filtering

# Blocks syscalls: 
Blocked Syscalls (64):
    MSGRCV SYSCFG SETPGID SETSID USELIB USTAT SYSFS VHAVGUP PIVOT_ROOT _SYSCTL ACCT SETTIMEOFDAY MOUNT UMOUNT2 SWAPON SWAPOFF REBOOT SETHOSTNAME SETDOMAINNAME IOPL IOPEM CREATE_MODULE INIT_MODULE DELETE_MODULE GET_KERNEL_SYMS QUERY_MODULE QUOTACLI NESSERVCTL GETPMSG PUTMSG AFS_SYSCALL TUXCALL SECURITY LOOKUP_DCOOKIE CLOCK_SETTIME VSERVER MBIND SET_MEMPOLICY GET_MEMPOLICY KEXEC_LOAD ADD_KEY REQUEST_KEY KEYCTL MIGRATE_PAGES UNSHARE MOVE_PAGES PERF_EVENT_OPEN FANOTIFY_INIT NAME_TO_HANDLE_AT OPEN_BY_HANDLE_AT CLOCK_ADJTIME SETNS PROCESS_VM_READV PROCESS_VM_WRITEV KCMP FINIT_MODULE KEXEC_FILE_LOAD
Looking for Docker.sock
```
**In Kubernetes**: Running the `amicontainerd` Container as a Kubernetes Pod
```sh
kubectl run amicontained --image r.j3ss.co/amicontained amicontained -- amicontained
kubectl logs amicontainerd
```
The log output will display Seccomp as `disabled` along with only `21` syscalls being blocked as Kubernetes V1.20 does not implement seccomp by default:
```text
Container Runtime: docker
Has Namespaces:
  pid: true
  user: false
AppArmor Profile: docker-default (enforce)
Capabilities:
  BOUNDING -> chown dac_override fowner fsetid kill setgid setuid setpcap
  net_bind_service net_raw sys_chroot mknod audit_write setcap
Seccomp: disabled

Blocked Syscalls (21):
  SYSCLOG SETGID SETSID VHANGUP PIVOT_ROOT ACCT SETTIMEOFDAY UMOUNT2 SWAPON
  SWAPOFF REBOOT SETHOSTNAME SETDOMAINNAME INIT_MODULE DELETE_MODULE LOOKUP_DCOOKIE
  KEXEC_LOAD FANOTIFY_INIT OPEN_BY_HANDLE_AT FINIT_MODULE KEXEC_FILE_LOAD

Looking for Docker.sock
```

> [!important]
> By default, Kubernetes pods do not enforce Seccomp filtering; hence, fewer syscalls are blocked.

#### Enabling Seccomp in a Kubernetes Pod
To enable Seccomp filtering in a pod, specify a Seccomp profile in the pod (or container) manifest. The example below demonstrates using the **default Docker profile** via the `seccompProfile` field under the pod-level security context:
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: amicontained
  name: amicontained
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault # it means the default docker profile. the default type is Unconfined , With Unconfied The container runs without seccomp syscall filtering. Kubernetes/container runtime will not block dangerous Linux syscalls using seccomp for that Pod/container
  containers:
    - args:
        - amicontained
      image: r.j3ss.co/amicontained
      name: amicontained
      securityContext:
        allowPrivilegeEscalation: false
```
Setting `allowPrivilegeEscalation`: false restricts the container process from gaining additional privileges beyond what is needed.
```sh
kubectl apply -f pod-definition.yaml
kubectl logs amicontained
```
The logs should now show that Seccomp filtering is active with additional syscalls being blocked:
```text
Container Runtime: docker
Has Namespaces:
  pid: true
  user: false
AppArmor Profile: docker-default (enforce)
Capabilities:
  BOUNDING -> chown dac_override fowner fsetid kill setgid setuid setpcap net_bind_service net_raw

# Filtering mode
Seccomp: filtering

# Blocked Syscalls 
Blocked Syscalls (64):
  SYSCLOG SETPGID SETSID USELIB USTAT SYSFS Vhangup PIVOT_ROOT _SYSCtl ACCT SETTIMEOFDAY MOUNT
  UMOUNT2 SWAPON SWAPOFF REBOOT SETHOSTNAME SETDOMAINNAME IOPL CREATE_MODULE INIT_MODULE DELETE_MODULE
  GET_KERNEL_SYMS QUERY_MODULE QUOTACTL NFS_SERVERCTL GETMSG PUTMSG AFS_SYSCALL TUXCALL SECURITY
  LOOKUP_DCOOKIE CLOCK_SETTIME VSERVER MBIND SET_MPOLICY GET_MEMPOLICY KEXEC_LOAD ADD_KEY REQUEST_KEY
  KEYCTL MIGRATE_PAGES UNSHARE MOVE_PAGES PERF_EVENT_OPEN FANotify_INIT NAME_TO_HANDLE_AT OPEN_BY_HANDLE_AT
  CLOCK_ADJTIME SETNS PROCESS_VM_READV PROCESS_VM_WRITEV KCMP FINIT_MODULE KEXEC_FILE_LOAD BPF USERFAULTFD
Looking for Docker.soc
```

#### Creating a Pod with a Custom Profile
The pod manifest below applies a custom Seccomp profile from a file on the node:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-audit
spec:
  securityContext:
    seccompProfile:
      type: Localhost
	  # Localhost means: use a custom seccomp profile file that already exists on the Kubernetes node.
      localhostProfile: <path to the custom JSON file>
  containers:
    - command: ["bash", "-c", "echo 'I just made some syscalls' && sleep 100"]
      image: ubuntu
      name: ubuntu
      securityContext:
        allowPrivilegeEscalation: false
```
The `localhostProfile` path is relative to the default Seccomp profile directory `/var/lib/kubelet/seccomp`. For example, if you place your custom profile in `/var/lib/kubelet/seccomp/profiles/`, the path might be `profiles/audit.json`
```sh
vi /var/1ib/kubelet/seccomp/profiles/audit.json
```

```json
{
  "defaultAction": "SCMP_ACT_LOG"
// SCMP_ACT_LOG does not block the syscall. It lets the process continue, then records the syscall logs.
}
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-audit
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json
  containers:
    - command: ["bash", "-c", "echo 'I just made some syscalls' && sleep 100"]
      image: ubuntu
      name: ubuntu
      securityContext:
        allowPrivilegeEscalation: false
```
Once the pod is running, all container syscalls are logged to the node’s syslog, You can check the logs with:
```sh
grep syscall /var/log/syslog
```
Once the pod is created, the syscalls generated by the process in the container will be logged in the `/var/log/syslog` file: 
```text
...
syscall=257 compat=0 ip=0x5642801010aa code=0x7ffc0000
...
```
To Find syscall name that related to this syscall number `257`: 
```sh
grep -w 257 /usr/include/asm/unistd_64.h
# #define __NR_openat 257
# Meaning: syscall number 257 = openat
```
Additionally, tools like `Tracee` can be useful for analyzing syscalls. For instance, run the Tracee container to monitor new container syscalls:
```sh
sudo docker run --name tracee --rm --privileged --pid=host \
  -v /lib/modules/:/lib/modules:ro -v /usr/src:/usr/src:ro \
  -v /tmp/tracee:/tmp/tracee aquasec/tracee:0.4.0 --trace container=new
```
#### Creating a Profile That Rejects All Syscalls
To enforce a stricter security posture, you can create a profile that denies any syscall by default. Create a JSON file:
```sh
vi /var/1ib/kubelet/seccomp/profiles/violation.json
```

```json
{
  "defaultAction": "SCMP_ACT_ERRNO"
  // reject any syscalls generated by the container
}
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-audit
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/violation.json
  containers:
    - command: ["bash", "-c", "echo 'I just made some syscalls' && sleep 100"]
      image: ubuntu
      name: ubuntu
      securityContext:
        allowPrivilegeEscalation: false
```
Apply this profile in your pod’s security context. When created, the pod status will be **ContainerCannotRun** because even essential syscalls are blocked. For example:
```sh
kubectl get pods
# NAME             READY   STATUS                RESTARTS   AGE
# test-violation   0/1     ContainerCannotRun    0          2m2s
```

> [!important]
> While a strict Seccomp profile can improve security, it may render the pod non-functional if critical syscalls are blocked.

#### Customizing and Using a Seccomp Profile
- After analyzing your application’s syscall requirements—by inspecting audit logs or using tools like `Tracee`—you can craft a custom Seccomp profile that allows only the required syscalls.
- Create a custom profile JSON file custom.json `/var/lib/kubelet/seccomp/profiles/custom.json` with specific rules , then use it in Pod yaml file:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-custom
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/custom.json
  containers:
    - command: ["bash", "-c", "echo 'I just made some syscalls' && sleep 100"]
      image: ubuntu
      name: ubuntu
  restartPolicy: Never
```
If the profile was created correctly, allowing all system calls required by the application, the pod should start successfully.
```sh
kubectl get pods
# NAME         READY   STATUS    RESTARTS   AGE
# test-custom  1/1     Running   0          2m2s
```

---

## 10. AppArmor
- AppArmor is a Linux security system that restricts what a process/container can do, even if the process is running as root.
- AppArmor, a Linux security module that limits application access to system resources, enhancing container security beyond Seccomp profiles.

```text
seccomp   -> controls syscalls
AppArmor  -> controls file/resource access and process behavior
capabilities -> controls root powers
```
To ensure that AppArmor is active:
```sh
systemctl status apparmor
```
Make sure that the AppArmor kernel module is loaded on every node hosting your container. You can verify this by checking the enabled parameter: 
```sh
cat /sys/module/apparmor/parameters/enabled
# The expected output should be:Y 
```
Like seccomp, AppArmor is applied to an application via a profile. this Profile must be loaded into the kernel, To view all loaded AppArmor profiles, inspect the profiles file:
```sh
cat /sys/kernel/security/apparmor/profiles
```
This command might produce output similar to:
```text
docker-default (enforce)
/usr/sbin/tcpdump (enforce)
/usr/sbin/ntpd (enforce)
/usr/lib/snapd/snap-confine (enforce)
/usr/lib/snapd/snap-confine/mount-namespace-capture-helper (enforce)
/usr/lib/connman/scripts/dhclient-script (enforce)
/usr/lib/NetworkManager/nm-dhcp-helper (enforce)
/usr/lib/NetworkManager/nm-dhcp-client.action (enforce)
/sbin/dhclient (enforce)
/usr/bin/man (enforce)
/usr/bin/man_filter (enforce)
```
An **AppArmor profile** is a plain text file that defines the resources accessible to an application, such as Linux capabilities, network access, and file permissions. 
**For Example,**
1. The profile below restricts write access across the entire filesystem - the profile restricts all file writes within a filesystem can be defined like this:
```text
profile apparmor-deny-write flags=(attach_disconnected) {
    file,
    # Deny all file writes.
    deny /** w,
}
```
It creates an AppArmor profile called `apparmor-deny-write`. The profile mostly allows file access, but explicitly blocks write access to every path.
This simple profile called `AppArmorDenyWrite` contains two rules: 
- The First Rule is **File**, This rule allows complete access to the entire filesystem.
- The Second Rule is **Deny Rule**, which prevents write access to all files under the root filesystem, including the subdirectories.
- This AppArmor profile allows the container/process to **read and access files**, but blocks it from **writing anywhere in the filesystem**

2. The process/container can read files, but it cannot write to direct files inside `/proc`:
```text
profile apparmor-deny-proc-write flags=(attach_disconnected) {
    file,
    # Deny all file writes to /proc.
    deny /proc/* w,
}
```
This AppArmor profile is named: `apparmor-deny-proc-write`
- It allows normal file access generally because of: `file`
- But it blocks write access to files directly under `/proc` because of: `deny /proc/* w,`

3. prevent remounting the root filesystem as read-only:
```text
profile apparmor-deny-remount-root flags=(attach_disconnected) {
  # Deny remounting the root filesystem as read-only.
  deny mount options=(ro, remount) -> /,
}
```

The AppArmor profiles that are loaded and the status can be checked:  
```sh
aa-status
```
We can see that the AppArmor module and `12` AppArmor profiles are loaded into the kernel:
```text
apparmor module is loaded.


12 profiles are loaded.
12 profiles are in enforce mode.
    /sbin/dhclient
    /usr/bin/man
    /usr/lib/NetworkManager/nm-dhcp-client.action
    /usr/lib/NetworkManager/nm-dhcp-helper
    ...
    /usr/sbin/tcpdump
    docker-default
    man_filter
    man_groff
0 profiles are in complain mode.
11 processes have profiles defined.

######## Enforce Mode 
11 processes are in enforce mode: 
    /sbin/dhclient (621)
    docker-default (3970)
    docker-default (4025)
    docker-default (9853)
    docker-default (9964)

######## Complain Mode 
0 processes are in complain mode.

######## Unconfined Mode 
0 processes are 'unconfined' but have a profile defined.
```

### AppArmor Profile Modes
Profiles can be loaded in three different modes: `enforce`, `complain`, and `unconfined`
- **Enforce mode**: The profile rules are strictly enforced on the application - AppArmor will **monitor and enforce** the rules on any application that fits the profile.
- **Complain mode**: The application is allowed to **perform any actions Without any restriction** but it will **log them as events**
- **Unconfined mode**: No restrictions are applied, and actions are not logged.

### Creating AppArmor Profiles
Below is a sample Bash script named `add_data.sh` that creates directory `/opt/app/data` and writes a log file within the new directory:
```sh
#!/bin/bash
data_directory=/opt/app/data
mkdir -p "${data_directory}"
echo "=> File created at $(date)" | tee "${data_directory}/create.log"
```

```sh
cat /opt/app/data/create.log
# Expected terminal output: => File created at Mon Mar 12 03:29:22 UTC 2021
```
Instead of creating a profile manually, you can use AppArmor’s built-in tools. First, install the AppArmor-utils package. On Ubuntu, run:
```sh
apt-get install -y apparmor-utils
```
Once installed, generate a profile for the Bash script using the following command:
```sh
aa-genprof /root/add_data.sh
```
The output will be similar to:
```text
Writing updated profile for /root/add_data.sh.
Setting /root/add_data.sh to complain mode!

Before you begin, you may wish to check if a profile already exists for the application you wish to confine. See the following wiki page for more information:
https://gitlab.com/apparmor/apparmor/wikis/Profiles
Profiling: /root/add_data.sh

Please start the application to be profiled in another window and exercise its functionality now.

Once completed, select the "Scan" option below in order to scan the system logs for AppArmor events.

For each AppArmor event, you will be given the opportunity to choose whether the access should be allowed or denied.

[(S)can system log for AppArmor events] / (F)inish
```
Then From a separate terminal run the Bash script to generate AppArmor events:
```sh
./add_data.sh
```
 Return to the `aa-genprof` prompt and press `S` to scan the system logs. The tool will then display multiple prompts for each event, such as:
 ```text
Profile: /root/add_data.sh
Execute: /usr/bin/mkdir
Severity: unknown 

Press: I (inherit) => To allow the execution of the mkdir command, choose the inherit option by entering I

Profile: /root/add_data.sh
Execute: /usr/bin/tee
Severity: 3
(I)nherit / (C)hild / (P)rofile / (N)amed / (U)nconfined / (X) ix On / (D)eny / Abo(r)t / (F)inish

Press: I 

Another prompt may request permission to access the tty interface. If a prompt with severity 9 appears when printing to the console, enter a (allow).


Profile: /root/add_data.sh
Path: /proc/filesystems
New Mode: owner r
Severity: 6
[1] 'owner /proc/filesystems r,'
(A)llow / [D]eny / [I]gnore / [G]lob / Glob with [E]xtension / [N]ew /
Audi(t) / [O]wner permissions off / Abo(t) / [F]inish

Since the script does not need access to this file, choose d to deny access.


 ```
 After processing all events, press `S` to save and `F` to finish. Your new AppArmor profile is now running in `enforce` mode.
**Example**
```sh
aa-genprof /root/data.sh
```
From Another Terminal:
```sh
./data.sh 
```
Back to the First Terminal:
```text
For each AppArmor event, you will be given the 
opportunity to choose whether the access should be 
allowed or denied.

[(S)can system log for AppArmor events] / (F)inish
Press: S


Profile:  /root/data.sh
Execute:  /usr/bin/mkdir
Severity: unknown

(I)nherit / (C)hild / (N)amed / (X) ix On / (D)eny / Abo(r)t / (F)inish
Press: I


Profile:  /root/data.sh
Execute:  /usr/bin/tee
Severity: 3

(I)nherit / (C)hild / (P)rofile / (N)amed / (U)nconfined / (X) ix On / (D)eny / Abo(r)t / (F)inish
I

Profile:  /root/data.sh
Execute:  /usr/bin/date
Severity: unknown

(I)nherit / (C)hild / (P)rofile / (N)amed / (U)nconfined / (X) ix On / (D)eny / Abo(r)t / (F)inish
I


Profile:  /root/data.sh
Path:     /dev/tty
New Mode: owner rw
Severity: 9

 [1 - include <abstractions/consoles>]
  2 - owner /dev/tty rw, 
(A)llow / [(D)eny] / (I)gnore / (G)lob / Glob with (E)xtension / (N)ew / Audi(t) / (O)wner permissions off / Abo(r)t / (F)inish
A


 [1 - owner /proc/filesystems r,]
(A)llow / [(D)eny] / (I)gnore / (G)lob / Glob with (E)xtension / (N)ew / Audi(t) / (O)wner permissions off / Abo(r)t / (F)inish
D


 [1 - owner /etc/locale.alias r,]
(A)llow / [(D)eny] / (I)gnore / (G)lob / Glob with (E)xtension / (N)ew / Audi(t) / (O)wner permissions off / Abo(r)t / (F)inish
D

...


The following local profiles were changed. Would you like to save them?

 [1 - /root/data.sh]
(S)ave Changes / Save Selec(t)ed Profile / [(V)iew Changes] / View Changes b/w (C)lean profiles / Abo(r)t
S

[(S)can system log for AppArmor events] / (F)inish
F
```

```sh
# aa-status
apparmor module is loaded.
127 profiles are loaded.
25 profiles are in enforce mode.
   /root/data.sh
...

## The new profile, along with other existing profiles, is stored in the /etc/apparmor.d directory
/etc/apparmor.d# pwd
/etc/apparmor.d

/etc/apparmor.d# ls  | grep data.sh
root.data.sh

/etc/apparmor.d# cat root.data.sh 
# Last Modified: Thu Jun 11 20:48:53 2026
abi <abi/3.0>,

include <tunables/global>

/root/data.sh {
  include <abstractions/base>
  include <abstractions/bash>
  include <abstractions/consoles>

  deny owner /etc/ld.so.cache r,
  deny owner /etc/locale.alias r,
  deny owner /proc/filesystems r,

  /root/data.sh r,
  /usr/bin/bash ix,
  /usr/bin/date mrix,
  /usr/bin/mkdir mrix,
  /usr/bin/tee mrix,
  owner /opt/app/data/ r,
  owner /opt/app/data/create.log w,

}

```
**Testing the Enforced Profile**: 
To verify that the enforced profile restricts unauthorized access, modify the script to change the log file path from `/opt/app/data` to `/opt`
 ```sh
root@ubuntu-s-2vcpu-4gb-120gb-intel-nyc1:~# 
root@ubuntu-s-2vcpu-4gb-120gb-intel-nyc1:~# ./data.sh 
=> File created at Thu Jun 11 20:56:27 UTC 2026
root@ubuntu-s-2vcpu-4gb-120gb-intel-nyc1:~# date 
Thu Jun 11 20:56:31 UTC 2026

root@ubuntu-s-2vcpu-4gb-120gb-intel-nyc1:~# 
root@ubuntu-s-2vcpu-4gb-120gb-intel-nyc1:~# vi data.sh 
root@ubuntu-s-2vcpu-4gb-120gb-intel-nyc1:~# 
root@ubuntu-s-2vcpu-4gb-120gb-intel-nyc1:~# cat data.sh 
#!/bin/bash
data_directory=/opt ################################################### changed from /opt/app/data to /opt
mkdir -p "${data_directory}"
echo "=> File created at $(date)" | tee "${data_directory}/create.log"
root@ubuntu-s-2vcpu-4gb-120gb-intel-nyc1:~# 
root@ubuntu-s-2vcpu-4gb-120gb-intel-nyc1:~# ./data.sh 
tee: /opt/create.log: Permission denied ###################################### Denied
=> File created at Thu Jun 11 20:56:49 UTC 2026
 ```
It is denied as **AppArmor only allows writes to the directory `/opt/app/data` not directory to `/opt`**

To load the Profile:
```sh
apparmor_parser /etc/apparmor.d/root.data.sh
```
To Disable the Profile
```sh
apparmor_parser -R /etc/apparmor.d/root.data.sh
# Then create a symlink to the profile in the /etc/apparmor.d/disable directory
ln -s /etc/apparmor.d/root.data.sh /etc/apparmor/disable
```

### AppArmor in Kubernetes
To enable AppArmor on Kubernetes pods, ensure that each node in your cluster meets the following requirements: 
- The AppArmor kernel module must be enabled.
- The AppArmor profile must be loaded on each node.
- The container runtime (e.g., Docker, CRI-O, Containerd) must support AppArmor.

1. Deny write permission to every path under `/`, `/etc/apparmor.d/apparmor-deny-write`: 
```json
profile apparmor-deny-write flags=(attach_disconnected) {
    file,
    # Deny all file writes.
    deny /** w,
}
```

2. Before deploying your pod, ensure the AppArmor profile **is loaded on all nodes**. Run the following command on each worker node:
```sh
# Load the Profile
apparmor_parser /etc/apparmor.d/apparmor-deny-write
```

```sh
# Check it is loaded:
$ sudo aa-status | grep apparmor-deny-write
# or
$ aa-status
# apparmor module is loaded.
# 13 profiles are loaded.
# 13 profiles are in enforce mode.
#     apparmor-deny-write ############################ this is the required profile that deny all write in filesystem

# Also check AppArmor is enabled:
cat /sys/module/apparmor/parameters/enabled
# Expected: Y
```

3. Use it in Kubernetes with Localhost:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: apparmor-deny-write-test
spec:
  containers:
    - name: test
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        appArmorProfile:
          type: Localhost
          localhostProfile: apparmor-deny-write
```
Test the AppArmor Profile:
```sh
kubectl exec -ti ubuntu-sleeper -- touch /tmp/test
```
The operation should fail with a permission denied error:
```text
touch: cannot touch '/tmp/test': Permission denied
command terminated with exit code 1
```

> [!note]
> If the file creation attempt fails as shown above, it confirms that the AppArmor profile is enforcing the security restrictions correctly.

---

## 11. Linux Capabilities
By default, Kubernetes pods do not utilize Seccomp, and a container—even running as root (UID 0)—may still be restricted from performing certain operations:
```sh
$ docker run -it --rm --security-opt seccomp=unconfined docker/whalesay /bin/sh
# date -s '19 APR 2012 22:00:00'
date: cannot set date: Operation not permitted
Thu Apr 19 22:00:00 UTC 2012
```

```sh
$ kubectl run --rm -it ubuntu-sleeper --image=ubuntu -- bash
If you don't see a command prompt, try pressing enter.
root@ubuntu-sleeper:/# date -s '19 APR 2012 22:00:00'
date: cannot set date: Operation not permitted
Thu Apr 19 22:00:00 UTC 2012
root@ubuntu-sleeper:/#

root@ubuntu-sleeper:/# whoami
root
root@ubuntu-sleeper:/#

root@ubuntu-sleeper:/# id
uid=0(root) gid=0(root) groups=0(root)
root@ubuntu-sleeper:/#
```

### Understanding Linux Process Privileges
**Before Linux kernel 2.2**, processes were classified into:
- Privileged processes: Run by the root user (UID 0) and bypass many kernel permission checks.
- Unprivileged processes: Run by non-root users and are subject to various kernel restrictions.

**Starting with Linux kernel 2.2**, 
- the traditional superuser privileges were broken down into individual units called capabilities. This allows administrators to grant only specific privileges to processes, **even if they run as the root user**.
- Some examples of these capabilities include:
	- **CAP_CHOWN**: Allows changing file ownership.
	- **CAP_NET_ADMIN**: Permits operations like modifying network interface configurations, managing routing tables, and binding processes to specific addresses.
	- **CAP_SYS_BOOT**: Enables a process to reboot the system.
	- **CAP_SYS_TIME**: Permits setting or adjusting the system clock.

### Checking Capabilities
 You can determine the capabilities required by a command using the `getcap` command. For example, the `ping` command requires the `CAP_NET_RAW` capability:
 ```sh
getcap /usr/bin/ping
# The expected output is: /usr/bin/ping = cap_net_raw+ep
 ```

To inspect the capabilities of a running process, use the `getpcaps` command:
```sh
ps -ef | grep /usr/sbin/sshd | grep -v grep
# root     779     1  0 03:55 ?        00:00:00 /usr/sbin/sshd -D

# Use getpcaps with the PID:
getpcaps 779
```

### Linux Capabilities in Kubernetes Containers
```sh
kubectl run --rm -it ubuntu-sleeper --image=ubuntu -- bash
root@ubuntu-sleeper:# date -s '19 APR 2012 22:00:00'
date: cannot set date: Operation not permitted
Thu Apr 19 22:00:00 UTC 2012
root@ubuntu-sleeper:#
```
- This failure occurs because containers, even when running as root, are started with a limited set of capabilities. By Default , Docker, the container runtime, initiates containers with only `14` capabilities Without the specific capability `CAP_SYS_TIME` required to modify the system clock, the operation is prohibited.

#### Default Capabilities in Linux 
The following Go code snippet demonstrates how default capabilities are defined in a Linux environment:
```go
// DefaultCapabilities returns a Linux kernel default capabilities
func DefaultCapabilities() []string {
    return []string{
        "CAP_CHOWN",
        "CAP_DAC_OVERRIDE",
        "CAP_FOWNER",
        "CAP_MKNOD",
        "CAP_NET_RAW",
        "CAP_SETGID",
        "CAP_SETUID",
        "CAP_SETFCAP",
        "CAP_SETPCAP",
        "CAP_NET_BIND_SERVICE",
        "CAP_SYS_CHROOT",
        "CAP_KILL",
        "CAP_AUDIT_WRITE",
    }
}
```

### Modifying Container Capabilities
1. Added Capability 
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu
    command: [ "sleep", "1000" ]
    securityContext:
      capabilities:
        add: ["SYS_TIME"]
```

```sh
$ kubectl apply -f ubuntu-sleeper.yaml
$ kubectl exec -ti ubuntu-sleeper -- bash
root@ubuntu-sleeper:/# date
Sat Apr  3 05:32:06 UTC 2021

root@ubuntu-sleeper:/# date -s '19 APR 2012 22:00:00'
Thu Apr 19 22:00:00 UTC 2012

root@ubuntu-sleeper:/# date
Thu Apr 19 22:00:02 UTC 2012

root@ubuntu-sleeper:/#
```

2. Remove Capability
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu
    command: [ "sleep", "1000" ]
    securityContext:
      capabilities:
        drop: ["CHOWN"]
```

```sh
$ kubectl apply -f ubuntu-sleeper.yaml
$ kubectl exec -ti ubuntu-sleeper -- bash
root@ubuntu-sleeper:~# touch /tmp/test

root@ubuntu-sleeper:~# ls -l /tmp/test
-rw-r--r-- 1 root root 0 Apr  3 05:46 /tmp/test

root@ubuntu-sleeper:~# chown backup /tmp/test
chown: changing ownership of '/tmp/test': Operation not permitted

root@ubuntu-sleeper:~#
```

---

