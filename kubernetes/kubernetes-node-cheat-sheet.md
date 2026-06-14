Cheat sheet for diagnosing and resolving general Kubernetes node issues in cloud environments.

## Node Troubleshooting Flowchart
```
                          [Is Node Ready?]
                             /        \
                          (Yes)       (No)
                           /            \
           [Check Pod Status]          [Run: kubectl describe node]
             /           \                    /              \
     (Running)         (Pending)       (Condition Flagged)   (No Response/Unknown)
        /                 /                   /                      \
 [Healthy Node]   [Check Taints/       [Fix Pressure:         [SSH into Node Instance]
                   Resource Limits]     RAM, Disk, PID]           /            \
                                                           [Check Services]   [Check System]
                                                            systemctl status   dmesg -T | grep
                                                            kubelet/runtime    -i "oom/panic"
```

## Initial Triage & Node Status
Run these commands from your local workstation or bastion host to identify the scope of the failure.

* `kubectl get nodes`: Checks the high-level status of all nodes in the cluster.
* `kubectl describe node <node-name>`: Reveals the Conditions block (e.g., MemoryPressure, DiskPressure, PIDPressure, NetworkUnavailable).
* `kubectl get events --sort-by='.metadata.creationTimestamp'`: Shows a cluster-wide chronological timeline of recent infrastructure errors.

## Node Condition Diagnostics
If kubectl describe node flags a specific pressure condition, use the following mapping to troubleshoot the underlying system resource:

| Condition Flag | Common Root Cause | Actionable Linux Commands (SSH into Node) |
|---|---|---|
| DiskPressure | Container logs, images, or root volume full. | `df -h, du -h --max-depth=1 /var/lib/docker` (or `/var/lib/containerd`) |
| MemoryPressure | High RAM utilization or daemon memory leaks. | `free -h`, `ps aux --sort=-%mem \| head -n 10` |
| PIDPressure | Too many active processes/threads on the host. | `sysctl kernel.pid_max`, `ps -eLf \| wc -l` |
| NetworkUnavailable | CNI plugin crash or routing configuration failure. | `ip route show`, `ip link show` |

## Deep Dive via Node SSH
If the node is completely unresponsive to the control plane, SSH into the underlying instance to inspect the container runtime and kubelet.

* `systemctl status kubelet`: Checks if the primary Kubernetes node agent is running, crashed, or dead.
* `journalctl -u kubelet -n 100 --no-pager`: Fetches the last 100 log lines from the kubelet to find API registration failures or certificate expiration errors.
* `systemctl status containerd` (or `docker`): Verifies that the underlying container runtime engine is operating normally.
* `journalctl -k -p err \| grep -i oom`: Checks the kernel logs to see if the Linux Out-Of-Memory killer terminated vital system daemons.  

## Automated & Fast Remediations
When a node cannot be safely debugged in real-time, use these orchestration commands to safeguard your applications:

* `kubectl cordon <node-name>`: Marks the node as unschedulable to prevent Kubernetes from placing new pods on it.
* `kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data`: Safely evicts all active workloads from the failing node and reschedules them onto healthy nodes.
* Cloud Auto-Healing: In managed environments (AWS EKS, GCP GKE, Azure AKS), terminating the underlying virtual machine instance will trigger the Node Group / Auto-Scaling Group to automatically replace it with a fresh, healthy node. [20, 21, 22, 23, 24] 

## Control Plane & Agent Failures
These issues prevent the node from communicating with the Kubernetes API server, forcing a *NotReady* status.

* Kubelet Crash: The primary node agent stops running. Check status with `systemctl status kubelet` on the node.
* Expired Certificates: Kubelet client certificates fail to auto-renew. Check logs using `journalctl -u kubelet | grep -i "cert"`.
* Runtime Disruption: Containerd or Docker freezes or crashes. Inspect the engine status via `systemctl status containerd`.
* API Server Timeout: Network or firewall rules block the node from reaching the control plane endpoint. Test connectivity with `nc -zv <api-server-ip> 6443`.
* Misconfigured Cgroup Driver: A mismatch between the Kubelet and container runtime cgroup drivers (e.g., `cgroupfs` vs `systemd`) causes immediate Kubelet crashes. Check `/var/lib/kubelet/config.yaml`.

## Node Resource Pressures (Eviction Triggers)
When physical hardware thresholds are crossed, the Kubelet flags these conditions in `kubectl describe node <name>` and begins evicting pods. 

* MemoryPressure: Available node memory falls below the eviction threshold (typically < 100Mi). Run `free -h` to verify.
* DiskPressure: Root filesystem or container runtime storage exceeds usage limits (typically > 85%). Clean space or check with `df -h`.
* PIDPressure: Active Linux processes exceed the allowed maximum, blocking new process creation. Check usage with `ps -eLf | wc -l`.
* NetworkUnavailable: The node cannot route pod-to-pod traffic, often due to a crashed CNI daemonset pod (e.g., Calico, Flannel, Cilium). Check via `kubectl get pods -n kube-system`.

## Hardware & Infrastructure Failures
Underlying cloud infrastructure or bare-metal hardware breakdowns that degrade node stability.

* Kernel OOM Killing: The Linux kernel runs completely out of memory and kills critical processes like Kubelet or Containerd to save the OS. Spot this using `dmesg -T | grep -i oom`. 
* Kernel Panics / Freezes: Unrecoverable OS kernel errors completely lock up the machine, stopping all telemetry.
* AWS/GCP/Azure Maintenance: Cloud providers performing scheduled underlying physical host migrations or experiencing infrastructure outages.
* Disk I/O Throttling: Cloud volumes (like AWS EBS GP2/GP3) exhausting their burst IOPS balance, grinding all disk read/write operations to a halt. Verify with `iostat -xz 1`.

## Configuration & State Mismatches
Soft errors caused by human configuration mistakes or mismatched node properties.

* Insufficient Resource Allocations: High cluster workload demands but the node lacks allocated space, keeping pods stuck in Pending.
* Kubelet Clock Drift: Time synchronization breaks between nodes and the control plane, causing security token rejections. Fix with `timedatectl status`.
* Mismatched Architecture / OS: Attempting to schedule an `arm64` container image onto an `amd64` node, or running Windows containers on Linux hosts.
* Taint / Toleration Mismatches: Nodes are intentionally cordoned or tainted (e.g., `Dedicated=NoSchedule`), but incoming workloads lack the required tolerations to land on them.
