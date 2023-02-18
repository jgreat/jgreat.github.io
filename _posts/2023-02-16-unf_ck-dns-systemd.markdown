---
layout: post
title:  "How to Unf*ck systemd DNS - Don't use it."
date:   2023-02-18 13:33:17 -0600
categories: systemd-sucks vpn dns
---

Everytime I upgrade my Ubuntu based OS, I think _maybe this is the year that systemd-resoved just works_ and every year its just a big bag of disappointment. I spend hours troubleshooting why my VPN DNS resolution has just stopped working. Maybe next time I'll remember I wrote this post.

## How to disable `systemd-resolved`

```
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved

sudo rm /etc/resolv.conf

# Then put the following lines in the [main] section of your /etc/NetworkManager/NetworkManager.conf:
dns=default
rc-manager=file

# Ubuntu <= 20.04
sudo systemctl restart network-manager

# Ubuntu >= 20.10
sudo systemctl restart NetworkManager.service
```

That's it.  Now we're back to regular old `/etc/resolv.conf` based DNS configuration.
