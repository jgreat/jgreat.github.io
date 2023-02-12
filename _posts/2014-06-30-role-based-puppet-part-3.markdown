---
layout: post
title:  "Role based Puppet/MCollective in EC2 -  Part 3 Installing MCollective with SSL"
date:   2014-06-11 13:33:17 -0600
categories: puppet ec2 aws
---

This post will take you through adding MCollective orchestration software to the puppetmaster you just finished building. This howto includes configuring MCollective over SSL. This builds on the puppetmaster you built in <a href="http://jgreat.me/?p=32" title="Role based Puppet/MCollective in EC2 –  Part 2 Installing puppetmaster">Part 2</a>. Ubuntu 14.04 running on EC2. On a side note, if you have the budget, I highly suggest supporting Puppet Labs and their efforts by subscribing to Puppet Enterprise. Of course if you did, you wouldn't need these instructions :)

## About MCollective.

MCollective is orchestration software by Puppet Labs. It allows you to connect to and run "jobs" simultaneously across groups of instances. These groups of instances can be manually defined or selected based on something like the Facter facts we defined in the `user-data` in <a href="http://jgreat.me/?p=27" title="Role based Puppet/MCollective in EC2 –  Part 1 Roles">Part 1</a> MCollective consists of 3 basic components:


- Middleware: This is really the "server" portion. ActiveMQ messaging software runs here. I run this on my puppetmaster.
- Agent: `mcollectived` runs on all the instances you want to control with mcollective.
- Client: `mco` command. This is where you control the agents. I think of this part as the command console. I also install this on the puppetmaster.


**ActiveMQ**

Apache ActiveMQ is really the "server" portion of MCollective. You can use other messaging servers, but that's way beyond the scope of what is covered here. It's written in Java.

- Listening Port: 61614
- Start, Stop, Restart: `service activemq [action]`
- Configuration: `/etc/activemq/instances-enabled/mcollective/`
- Logs: Not setup by default. <a href="http://activemq.apache.org/how-can-i-enable-detailed-logging.html">ActiveMQ Logging</a>

**mcollectived**

mcollectived is the "agent" portion. It connects to the ActiveMQ server, then the server can pass messages back. It's written in Ruby.

- Start, Stop, Restart: `service mcollective [action]`
- Configuration: `/etc/mcollective`
- Logs: `/var/log/mcollective.log`

**mco - the "client"**

MCollective calls the places where you run the `mco` command the "client". I think this is terribly confusing, it's too close to "agent". I generally install `mco` on my `puppetmaster` instances. Be careful who you give access to the `mco` command - it can be the equivalent of giving root access to all your instances. Mistakes here can affect all of your instances at once. It is possible to lock down what a user can do with this command, but that's beyond the scope of this tutorial. This is also written in Ruby.

## Installing MCollective.

I find the Puppet Labs documentation extremely confusing and out of date. But there is hope! I just about had it all figured out when I discovered that you can do the full installation of middleware/agent/client with SSL though the `puppetlabs/mcollective` Puppet module. Using the module _almost_ makes the install easy.

### Install `puppetlabs/mcollective` Puppet module.

Install the module with the `puppet` command. It will install a bunch of dependencies.

```bash
puppet module install puppetlabs/mcollective
```

### Install my `site_mcollectve` Module.

I wrote a module that wraps the `puppetlabs/mcollective` module. Mostly the `site_mcollective` module gives you a place keep the certs and define your users so Puppet can install them.

**Clone the Repo into `/etc/puppet/modules`.**

I put the module up on GitHub - <a href="https://github.com/jgreat/site_mcollective">https://github.com/jgreat/site_mcollective</a>

```bash
cd /etc/puppet/modules
git clone https://github.com/jgreat/site_mcollective.git
```

**Setup Config File.**

All the configuration is done in a yaml file. This will require Hiera to work. Create a folder to store yaml configs for Hiera and copy the `site_mcollective.yaml` to it.

```bash
mkdir /etc/puppet/yaml
cp /etc/puppet/modules/site_mcollective/site_mcollective.yaml /etc/puppet/yaml
```

**Modify/Create `/etc/puppet/hiera.yaml`.**

Include `site_mcollective` as a `:hierarchy:` source.

```yaml
---
:backends:
  - yaml
:yaml:
  :datadir: /etc/puppet/yaml
:hierarchy:
  - site_mcollective
  - common
```

**Restart `apache2`.**

You will need to restart the `puppetmaster` to read the `hirea.yaml`.

```
service apache2 restart
```

**Modify `/etc/puppet/yaml/site_mcollective.yaml`.**

Change the values to suit your site.

- `middleware_hosts:` a list of your puppetmaster/mcollective servers.
- `activemq_password:` Change it to something long and random.
- `activemq_admin_password:` Change it to something long and random.
- `ssl_server_public:` Change cert file name to your server name.
- `ssl_server_private:` Change key file name to your server name.
- `users:` a list of your users that can run mco. These users must already have accounts.

### Copy the Server Certificates and Keys.

MCollective can use the `puppetmaster` SSL certificate authority for its SSL certificates. We will copy the files from the `puppetmaster` directories into the `site_mcollective` module so they can be distributed to the appropriate places.

**`puppetmaster` CA Certificate.**

```bash
cp /var/lib/puppet.example.com/ssl/certs/ca.pem /etc/puppet/modules/site_mcollective/files/server/certs/
```

