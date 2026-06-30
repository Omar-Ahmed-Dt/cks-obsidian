---
tags:
  - cks
  - kubernetes
---
# Falco Overview and Installation
- **Falco** is used for runtime threat detection. It watches activity from containers, hosts, and Kubernetes workloads, then alerts when behavior matches suspicious rules. 
- Falco docs describe it as near real-time threat detection using runtime insights from sources like the Linux kernel, Kubernetes API metadata, and container runtime metadata
- Falco finds suspicious behavior after the container is already running.
- Falco is a runtime security tool that monitors Linux syscalls from containers and hosts, then checks them against Falco rules to detect suspicious behavior like shell access, reading sensitive files, privilege escalation, or log deletion
- After Falco captures syscalls, it sends them to the Falco policy engine. The engine compares events with rules, then sends alerts to stdout, syslog, Slack, email.

My Article: [ Falco With Kubernetes ](https://dev.to/omar_ahmed/falco-2f7n)

---

# Use Falco to Detect Threats
```sh
# Open shell inside container:
kubectl exec -ti nginx -- bash
# Read sensitive file inside container:
cat /etc/shadow
```
Falco should detect both actions:
```text
Shell spawned inside container
Sensitive file opened by container
```

**Falco rules** are written in YAML. Every rule has 5 main parts:

| Key         | Meaning                      |
| ----------- | ---------------------------- |
| `rule`      | Rule name                    |
| `desc`      | Description                  |
| `condition` | When the rule should trigger |
| `output`    | Alert message                |
| `priority`  | Severity level               |

Example rule: If any shell process like `bash/sh/zsh` starts inside a container, generate an alert.
```yaml
- rule: Detect Shell inside a container
  desc: Alert if a shell such as bash is open inside a container
  condition: container and proc.name in (linux_shells)
  # Trigger alert if:
  # 	1. The event happened inside a container
  # 	AND
  # 	2. The process name is one of the Linux shell names - will explain it below 
  
  output: Bash Opened (user=%user.name container=%container.id)
  # %user.name       -> user who started the process
  # %container.id    -> container ID
  
  priority: WARNING # alert severity
- list: linux_shells # This defines a reusable list called linux_shells
  items: [bash, zsh, ksh, sh, csh]
  # Instead of writing:
  # proc.name=bash or proc.name=zsh or proc.name=ksh or proc.name=sh or proc.name=csh
  # you write:
  # proc.name in (linux_shells) 

- macro: container # This defines a reusable condition called container.
  condition: container.id != host # The event is from a container, not from the host.
```

---

# Falco Configuration Files

| File                                | Purpose                        |
| ----------------------------------- | ------------------------------ |
| `/etc/falco/falco.yaml`             | Main Falco config file         |
| `/etc/falco/falco_rules.yaml`       | Default built-in rules         |
| `/etc/falco/falco_rules.local.yaml` | Your custom rules or overrides |
| `/etc/falco/k8s_audit_rules.yaml`   | Kubernetes audit rules         |
| `/etc/falco/rules.d/`               | Extra rule files directory     |

**Main Config File**
Falco starts using `/etc/falco/falco.yaml`, You can verify it from logs `journalctl -u falco`, You should see something like `Falco initialized with configuration file /etc/falco/falco.yaml`

**Rule Loading Order**
Inside `/etc/falco/falco.yaml`, Falco loads rule files using `rules_file`: 
```text
# Rules files
rules_file:
  - /etc/falco/falco_rules.yaml
  - /etc/falco/falco_rules.local.yaml
  - /etc/falco/k8s_audit_rules.yaml
  - /etc/falco/rules.d/

# Output Settings, will explain it below
stdout_output:
...

```
  
**Order Matters**:
- If the same rule exists in multiple files, the rule loaded later overrides the earlier one.
- So this file can override default rules: `/etc/falco/falco_rules.local.yaml`
- `/etc/falco/falco_rules.yaml` Do Not Edit This Directly, Because it contains default rules and can be overwritten during Falco updates.
- Edit This Instead `/etc/falco/falco_rules.local.yaml`

Example override:
```yaml
- rule: Terminal shell in container
  desc: A shell was used as the entrypoint/exec point into a container with an attached terminal.
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
  output: >
    A shell was spawned in a container
    (user=%user.name shell=%proc.name container_id=%container.id image=%container.image.repository)
  priority: WARNING
```

| Part              | Meaning                                         |
| ----------------- | ----------------------------------------------- |
| `spawned_process` | A new process was created                       |
| `container`       | The process happened inside a container         |
| `shell_procs`     | The process is a shell like `bash`, `sh`, `zsh` |
| `proc.tty != 0`   | The shell has an interactive terminal attached  |
- The condition means: A new shell process started inside a container with an **interactive terminal**: `kubectl exec -it nginx -- bash` , `-it` attaches a terminal. 
- But this may not trigger the same way `kubectl exec nginx -- bash -c "ls"`, Because it may run non-interactively.

**Output Settings**: `/etc/falco/falco.yaml`, ***Out of Scope of this course***
Falco can output alerts to stdout, file, program, syslog, or HTTP. Example:
```yaml
stdout_output:
  enabled: true

file_output:
  enabled: true
  filename: /opt/falco/events.txt

http_output:
  enabled: true
  url: http://some.url/some/path/
```

Reload Falco After Changes using `systemctl restart falco`, or Hot reload without full restart using `kill -1 $(cat /var/run/falco.pid)`

---

# Mutable vs Immutable Infrastructure
- Mutable means change running systems.
- Immutable means rebuild and redeploy instead of modifying live containers.

**Mutable Infrastructure**
You change the running server/container directly.
```sh
ssh node01
apt update
apt install nginx
vi /etc/nginx/nginx.conf
systemctl restart nginx
```
In Kubernetes/container style:
```sh
kubectl exec -it nginx -- bash
apt install curl
vi /etc/nginx/nginx.conf
```
**Problem**: The running system changes over time.
**This causes**:
- Configuration drift
- Hard rollback
- Different servers/pods behaving differently
- More security risk
- Harder investigation after compromise

**Immutable Infrastructure**
You do not modify the running system. You build a new version and replace the old one.
```sh
docker build -t myapp:v2 .
docker push myapp:v2
kubectl set image deployment/myapp myapp=myapp:v2

# Rollback is easier:
kubectl rollout undo deployment/myapp
```
Immutable infrastructure reduces attack impact. If a container is compromised, you should not `fix` it manually inside the pod. You should:
```sh
kubectl delete pod <pod-name>
# or redeploy a clean image:
kubectl rollout restart deployment/<deployment-name>

# Important Security Settings
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

> [!summary]
> - Mutable = patch live systems.
> - Immutable = rebuild, redeploy, replace.

## Ensure Immutability of Containers at Runtime
**Mutable Infrastructure** means you update the running system directly.
Example:
```sh
kubectl exec -it nginx -- bash
apt install curl
vi /etc/nginx/nginx.conf
```
This is bad for Kubernetes security because the container is changed after deployment.

**Immutable Infrastructure** means you never modify the running container. You build a new image and redeploy.
```sh
docker build -t myapp:v2 .
docker push myapp:v2
kubectl set image deployment/myapp myapp=myapp:v2

# Rollback:
kubectl rollout undo deployment/myapp
```

**Security Context For Immutable Containers**
```yaml
securityContext:
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  # Nginx container starts with a read-only root file system, preventing any unauthorized copying or writing. However, this configuration might disrupt application functionality. For example, deploying the pod as configured above could result in an error because Nginx typically requires write permissions for certain directories.
  
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```
If you create the pod with this configuration, you may see the following output:
```sh
kubectl create -f nginx.yaml
pod/nginx created

kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   0/1     Error     0          20s
```
Nginx requires write access to directories such as `/var/run` (to store runtime data) and `/var/cache/nginx` (for caching). The pod logs will indicate failures when it attempts to write to these directories.

> [!important]
> Before enforcing a read-only file system, ensure your applications do not depend on writing to the root file system during runtime.

### Using Volumes to Allow Limited Write Access
To resolve these issues, mount volumes on the directories that require write access. In the example below, we use an `emptyDir` volume since the data does not need to persist after the pod terminates - `emptyDir` Create an empty writable directory and mount it inside the container: 
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: cache-volume
      mountPath: /var/cache/nginx
    - name: runtime-volume
      mountPath: /var/run
  volumes:
  - name: cache-volume
    emptyDir: {}
  - name: runtime-volume
    emptyDir: {}
```
After applying this configuration, the `/var/cache/nginx` and `/var/run` directories inside the container become writable through the mounted volumes, while the rest of the file system remains read-only.

### Testing the Immutable Container with Privileged Mode
Create a pod with the configuration below:
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    securityContext:
      readOnlyRootFilesystem: true
      privileged: true # means the container gets almost full access to the host system.
      # This is dangerous because the container can access host-level resources more freely, like devices, kernel features, and capabilities.
      
    volumeMounts:
    - name: cache-volume
      mountPath: /var/cache/nginx
    - name: runtime-volume
      mountPath: /var/run
  volumes:
  - name: cache-volume
    emptyDir: {}
  - name: runtime-volume
    emptyDir: {}
```
On deployment, you might observe messages similar to:
```sh
kubectl create -f nginx.yaml
pod/nginx created

kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          20s
```
Attempting a package update inside the container will still fail due to the read-only root file system:
```sh
kubectl exec -ti nginx -- apt update
Reading package lists... Done
E: List directory /var/lib/apt/lists/partial is missing. - Acquire (30: Read-only file system)
command terminated with exit code 100
```

### Best Practices for Container Immutability
To ensure that your containers remain immutable, follow these best practices:

| Best Practice              | Description                                                                                           |
| -------------------------- | ----------------------------------------------------------------------------------------------------- |
| Read-Only Root File System | Set containers with a read-only root file system to prevent in-place modifications.                   |
| Limited Write Volumes      | Mount volumes (e.g., `emptyDir` or persistent volumes) only on directories that require write access. |
| Avoid Privileged Mode      | Refrain from using the privileged flag to limit the container’s impact on the host system.            |
| Non-Root Containers        | Run containers as non-root users whenever possible to minimize risks.                                 |
| Enforce Security Policies  | Use Pod Security Policies (PSPs) to enforce immutability and other security best practices.           |

Below is an example of a Pod Security Policy that reinforces these practices:
```yaml theme={null}
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: example
spec:
  privileged: false
  readOnlyRootFilesystem: true
  runAsUser:
    rule: RunAsNonRoot
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
```

This policy ensures that containers are non-privileged, have a read-only root file system, run as non-root users, and do not carry unnecessary privileges.

---

# Kubernets Auditing
**Kubernetes audit policy**: It tells the Kubernetes API server **what actions should be logged** and **how much detail should be logged**.
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages: ["RequestReceived"] # Do not log the early stage when the API server first receives the request. This reduces noisy audit logs.
rules:
  - namespaces: ["prod-namespace"]
    verbs: ["delete"]
    resources:
      - groups: ""
        resources: ["pods"]
        resourceNames: ["webapp-pod"]
    level: RequestResponse
	# If someone deletes the pod named webapp-pod in namespace prod-namespace, log the request and response details.
	# level: RequestResponse is very detailed. It logs metadata plus request body and response body.

  - level: Metadata
    resources:
      - groups: ""
        resources: ["secrets"]
		# For any action on Secrets, log only metadata.
		# But it does not log the secret data itself.
```

If API-Server is running as a service: `kube-apiserver.service`: 
```sh
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --enable-swagger-ui=true \\
  --etcd-servers=https://127.0.0.1:2379 \\
  --event-ttl=1h \\
  --runtime-config=api/all \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --v=2 \\
  --audit-log-path=/var/log/k8-audit.log \\ # Where logs are stored
  --audit-policy-file=/etc/kubernetes/audit-policy.yaml \\ # Tells the API server which audit policy file to use
  --audit-log-maxage=10 \\ # Keep old audit log files for 10 days.
  --audit-log-maxbackup=5 \\ # Keep maximum 5 old audit log files.
  --audit-log-maxsize=100  # Rotate the log file when it reaches 100 MB.
```

OR if API-Server is running as a pod: `/etc/kubernetes/manifests/kube-apiserver.yaml`: 
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
    - command:
        - kube-apiserver
        - --authorization-mode=Node,RBAC
        - --advertise-address=172.17.0.107
        - --allow-privileged=true
        - --enable-bootstrap-token-auth=true
        - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
        - --audit-log-maxage=10
        - --audit-log-maxbackup=5
        - --audit-log-maxsize=100
```

---
