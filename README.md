[![Bash](https://img.shields.io/badge/Bash-4%2B-4EAA25?logo=gnu-bash&logoColor=white)](https://www.gnu.org/software/bash/)
[![Prometheus](https://img.shields.io/badge/Prometheus-ready-E6522C?logo=prometheus&logoColor=white)](https://prometheus.io/)
[![Grafana](https://img.shields.io/badge/Grafana-dashboards-F46800?logo=grafana&logoColor=white)](https://grafana.com/)
[![Platform](https://img.shields.io/badge/platform-Linux-lightgrey)](https://www.kernel.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

# Linux System Monitoring

A guide to building a Linux monitoring loop from first principles — using only
`/proc` and standard command-line tools, with no agents and no dependencies.

## Table of contents

- [The loop](#the-loop)
- [Core commands](#core-commands)
- [What to watch](#what-to-watch)
- [Thresholds](#thresholds)
- [Going further](#going-further)
- [License](#license)

## The loop

Every N seconds:

1. **Collect** metrics from `/proc` and standard tools (`top`, `free`, `df`)
2. **Compare** them against thresholds
3. **Log** the metrics, and any alert, to a file
4. Optionally **print** a summary to the terminal

Keeping collection, evaluation, and reporting as separate steps is what makes
the loop testable and safe to leave running.

## Core commands

**CPU**
```bash
top -bn1 | grep Cpu          # one-shot CPU line
uptime                       # load average
nproc                        # core count, for load context
```

**Memory**
```bash
free -m                      # watch "available", not "free"
```

**Disk**
```bash
df -h                        # space
df -i                        # inodes
du -sh /var/log
```

**Processes**
```bash
ps -eo pid,comm,%cpu,%mem --sort=-%cpu | head -n 11
```

## What to watch

| Signal | Source | Why it matters |
|---|---|---|
| Load average | `uptime` | Sustained load above core count means work is queuing |
| CPU idle | `top` | Low idle with high `iowait` points at disk, not CPU |
| Memory available | `free` | `available`, not `free` — page cache is reclaimable |
| Disk usage | `df -h` | A full `/` or `/var` breaks logging and services |
| Inodes | `df -i` | Inodes can run out while space remains |

## Thresholds

Alert on **sustained** conditions, never a single sample — one spike is noise.

| Metric | Warning | Critical |
|---|---|---|
| Load (per core) | > 0.7 for 5 min | > 1.0 for 5 min |
| Memory available | < 20% | < 10% |
| Disk used | > 80% | > 90% |
| Inodes used | > 80% | > 90% |

## Going further

- `sysstat` / `mpstat` for accurate per-core CPU
- Prometheus + `node_exporter` + Grafana for real dashboards and alerting
- Alertmanager for routing alerts to Slack, email, or PagerDuty

## License

[MIT](LICENSE) © Joash Odingo
