---
layout: post
title:  "Role based Puppet/MCollective in EC2 -  Part 1 Roles"
date:   2014-06-09 13:33:17 -0600
categories: puppet ec2 aws
---

This is the first in a short series of posts that will take you through building simple role based infrastructure in EC2 using Puppet for configuration and MCollective for orchestration.

### What do you mean "role based"?

When you are talking AWS EC2 or any other "cloud infrastructure" traditional ways of naming and organizing systems are just not flexible enough. When autoscaling, instances come and instances go. Your 4 web servers may, at any moment turn into 6 servers. When you scale back down it might not be the original 4. The EC2 assigned DNS like ec2-54-86-7-157.compute-1.amazonaws.com really don't tell you anything about what that server does. To solve this problem, I create a couple of simple definitions (in this case Puppet "facts") that I use to organize the systems in a human way. With tools like Puppet I can deploy my configuration and code to servers in various roles. MCollective allows me to run commands and manage the systems simultaneously by role. It doesn't matter if it's 4 servers or 100.

### A bit of background.

Puppet and MCollective use a program called Facter to generate "facts" about a system. Facter runs on the agent systems during the Puppet agent run. These facts end up as top level variables in Puppet. For example `$::hostname` is `facter hostname`.

### Assigning roles.

Now we need a way to assign roles to the instances when we boot them. To keep it simple, I'm using inline yaml in the user-data to create some basic "facts" at build time. Using yaml makes it easy just populate the user-data box in the AWS Management GUI, AWS CLI or for use in more advanced automation like CloudFormation. Here is what I include in the user-data:


- env: Instance environment - Examples: dev, qa, stg or prd
- role: Instance role - Examples: www, api, mongodb<

```yaml
{env: prd, role: mrfancypantsapp}
```

This simple python script reads the user-data (provided to the instance by an AWS magic url) and spits out `key=value` pairs for Facter so we can use these values in Puppet. I namespace these values with `site='mysite'` so they don't stomp on values set by other data sources. Just place this script in the

`/etc/facter/facts.d` directory and Facter will process it on each run. I include this script in our custom AMI builds so this works at boot.

```python
#!/usr/bin/env python
site = "site" # change this to your own site

import yaml
import subprocess

userData = subprocess.check_output('/usr/bin/ec2metadata --user-data', shell=True)
facts = yaml.load(userData)
for k in facts.iterkeys():
    print site + "_" + k + "=" + facts[k]
```

These facts will now be top level variables in Puppet. You can see them and other things Puppet/Facter thinks are important for identifying your system by running `facter` on the agent host. Using this simple yaml makes it easy to add more facts at build time.

### But wait, I have a bunch of systems already running.

You can't modify user-data on running instances, but all Facter wants is something that spits out key=value pairs. For legacy systems I use a simple shell script to echo values. Place this script in the `/etc/facter/facts.d` directory.

```bash
#!/bin/sh
echo site_env=prd
echo site_role=mrfancypantsapp
```
