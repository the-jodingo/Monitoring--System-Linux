Architechture Overview 
Think of this as a loop that every N nummber of seconds:

Collects metrics from /proc and standard commands (top/free/df)
Checks them against thresholds
Logs metrics and any alerts to a log file
Optionally prints a summary to the terminal

Core Linux commands : Mostly Used
When we are loading the disk or cpu - 
:$ top -bn1 | grep Cpu
:$ uptime

Memory:
free -m (or free)

Disk:
df -H or df -k

Network (optional):
/proc/net/dev 
ifconfig/ip

Step 1 – Basic project structure
On your Linux machine (ideally a VM or test server):

Create a project directory:
--
mkdir -p ~/bash-monitor
cd ~/bash-monitor

Files you’ll have:
system_monitor.sh – main script
system_monitor.conf – optional config (thresholds, interval)
monitor.log – local log (or use /var/log)

You can start without config and hardcode thresholds at the top of the script.

Step 2 – Write the main monitoring script
Below is a complete, ready-to-use example script.
Adjust thresholds and paths as needed. 

Create: system_monitor.sh
---
#!/usr/bin/env bash
#
# Simple Linux System Monitor - Bash DevOps Project
# Monitors CPU, memory, swap, disk, load average and logs to a file.
# Supports configurable thresholds for alerting.
#

# ------------------------------
# Configuration
# ------------------------------

INTERVAL_SEC=${INTERVAL_SEC:-5}         # How often to collect metrics
LOG_FILE=${LOG_FILE:-/var/log/system_monitor.log}
RUN_AS_DAEMON=${RUN_AS_DAEMON:-false}   # If true, loop forever; if false, run once

# Thresholds (% for CPU/memory/disk)
CPU_WARN_THRESHOLD=80
MEM_WARN_THRESHOLD=80
SWAP_WARN_THRESHOLD=50
DISK_WARN_THRESHOLD=85
LOAD_WARN_THRESHOLD=$(nproc)           # Alert if 1min load >= # cores

# ------------------------------
# Logging function
# ------------------------------

log() {
  local msg="$1"
  local severity="${2:-INFO}"  # INFO, WARN, ALERT

  echo "$(date '+%Y-%m-%d %H:%M:%S') [$severity] $msg" | tee -a "$LOG_FILE"
}

# ------------------------------
# Metric collection functions
# ------------------------------

# CPU usage (overall, all cores)
get_cpu_usage() {
  # Parse "Cpu(s):  1.2 us,  0.5 sy,  0.0 ni, 97.8 id, ..."
  # Many Bash monitoring examples use similar grep/sed/awk logic【turn0search2】【turn0search7】
  top -bn1 | grep -E '^%Cpu' | awk '{us=$2; sy=$4; print us+sy}'
}

get_load_average() {
  # Get 1-minute load average from uptime
  uptime | awk -F'load average:' '{print $2}' | awk -F',' '{print $1}' | tr -d ' '
}

get_process_count() {
  # Count processes in /proc (each PID directory)
  ls /proc 2>/dev/null | grep -E '^[0-9]+$' | wc -l
}

# Memory usage (%)
get_mem_usage() {
  # Many Bash monitor examples use free + awk for percentages【turn0search2】【turn0search11】
  free | awk '/^Mem:/ {
    total=$2
    used=$3
    printf "%.0f", (used/total)*100
  }'
}

get_swap_usage() {
  free | awk '/^Swap:/ {
    if ($2 == 0) { print 0; exit }
    printf "%.0f", ($3/$2)*100
  }'
}

# Disk usage (%)
get_disk_usage() {
  # Return: mount_point usage_percent
  df -H -x tmpfs -x devtmpfs | awk 'NR>1 {
    print $NF " " $5
  }'
}

# Network RX/TX (basic)
get_network_bytes() {
  local iface="${1:-eth0}"
  local stats_file="/sys/class/net/${iface}/statistics"

  if [[ -d "$stats_file" ]]; then
    local rx=$(< "$stats_file/rx_bytes")
    local tx=$(< "$stats_file/tx_bytes")
    echo "$iface $rx $tx"
  else
    echo "$iface 0 0"
  fi
}

# ------------------------------
# Alert helper
# ------------------------------

check_threshold() {
  local name="$1"
  local value="$2"
  local threshold="$3"
  local unit="${4:-%}"

  if (( $(echo "$value >= $threshold" | bc -l) )); then
    log "$name is $value$unit (threshold: $threshold$unit)" "ALERT"
    return 1  # 1 = threshold exceeded
  else
    log "$name is $value$unit (threshold: $threshold$unit)" "INFO"
    return 0
  fi
}

# ------------------------------
# Main collection loop
# ------------------------------

collect_and_log() {
  log "===== System metrics collection start ====="

  # CPU
  cpu_usage=$(get_cpu_usage)
  check_threshold "CPU usage" "$cpu_usage" "$CPU_WARN_THRESHOLD" "%"

  # Load average
  load_avg=$(get_load_average)
  if (( $(echo "$load_avg >= $LOAD_WARN_THRESHOLD" | bc -l) )); then
    log "Load 1m is $load_avg (threshold: $LOAD_WARN_THRESHOLD)" "ALERT"
  else
    log "Load 1m is $load_avg (threshold: $LOAD_WARN_THRESHOLD)" "INFO"
  fi

  # Process count
  proc_count=$(get_process_count)
  log "Process count: $proc_count" "INFO"

  # Memory
  mem_usage=$(get_mem_usage)
  check_threshold "Memory usage" "$mem_usage" "$MEM_WARN_THRESHOLD" "%"

  # Swap
  swap_usage=$(get_swap_usage)
  check_threshold "Swap usage" "$swap_usage" "$SWAP_WARN_THRESHOLD" "%"

  # Disk
  while read -r mount usage_str; do
    usage_percent="${usage_str%\%}"
    check_threshold "Disk usage on $mount" "$usage_percent" "$DISK_WARN_THRESHOLD" "%"
  done < <(get_disk_usage)

  # Network (optional)
  # Uncomment and replace eth0 with your actual interface if needed
  # read -r iface rx tx < <(get_network_bytes eth0)
  # log "Network $iface RX bytes: $rx, TX bytes: $tx" "INFO"

  log "===== System metrics collection end ====="
  echo ""
}

