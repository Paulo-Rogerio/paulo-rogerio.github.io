---
layout: post
collection: linux
permalink: /linux/at
title:  "Linux Command At"
author: Paulo Rogério
date:   2025-01-03 11:27:00 -0300
categories: [linux]
published: true
---

# Linux Command At

- [1) Running At By Systemd](#1-running-at-by-systemd)
  - [1.1) On-Active](#11-on-active)
  - [1.2) On-Calendar](#12-on-calendar)
  - [1.3) List Cron](#13-list-cron)

## 1) Running At By Systemd

Replace command at by systemd-run


# 1.1) On-Active

```bash
root@study:~# chmod +x /opt/scripts/paulo.sh
root@study:~# systemd-run --on-active="2m" /opt/scripts/paulo.sh
Running timer as unit: run-r91aa2dfd6635488d994ce8b609302511.timer
Will run service as unit: run-r91aa2dfd6635488d994ce8b609302511.service
```

# 1.2) On-Calendar

```bash
root@study:~# systemd-run --on-calendar="2024-12-17 11:25" /opt/scripts/paulo.sh
Running timer as unit: run-r91aa2dfd6635488d994ce8b609302511.timer
Will run service as unit: run-r91aa2dfd6635488d994ce8b609302511.service
```

# 1.3) List Cron

```bash
systemctl list-units --type=timer
```


