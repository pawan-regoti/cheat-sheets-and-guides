To get system stats in Linux, you can use built-in or terminal tools depending on whether you need real-time performance monitoring or hardware specifications.
## Real-Time Performance Stats

* `htop`: Provides an interactive, color-coded, real-time overview of CPU, memory, and running processes. (Run sudo apt install htop if it is not installed).
* `top`: The universal built-in utility for tracking active processes and live CPU/RAM usage.
* `vmstat 2`: Displays system virtual memory, disk I/O, swap, and CPU activity, updating every two seconds.
* `iostat -z 2`: Shows detailed storage drive throughput, utilization rates, and CPU metrics. 

## Hardware & Allocation Specs

* `free -h`: Displays total, used, and available RAM and Swap space in human-readable gigabytes or megabytes.
* `df -h`: Shows disk space allocation, usage percentages, and remaining capacity for all mounted filesystems.
* `lscpu`: Lists detailed technical architecture information about your CPU, including cores, threads, and clock speed.
* `sudo lshw -short`: Generates a compact summary profile of all physical hardware paths and devices.
* `uname -a`: Prints your current Linux kernel version, system architecture, and hostname. 

## Network & Connectivity Failures

* `ping -c 4 8.8.8.8`: Verifies basic outbound internet connectivity.
* `curl -I https://google.com`: Tests HTTP layer connectivity and outbound DNS resolution.
* `nc -zv <IP> <PORT>`: Checks if a specific remote port is open and reachable.
* `ss -tulwn`: Lists all local ports currently listening for traffic on the server.
* `ip route show`: Displays the local routing table to diagnose Gateway or VPC issues.
* `nslookup <domain>`: Troubleshoots internal or external DNS resolution failures. 

## Storage & Disk Space Issues
When logs or temporary files fill up your disk, applications will crash or refuse to start.

* `df -h`: Shows overall disk space usage per partition.
* `df -i`: Checks inode usage (too many tiny files can exhaust inodes even if disk space is free).
* `du -h --max-depth=1 /var/log`: Finds the largest directories inside a specific path.
* `sudo lsof +L1`: Lists deleted files that are still held open by running processes (releasing this space requires restarting the process).
* `> /var/log/app.log`: Clears a massive log file safely without breaking the application pointer.

## High CPU & RAM Bottlenecks
Cloud instances can slow down dramatically due to memory leaks or runaway background threads.

* `htop or top`: Identifies the exact processes consuming the most CPU or RAM.
* `free -h`: Displays total, used, free, and available physical memory and swap.
* `ps aux --sort=-%mem | head -n 10`: Lists the top 10 memory-hogging processes.
* `kill -9 <PID>`: Force-terminates a completely frozen or unresponsive process.
* `sysctl vm.overcommit_memory`: Checks memory allocation policies to prevent Out-Of-Memory (OOM) crashes.

## Service & Application Crashes

* `systemctl status <service_name>`: Checks if a system service is active, errored, or dead.
* `systemctl restart <service_name>`: Gracefully restarts a failed system service.
* `journalctl -eu <service_name> --no-pager`: Shows the most recent error logs for a specific service.
* `journalctl -k -p err`: Filters the system logs to show only hardware or kernel errors.
* `dmesg -T | grep -i oom`: Checks if the Linux kernel killed a process due to insufficient memory. 

## Cloud-Specific Metadata & Permissions

* `curl http://169.254.169`: Fetches AWS EC2 instance metadata (IAM roles, network tokens).
* `curl -H "Metadata-Flavor: Google" http://google.internal`: Fetches GCP metadata.
* `curl -H Metadata:true --noproxy "*" "http://169.254.169"`: Fetches Azure metadata.
* `chmod 400 key.pem`: Fixes SSH connection rejections caused by overly permissive private keys. 