# ------------------------------
# Script entrypoint
# -----------------------------

main() {
  log "Starting system monitor with INTERVAL_SEC=$INTERVAL_SEC, LOG_FILE=$LOG_FILE"

  if [[ "$RUN_AS_DAEMON" == "true" ]]; then
    while true; do
      collect_and_log
      sleep "$INTERVAL_SEC"
    done
  else
    collect_and_log
  fi
}

main "$@"

Make it executable:

chmod +x system_monitor.sh

Step 3 – Test the script manually

1. Run a single collection (default behavior):
./system_monitor.sh

You should see output like:

2026-01-08 12:34:56 [INFO] Starting system monitor...
2026-01-08 12:34:56 [INFO] CPU usage is 12% (threshold: 80%)
2026-01-08 12:34:56 [INFO] Load 1m is 0.23 (threshold: 4)
2026-01-08 12:34:56 [INFO] Process count: 187
2026-01-08 12:34:56 [INFO] Memory usage is 45% (threshold: 80%)
2026-01-08 12:34:56 [INFO] Swap usage is 0% (threshold: 50%)
2026-01-08 12:34:56 [INFO] Disk usage on / is 30% (threshold: 85%)

2.Check the log:
sudo cat /var/log/system_monitor.log

If writing to /var/log is not allowed, change LOG_FILE in the script to a location your user can write, for example:

LOG_FILE="$HOME/bash-monitor/monitor.log"

Step 4 – Run it continuously / via cron
You have two modes:

a) One-shot via cron

Crontab example to run every 5 minutes:

crontab -e
*/5 * * * * /home/youruser/bash-monitor/system_monitor.sh
This will:

Collect one snapshot of metrics every 5 minutes
Append to LOG_FILE
b) Daemon-style (always running in background)

Set environment variable when calling:

RUN_AS_DAEMON=true INTERVAL_SEC=10 ./system_monitor.sh
This will:

Collect metrics every 10 seconds
Keep running until you stop it
You can run this as a systemd service later.

Step 5 – Add a simple configuration file (optional but nice)
If you want to make thresholds more “friendly”, use an external config file.

Create system_monitor.conf:
--
INTERVAL_SEC=10
RUN_AS_DAEMON=false
CPU_WARN_THRESHOLD=75
MEM_WARN_THRESHOLD=80
SWAP_WARN_THRESHOLD=40
DISK_WARN_THRESHOLD=80
LOAD_WARN_THRESHOLD=4

Then, at the top of system_monitor.sh (after the shebang), source it:
--
CONFIG_FILE="$(dirname "$0")/system_monitor.conf"

if [[ -f "$CONFIG_FILE" ]]; then
  # shellcheck source=/dev/null
  source "$CONFIG_FILE"
else
  echo "Config file $CONFIG_FILE not found, using defaults." >&2
fi

**Now you can change behavior by editing .conf instead of touching the main script.**

We can also eport metrics for Prometheus / Grafana
Many “Bash monitoring” tutorials log to a file. To look more “DevOps”, you can:

Output in Prometheus text format (simple key-value with HELP/TYPE):
Expose via a small HTTP server (using nc or Python) or write to a node_exporter textfile collector directory.

# HELP system_cpu_usage CPU usage percent
# TYPE system_cpu_usage gauge
system_cpu_usage 12.3
# HELP system_memory_usage Memory usage percent
# TYPE system_memory_usage gauge
system_memory_usage 45.1
# HELP system_disk_usage Disk usage percent per mount
# TYPE system_disk_usage gauge
system_disk_usage{mount="/"} 30.0

Prometheus can scrape these and you can build dashboards in Grafana.
Add systemd service

Create /etc/systemd/system/bash-monitor.service:
[Unit]
Description=Bash System Monitor
After=network.target

[Service]
Type=simple
User=root
EnvironmentFile=/path/to/bash-monitor/system_monitor.conf
ExecStart=/path/to/bash-monitor/system_monitor.sh
Restart=always

[Install]
WantedBy=multi-user.target
Then:

sudo systemctl daemon-reload
sudo systemctl enable --now bash-monitor
Now you have a real service managed by systemd.

Add more metrics
Some nice additions:

Per-interface network I/O and delta (RX/TX rates)
Disk I/O stats from /proc/diskstats or iostat

Top processes by CPU or memory (top or ps)
Check specific service status (systemctl is-active nginx)

5. Add a simple dashboard CLI
You could add a mode that:

Reads LOG_FILE
Shows last N events or only ALERT entries
Example : 
---
tail -n 100 /var/log/system_monitor.log | grep ALERT

Potential pitfalls and how to avoid them

Floating point in Bash:

Bash only handles integers; use bc or awk for comparisons with decimals.
Parsing commands that change format:

Commands like top, df output can vary by version/locale.

In production, test your parsing on the target OS, or use more stable sources (/proc, awk).
Permissions:

Writing to /var/log usually requires root.

Start by logging to your home directory for testing, then move to /var log with proper permissions.

Security: Don’t expose secrets in scripts (e.g., webhook URLs). Consider putting them in a restricted config file with proper permissions (chmod 600).


