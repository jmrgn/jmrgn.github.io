---
layout: post
title:  "Centralized Configuration Management with Consul"
date:   2014-12-15 20:15:00
categories: consul, configuration, configuration management
---

Link: [url](description)
Image: ![Chocolatey goodness](/img/choco-install-git.png)

## The Problem

Many modern applications tend to be very configuration heavy. This fact is compounded by modern approaches toward software development - a developer may have a local instance running on his or her own machine or server, a build may deploy to a variety of environments (development, integration) depending on the stability of a change, and obviously there must be a production instance of an application. Each of these environments likely requires its own set of configuration settings. Wise security policy dictates that things like credentials, connection strings, hostnames, etc. should vary between environments. Even things like logging levels are likely to change as an application moves through the development pipeline. Consequently, it is important to make sure that any approach toward configuration management should be clear and intuitive.

Even within my own team, we find ourselves maintaining a suite of applications implemented in a number of differing technologies. The result of this is an equally varrying number of approaches toward managing configuration values, as each technology stack (or a developer constrained by said technology) has often tried to solve the problem in its own unique way. It can be both difficult and error-prone to keep track of the remarkably different techniques toward a relatively simple problem and we have suffered the consequences in the form of misconfigured deployments that require manual analysis and intervention. Even adding in the artificial constraint of considering only applications written within the .NET technology stack doesn't make the solution any easier, as different versions of Visual Studio and MSbuild have offered their own take on the subject. More mature (read: old) applications might have a completely separate configuration file per environment. A more modern approach involves the introduction of environment-specific configuration transforms to ensure that static, insensitive config settings can be replaced in a single file at compile time based on a given configuration parameter. I actually find this very convenient, but it still doesn't help with the introduction of sensitive data, and changing the same value (say, an external service enpoint shared across several applications) can still require making the same change in multiple places.

So what would I prefer? A configuration solution meeting the following criteria:

 * Able to be used across multiple, disparate technology stacks.
 * Able to encapsulate envionrment specific data
 * Able to store sensitive information
 * Able to be utilized regardless of environment

These criteria all-but scream HTTP API. However, the mechanism by which a client adds or retrieves configuraiton data isn't the only factor at play. Ideally, the configuration manager would be fast, scalable, and provide some mechanism for redundancy. Fortunately, I didn't have too look for long before coming across something promising. A colleague pointed me toward [http://consul.io](Consul), made at least in part by a few of the same people responsible for the terribly useful environment scaffolding tool (https://www.vagrantup.com/)[Vagrant]. This is already looking promising.

### What is Consul?

They do a much better job explaining things on their own site, but in short it is a tool for managing different services in a given environment. This includes:

* Service discovery
 * Register a service through Consul, and clients can discover said service through DNS or plain HTTP
* Health Checks
 * Used to determine the status of a clustered service, individual node, etc.
* Multi-DC
 * Support for multiple datacenters out-of-box.
* Key/Value Store
 * A heirarchical key/value store, exposed via HTTP. Suggested uses: dynamic configuration, feature flagging, even leader election of services.

The last bullet is obviously the subject of this post, but each could easily warrant its own. After perusing the documentation for awhile I was ready to get started.

### The Setup

Unsurprisingly, it didn't take long for to find a sample Vagrant file that could easily be modified into a simple (https://github.com/jmrgn/config-consul)[demo]. For the sake of simplicty, I'll only be working with one consul agent working in server mode rather than using a cluster.

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :
$script = <<SCRIPT

echo Installing dependencies...
sudo apt-get install -y unzip curl

echo Fetching Consul...
cd /tmp/
wget https://dl.bintray.com/mitchellh/consul/0.4.1_linux_amd64.zip -O consul.zip

echo Installing Consul...
unzip consul.zip
sudo chmod +x consul
sudo mv consul /usr/bin/consul

SCRIPT

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # All Vagrant configuration is done here. The most common configuration
  # options are documented and commented below. For a complete reference,
  # please see the online documentation at vagrantup.com.

  # Every Vagrant virtual environment requires a box to build off of.

  config.vm.define "consul1" do |consul1|
    consul1.vm.provision "shell", inline: $script
    consul1.vm.box = "hashicorp/precise64"
    consul1.vm.hostname = "consul1"
    consul1.vm.network "private_network", ip: "172.20.20.91"
  end
end
```

```powershell
C:\projects\config-consul [master]> vagrant up
C:\projects\config-consul [master]> vagrant ssh consul1
```

And we're off to the races. I've successfully spun up a VM, installed consul, registered it against the above IP and made it accessible via the **consul1** hostname. Starting up a Consul agent in server mode is as easy as a simple command:


```
vagrant@consul1:~$ consul agent -server -bootstrap-expect 1 -data-dir /tmp/consul
```

Now that the basics are taken care of, it's time to move on to consul's configuration management system.


### The Key/Value Store




### Conclusions

