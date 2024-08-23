---
title: 'Change macOS computer name using Terminal'
pubDate: 2024-08-29
description: 'Change your Macbook macOS computer name via the Terminal'
author: 'Foxsy'
image:
    url: 'https://foxsy.dev/blog-images/images/shell-script-icon.webp'
    alt: 'Shell script terminal logo'
heroImage: 'https://foxsy.dev/blog-images/images/shell-script-icon.webp'
tags: ["macos", "computername", "macbook", "terminal", "shell"]
---

## Run these three commands to change the computer name that your terminal displays
```
sudo scutil --set ComputerName "mycoolmacbook"
sudo scutil --set LocalHostName "mycoolmacbook"
sudo scutil --set HostName "mycoolmacbook"
```

## Flush the DNS cache by typing:
```
dscacheutil -flushcache
```

## Restart your Mac.
It didn't work for me, had to restart my mac for new name to show.