**`puppetmaster` Server Certificate and Key.**

The key is only readable by `root`, you will need to change the group ownership to `puppet` so the `puppetmaster` process can read it.

```bash
cp /var/lib/puppet.example.com/ssl/certs/puppet.example.com.pem /etc/puppet/modules/site_mcollective/files/server/certs/
cp /var/lib/puppet.example.com/ssl/private_keys/puppet.example.com.pem /etc/puppet/modules/site_mcollective/files/server/keys/
chgrp puppet /etc/puppet/modules/site_mcollective/files/server/keys/puppet.example.com.pem
```

### Adding Users.

We need to generate a SSL certificate and key for each of our users. Puppet has a really simple interface to do this. I'm using the `ubuntu` user as an example.

**Generate a Certificate and Key.**

```bash
puppet cert generate ubuntu
```

**Copy User Certificate and Key.**

Again the user keys are only readable by `root`, you will need to change the group ownership to `puppet` so the `puppetmaster` process can read it.

```bash
cp /var/lib/puppet.example.com/ssl/certs/ubuntu.pem /etc/puppet/modules/site_mcollective/files/user/certs/
cp /var/lib/puppet.example.com/ssl/private_keys/ubuntu.pem /etc/puppet/modules/site_mcollective/files/user/keys/
chgrp puppet /etc/puppet/modules/site_mcollective/files/user/keys/ubuntu.pem
```

### Install Middleware Server.

Now we are going to use puppet to install the middleware server, agent and `mco` on the `puppetmaster` instance.

**Create/Modify your `site.pp`.**

Assign the `site_mcollective` class to your `puppetmaster`. Instead of assigning classes to traditional `node` definitions, I'm using the `site_role` fact I created in the user-data when I built the instance. See <a href="http://jgreat.me/?p=27" title="Role based Puppet/MCollective in EC2 –  Part 1 Roles">Part 1</a> for details on setting up the roles with user data or simple shell script. The site_mcollective cass has 3 `install_type`'s available.

- `agent` - Default. Just the agent portion, install this on most instances.
- `client` - The `mco` software and the agent. Install this the instances you want to run commands from.
- `middleware` - The "server". ActiveMQ, agent and the client. I'm adding `site_mcollective` with - `install_type =&gt; 'middleware'` on my instances with the `puppetmaster` role.

```bash
case $::site_role {
    puppetmaster: {
        class { 'site_mcollective':
            install_type =&gt; 'middleware',
        }
    }
    default: {
    }
}
```

**Run `puppet agent`.**

Now run the puppet agent on your master instance. You should see a whole bunch of actions happen. Hopefully it's all green output.

```bash
puppet agent -t
```
**Test MCollective.**

Now login as the user you set up and run `mco ping`. If everything setup correctly you should see a list of instances running the agent.

```bash
ubuntu@puppet01:~$ mco ping
puppet01                                 time=61.73 ms


---- ping statistics ----
1 replies max: 61.73 min: 61.73 avg: 61.73
```

Hurrah it's working!

### Install on Additional Agents.

I'm adding `site_mcollective` on my instances with the `mrfancypantsapp` role. Since `install_type =&gt; 'agent'` is the default, you don't need to specify it.

```ruby
case $::site_role {
    puppetmaster: {
        class { 'site_mcollective':
            install_type =&gt; 'middleware',
        }
    }
    mrfancypantsapp: {
        class { 'site_mcollective': }
    }
    default: {
    }
}
```

**Run `puppet agent`.**

Wait for the automatic puppet agent run or trigger it manually.

```bash
puppet agent -t
```

**Test MCollective.**

Run `mco ping` from your puppetmaster. You should now see additional hosts.
```bash
ubuntu@puppet01:~$ mco ping
puppet01                                 time=61.73 ms
mrfancypantsapp-prd-ec2-111-111-111-111  time=75.33 ms
mrfancypantsapp-prd-ec2-111-111-111-112  time=80.31 ms
mrfancypantsapp-stg-ec2-111-111-111-231  time=63.34 ms

---- ping statistics ----
4 replies max: 80.31 min: 61.73 avg: 70.18
```

Now we can use the `site_role` fact to run commands on servers that are only in that role.

```bash
ubuntu@puppet01:~$ mco ping -F site_role=mrfancypantsapp
mrfancypantsapp-prd-ec2-111-111-111-111     time=75.33 ms
mrfancypantsapp-prd-ec2-111-111-111-112     time=80.31 ms
mrfancypantsapp-stg-ec2-111-111-111-231     time=63.34 ms

---- ping statistics ----
3 replies max: 80.31 min: 63.34 avg: 72.99
```

### Install Client (mco).

I'm adding `site_mcollective` `install_type =&gt; 'client'` on my instances with the `secure_jumppoint` role. From these instances I can run the mco command to send commands to other servers.

```ruby
case $::site_role {
    puppetmaster: {
        class { 'site_mcollective':
            install_type =&gt; 'middleware',
        }
    }
    mrfancypantsapp: {
        class { 'site_mcollective': }
    }
    secure_jumppoint: {
        class { 'site_mcollective':
            install_type =&gt; 'client',
        }
    }
    default: {
    }
}
```

**Run `puppet agent`.**

Wait for the automatic puppet agent run or trigger it manually.

```bash
puppet agent -t
```

Now you can log into the `secure_jumppoint` instance and run `mco` commands.
