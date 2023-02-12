---
layout: post
title:  "Role based Puppet/MCollective in EC2 -  Part 2 Installing puppetmaster"
date:   2014-06-11 13:33:17 -0600
categories: puppet ec2 aws
---

This how to will show you how to build a "production ready" puppetmaster in EC2 based on an Ubuntu 14.04 AMI.

## Prerequisites

It will help a lot if you already have some EC2 experience. I'm not going to explain all the ins and outs of the EC2 pieces, just what you will need to set up. This howto also assumes that you will be working in a VPC (since that's the current default).

## Set up Security Groups
In a traditional infrastructure you would need to approve any new agents that connect to the puppetmaster. In our case we want to be able to take advantage of autoscaling, so new instances may need access at anytime without manual approval. We will allow instances to automatically register and control access to the puppetmaster through security groups. Two groups will need to be created:


- puppet-agent - This group will be assigned to all the hosts that run the puppet agent. This group does not need any inbound ports configured.
- puppet - This group will be assigned to our puppetmaster and have permissions to allow the puppet-agent security group to access the system. This group should have inbound TCP ports 8140 and 61613 open to the security group id of the puppet-agent group.

### Create the puppetmaster instance

You can get the current Ubuntu 14.04 AMI here:

<a title="https://cloud-images.ubuntu.com/locator/ec2/" href="https://cloud-images.ubuntu.com/locator/ec2/" target="_blank">https://cloud-images.ubuntu.com/locator/ec2/</a>


- Click the AMI-xxxxxx link for 14.04 with EBS storage.
- Pick the m3.large size - m3.medium did not have enough memory to run MCollective.
- Up root storage to at least 15GB.
- Assign the `puppet` and `puppet-agent` security groups (don't forget to assign a default group with ssh access).
- Launch and pick your ssh key.

### Naming the System

For portability and scaling you will want to name your system something like `puppet01` and create a cname called `puppet` that points to `puppet01`. This way if you need to have more puppetmaster servers or a need a new one you can point the puppet.example.com cname to a load-balancer or another host.

- Once the instance has started, assign the instance an Elastic IP.
- Register a cname like puppet01.example.com in DNS to the ec2-xxx-xxx-xxx-xxx.compute-1.amazonaws.com public name for the Elastic IP.
- Register a cname puppet.example.com in DNS to puppet01.example.com.

**/etc/hosts**

Run `ec2metadata --local-ipv4` to find the local IP address. As root edit `/etc/hosts` and add your local address with the names you have registered in DNS.

```bash
127.0.0.1 localhost

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts

10.0.1.192 puppet01.example.com puppet01
10.0.1.192 puppet.example.com puppet
```

**/etc/hostname**

Put `puppet01` in `/etc/hostname`

```bash
puppet01
```

Reboot so the instance will start up with the new hostname. If this is all successful you should be greeted with a prompt that reads: `ubuntu@puppet01:~$`

### Install the PuppetLabs Apt Repo

The standard Ubuntu repos have a version of Puppet in them, but it's usually a bit dated. Install the PuppetLabs repo with their handy .deb package, to get the current version of Puppet.

```bash
wget http://apt.puppetlabs.com/puppetlabs-release-trusty.deb
sudo dpkg -i ./puppetlabs-release-trusty.deb
```

### Configure puppetmaster

We are going to configure puppetmaster before we install it. This is because the the `puppetmaster-passenger` package will use the settings in the master section to configure the Apache vhost for you.

**/etc/puppet/puppet.conf**

Create the directory

```
sudo mkdir -p /etc/puppet
```

We are going to set the `master` section to use `puppet.example.com` for its `certname` and `vardir`. This will keep the master files separate from the agent files make it easier to copy the ssl certs to new instances if you have to scale up.

**As root create the `/etc/puppet/puppet.conf` file.**
```ini
[main]
pluginsync = true

[master]
# These are needed when the puppetmaster is run by passenger
# and can safely be removed if webrick is used.
ssl_client_header = SSL_CLIENT_S_DN
ssl_client_verify_header = SSL_CLIENT_VERIFY
certname = puppet.example.com
vardir = /var/lib/puppet.example.com
ssldir=/var/lib/puppet.example.com/ssl

[agent]
server = puppet.example.com
vardir=/var/lib/puppet
ssldir=/var/lib/puppet/ssl
rundir=/var/run/puppet
```

**/etc/puppet/autosign.conf**

By default a new Puppet agent that connects to the master will need to manually have its SSL cert signed on the master. This isn't really practical for an autoscaling infrastructure. Instead we have the master automatically sign all the certs and control access through EC2 Security Groups. As root create a `/etc/puppet/autosign.conf` file.

```
*
```

<h5>Install the Packages.</h5>

Here's the real trick PuppetLabs doesn't tell you. You don't want to install the `puppetmaster` package, that installs the Ruby Webrick webserver that can only handle a handful of clients. You want to install the `puppetmaster-passenger` package. This package installs and configures Apache with Ruby Passenger so you can handle a "Production" load. No fussing with gems, compiling passenger or configuring vhosts, it's all done for you. As for scale I'm running 50-75 instances on one m3.large with no issues. I suspect that I could do 200 instances before I need to look at scaling up.

```bash
sudo apt-get update
sudo apt-get install puppetmaster-passenger puppet
```

**About the puppetmaster Service**

The puppetmaster is Ruby Passenger process that runs through the apache2 service. Here's the general layout of things:

- Stop, start or restart: `service apache2 [action]`
- puppetmaster config, manifests and modules: `/etc/puppet`
- puppetmaster working files: `/var/lib/puppet.example.com`
- Apache2 puppetmaster vhost: `/etc/apache2/sites-available/puppetmaster.conf`
- Access logs: `/var/log/apache2/other_vhosts_access.log`
- General puppetmaster logs: syslog `/var/log/syslog`

### Test out the agent

At this point you should be able to run the Puppet agent locally on the master instance. Since there's no configuration no actual actions will be taken. Remove the agent lock file.

```bash
sudo rm /var/lib/puppet/state/agent_disabled.lock
```

Enable the Puppet agent service in /etc/default/puppet.

```bash
sudo sed -i /etc/default/puppet -e 's/START=no/START=yes/'
```

Run the Puppet agent with the `-t` flag so it will show output.

```bash
sudo puppet agent -t
```

You should output similar to this.

The Puppet agent color codes its output so it should be all green. If you see red something has gone wrong.

```
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Caching catalog for puppet01.example.com
Info: Applying configuration version '1402079876'
Info: Creating state file /var/lib/puppet/state/state.yaml
Notice: Finished catalog run in 0.01 seconds
```

Congratulations you now have a functional puppetmaster server. Next we will use the puppet to install MCollective with SSL support.
