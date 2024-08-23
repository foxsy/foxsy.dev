---
title: 'Flush DNS cache in macOS using the Terminal'
pubDate: 2024-08-31
description: 'Flush your Macbook macOS DNS cache via the Terminal'
author: 'Foxsy'
image:
    url: 'https://foxsy.dev/blog-images/images/shell-script-icon.webp'
    alt: 'Shell script terminal logo'
heroImage: 'https://foxsy.dev/blog-images/images/shell-script-icon.webp'
tags: ["macos", "dns", "cache", "flush", "terminal", "shell"]
---

## Run these three commands to flush your DNS cache from your macOS Terminal
```
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
```

## Make it easier by putting a shell script in your $PATH
* Create a file `flushdns` somewhere in your $PATH (in my case ~/bin was added to my path)

* Add the following and save the file
```
#!/bin/bash
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
```
* Make it executable with `chmod +x flushdns`

Now you can just execute `flushdns` anywhere in your Terminal
