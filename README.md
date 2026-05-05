# AF_ALG Socket Detection for OpenShift

````markdown
# AF_ALG Socket Detection for OpenShift

This repository provides a small OpenShift DaemonSet to detect attempts to create `AF_ALG` sockets from containerized workloads.

`AF_ALG` socket creation is relevant because some kernel exploitation techniques require opening an `AF_ALG` socket as an initial step. This DaemonSet adds Linux audit rules on each node and reports matching events through pod logs.

> Detection only. To prevent exploitation, use a seccomp profile that blocks `socket(AF_ALG, ...)`.

---

## What this does

The DaemonSet runs one privileged pod per node and configures Linux audit rules for:

```c
socket(AF_ALG, ...)
````

Where:

```text
AF_ALG = 38 decimal = 0x26 hexadecimal
```

The audit rules installed are:

```bash
auditctl -a always,exit -F arch=b64 -S socket -F a0=0x26 -k af_alg_socket
auditctl -a always,exit -F arch=b32 -S socket -F a0=0x26 -k af_alg_socket
```

When a process attempts to create an `AF_ALG` socket, the DaemonSet prints an alert such as:

```text
[ALERT][AF_ALG_SOCKET_ATTEMPT][node=worker-1] ...
```

---

## Files

Recommended repository structure:

```text
.
├── README.md
├── manifests
│   ├── 00-namespace.yaml
│   ├── 01-serviceaccount.yaml
│   ├── 02-scc-rbac.yaml
│   └── 03-daemonset.yaml
└── tests
    └── afalg-test.yaml
```

---

## Requirements

* OpenShift 4.x
* Cluster-admin permissions to deploy the privileged DaemonSet
* RHCOS nodes with `auditctl` available on the host
* Access to `registry.redhat.io`

---

## Deployment

Apply all manifests:

```bash
oc apply -f manifests/
```

Verify the DaemonSet:

```bash
oc get pods -n af-alg-monitor -o wide
```

Expected result:

```text
NAME                         READY   STATUS    NODE
af-alg-audit-monitor-xxxxx   1/1     Running   worker-1
af-alg-audit-monitor-yyyyy   1/1     Running   worker-2
```

Check that audit rules were installed:

```bash
oc logs -n af-alg-monitor -l app=af-alg-audit-monitor --tail=50
```

Expected log:

```text
[INFO] Adding AF_ALG detection rules...
[INFO] Active rules:
-a always,exit -F arch=b64 -S socket -F a0=0x26 -F key=af_alg_socket
```

---

## Test the detection

Create a test pod that attempts to open an `AF_ALG` socket:

```bash
oc run afalg-test \
  --image=registry.redhat.io/ubi9/python-39 \
  --restart=Never \
  --command -- python3 -c 'import socket; socket.socket(38, socket.SOCK_SEQPACKET, 0); print("AF_ALG attempted")'
```

Then watch the DaemonSet logs:

```bash
oc logs -n af-alg-monitor -l app=af-alg-audit-monitor -f
```

Expected alert:

```text
[ALERT][AF_ALG_SOCKET_ATTEMPT][node=<node-name>] type=SYSCALL msg=audit...
```

---

## Validate directly on the node

Open a debug shell on the node where the test pod ran:

```bash
oc debug node/<node-name>
chroot /host
```

Search for audit events:

```bash
ausearch -k af_alg_socket
```

Expected output includes:

```text
type=SYSCALL ... syscall=socket ... a0=26 ...
```

`a0=26` is hexadecimal and corresponds to decimal `38`, which is `AF_ALG`.

---

## Limitations

* This solution requires privileged access because it configures host-level audit rules.
* It detects socket creation attempts but does not prove full exploitation.
* If seccomp blocks the syscall before it reaches audit, the audit event may not appear.
* Audit log volume can increase if many workloads repeatedly attempt this syscall.
* This should be combined with kernel patching and seccomp enforcement.

---

## Cleanup

Remove all resources:

```bash
oc delete -f manifests/
```

Optionally remove the audit rules manually from a node:

```bash
oc debug node/<node-name>
chroot /host

auditctl -d always,exit -F arch=b64 -S socket -F a0=0x26 -k af_alg_socket
auditctl -d always,exit -F arch=b32 -S socket -F a0=0x26 -k af_alg_socket
```

---

## Security note

This project is intended for defensive validation and monitoring in controlled OpenShift environments.

Recommended actions:

1. Patch affected kernels.
2. Block `AF_ALG` socket creation with seccomp.
3. Use this DaemonSet to detect attempted use.
4. Investigate any alert as suspicious unless the workload has a known legitimate need for Linux kernel crypto API sockets.

```
```

