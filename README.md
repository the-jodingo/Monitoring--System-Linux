# Linux System Monitoring

Notes and reference material for building a Linux monitoring loop from first
principles, using only `/proc` and standard command-line tools.

## The loop

Every N seconds:

1. **Collect** metrics from `/proc` and standard tools (`top`, `free`, `df`)
2. **Compare** them against thresholds
3. **Log** the metrics and any alerts to a file
4. Optionally **print** a summary to the terminal

## Core commands

**CPU**
```bash
top -bn1 | grep Cpu
uptime                      # load average
```

**Memory**
```bash
free -m
```

**Disk**
```bash
df -h
du -sh /var/log
```

**Processes**
```bash
ps -eo pid,comm,%cpu,%mem --sort=-%cpu | head -n 11
```

## What to watch

| Signal | Where | Why it matters |
|---|---|---|
| Load average | `uptime` | Sustained > core count means queuing |
| CPU idle | `top` | Low idle with high iowait points at disk, not CPU |
| Memory available | `free` | `available`, not `free` — page cache is reclaimable |
| Disk usage | `df` | Full `/` or `/var` breaks logging and services |
| Inodes | `df -i` | Inodes can exhaust while space remains |

## Going further

- `sysstat` / `mpstat` for accurate per-core CPU
- Prometheus + `node_exporter` + Grafana for real dashboards and alerting
- Alert on **sustained** conditions, not single samples

## License

MIT
