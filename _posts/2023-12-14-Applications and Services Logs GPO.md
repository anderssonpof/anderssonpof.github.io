---
layout: post
title: "Applications and Services Logs GPO"
date: 2023-12-14
categories: WINDOWS WINDOWS_SERVER EVENT EVENT_VIEWER EVENT_LOGS
---

# Enable Applications and Services Logs with GPO

There is not really a built-in way to enable these logs. However you can enable them via registry.

The settings are available under:
```
Computer\HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\WINEVT\Channels\
```

Find the log you are interested in and change the enabled value to 1 to enable the log.
