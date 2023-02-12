---
layout: post
title:  "Install Puppet 4 (Open-source Version)"
date:   2016-02-21 13:33:17 -0600
categories: puppet linux
---

Puppet 4 Master with PuppetDB, and SSL.
### Puppet 4 Management TL;DR
Service:

- Start: `service puppetserver start`
- Stop: `service puppetserver stop`

Agent Run:

- `puppet agent -t`

Configs:

- `/etc/puppetlabs/puppet`

Manifests:

- `/etc/puppetlabs/code`

Logs:

- `/var/log/puppetlabs`

SSL Certs:

- `/etc/puppetlabs/puppet/ssl`

Ports:

- puppetserver: `8140`
- mcollective (not covered here): `61613`

### The System

We assume you already have a server. These instructions are geared to running on Ubuntu 14.04LTS. If you are using another OS, YMMV.

I'm running in Azure on a Standard_D2_v2 (2 core 7GB system). This is more then enough for puppet with some overhead for other admin tasks.

I suggest setting the puppet server to a static internal IP address to make it easier to bootstrap future clients.

### FQDN

Before you install anything, configure your system to know its Fully Qualified Domain Name. If the FQDN is correct puppet will create the correct SSL certs for you when you first start up the server.

You will want to create the system's name and also an alias for `puppet` and `puppetdb`. This will make moving or scaling the puppetserver service a lot easier in the future. It will also separate the puppet-agent service on the system from the puppetserver service.

- System FQDN: `a1admpuppet01.jgreat.me`
- Puppet Alias(CNAME): `puppet.jgreat.me`
- PuppetDB Alias(CNAME): `puppetdb.jgreat.me`

Set the system name using the special `127.0.1.1` IP in `/etc/hosts`. Set the alias names to use the system IP address.

`/etc/hosts`

```
127.0.1.1 a1admpuppet01.jgreat.me a1admpuppet01
10.0.1.50 puppet.jgreat.me puppet
10.0.1.50 puppetdb.jgreat.me puppetdb
```

`/etc/hostname`

Make sure `/etc/hostname` is set to the system's short name.

```
a1admpuppet01
```

Test this all out. `hostname -f` should now return FQDN

```
$ hostname -f
a1admpuppet01.jgreat.me
```

## Install Puppetserver

### Set up PuppetLabs Apt Repo

Get the puppetlabs-release package and install. This will setup the PuppetLabs Apt Repo for you

```
wget https://apt.puppetlabs.com/puppetlabs-release-pc1-$(lsb_release -sc).deb
dpkg -i puppetlabs-release-pc1-$(lsb_release -sc).deb
apt-get update
```

### Install puppetserver

Install the puppetserver and puppet-agent package, but don't start the server yet (doesn't start automatically)

```
apt-get install -y puppetserver puppet-agent
```

### Configure puppetserver

The puppetserver config has moved to `/etc/puppetlabs/puppet/puppet.conf`. The important bits are setting the `dns_alt_names` to the the `puppet` alias.

Note: `autosign = true` will set the server to automatically accept new clients and sign certs for them. You are relying on the network layer for security. Don't do something silly like publishing port 8140 to the internet.

`/etc/puppetlabs/puppet/puppet.conf`

```ini
[main]
server = puppet.jgreat.me
environment = production
runinterval = 30m

[master]
dns_alt_names = puppet,puppet.jgreat.me
environment_timeout = 0
autosign = true
vardir = /opt/puppetlabs/server/data/puppetserver
logdir = /var/log/puppetlabs/puppetserver
rundir = /var/run/puppetlabs/puppetserver
pidfile = /var/run/puppetlabs/puppetserver/puppetserver.pid
codedir = /etc/puppetlabs/code
```

### Start puppetserver

Start the puppetserver service

```
service puppetserver start
```

You can monitor the progress of the service.

```
tail -f /var/log/puppetlabs/puppetserver/puppetserver.log
```

### Test puppet-agent

Test your puppet agent (you may have to re-login to find `puppet` in the path)

```
# puppet agent -t
Info: Using configured environment 'production'
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Caching catalog for d1admpuppet01.jgreat.me
Info: Applying configuration version '1456088496'
Info: Creating state file /opt/puppetlabs/puppet/cache/state/state.yaml
Notice: Applied catalog in 0.03 seconds
```

## Install PuppetDB

The easy way to do this is to use puppet to install puppetdb.

Install the `puppetdb` puppet module.

```
puppet module install puppetlabs-puppetdb
```

Here is a sample manifest.

```
node 'a1admpuppet01.jgreat.me' {
  # Set service to start automatically
  service { 'puppetserver':
    ensure =&gt; running,
    enable =&gt; true,
  }

  # Install and configure puppetdb
  class { 'puppetdb': }
  class { 'puppetdb::master::config': }
}
```

That's it. You should now have a puppet server with puppetdb as a storage/reporting backend.
