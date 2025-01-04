---
layout: post
collection: linux
permalink: /linux/crontab-systemd
title:  "Linux Crontab By Systemd"
author: Paulo Rogério
date:   2025-01-03 11:27:00 -0300
categories: [linux]
published: true
---

# Crontab By Systemd

- [1) Running Crontab By Systemd](#1-running-crontab-by-systemd)
  - [1.1) Unit Service](#11-unit-service)
  - [1.2) Unit Timer](#12-unit-timer)
  - [1.3) Usage](#13-usage)  


## 1) Running Crontab By Systemd

Manager crontab by systemd

# 1.1) Unit Service

```bash
root@study:~# cat > /opt/scripts/paulo.sh <<EOF
#!/bin/bash
echo $(date) >> /tmp/paulo.txt
EOF
root@study:~# chmod +x /opt/scripts/paulo.sh
```

```bash
cat > /etc/systemd/system/paulo.service <<EOF
[Unit]
Description=Script Paulo Test

[Service]
Type=oneshot
ExecStart=/bin/bash /opt/scripts/paulo.sh
EOF
```

# 1.2) Unit Timer

```bash
root@study:~# cat > /etc/systemd/system/paulo.timer <<EOF
[Unit]
Description=Run script paulao

[Timer]
OnCalendar=*-*-* *:00/01:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

root@study:~# systemctl daemon-reload
root@study:~# systemctl start paulo.timer
root@study:~# systemctl status paulo.timer
root@study:~# systemctl enable paulo.timer
root@study:~# systemctl stop paulo.timer
```

# 1.3) Usage

```bash
# Options Usage
# hourly    *-*-* *:00:00
# daily     *-*-* 00:00:00
# weekly    Mon *-*-* 00:00:00
# mounthly  *-*-01 00:00:00
# yearly    *-01-01 00:00:00

# Exemples
# *-*-* 08:30:00             => All days at 08:30
# Sta,Sun *-*-* 05:00:00     => Saturday and Sunday at 05:00:00
# *-*-01 13:15,30,45:00      => At 13:15, 13:30, 13:45 of the first day of the month
# Fri *-09..12-* 16:20:00    => At 16:20 every friday the september, october, november and december
# Mon, Tue *-*-1,15 08:30:00 => At 08:30 of the first and fifteenth day of each month if day equal monday and tuesday
# *-*-* *:00/05:00           => Every 5 minutes 
```
