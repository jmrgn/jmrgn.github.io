---
layout: post
tags: consul configuration configuration_management
title:  "Centralized Configuration Management with Consul pt. 1"
date:   2014-12-15 20:15:00
---

Managing configuration is a conceptually simple problem that developers tend to overcomplicate. Compounding the problem is the fact that many of us maintain substantial codebases, often times implementing their own disparate configuration management schemes - some customized and home grown, others utilizing techniques and tools offered by development platforms themselves (e.g. .NET's build configuration-based Config Transforms).

## The Problem

Many modern applications tend to be very configuration heavy. This fact is compounded by modern approaches toward software development - a developer may have a local instance running on his or her own machine or server, a build may deploy to a variety of environments (development, integration) depending on the stability of a change, and obviously there must be a production instance of an application. Each of these environments likely requires its own set of configuration settings. Wise security policy dictates that things like credentials, connection strings, hostnames, etc. should vary between environments. Even things like logging levels are likely to change as an application moves through the development pipeline. Consequently, it is important to make sure that any approach toward configuration management should be clear and intuitive.

Even within my own team, we find ourselves maintaining a suite of applications implemented in a number of differing technologies. The result of this is an equally varying number of approaches toward managing configuration values, as each technology stack (or a developer constrained by said technology) has often tried to solve the problem in its own unique way. It can be both difficult and error-prone to keep track of the remarkably different techniques toward a relatively simple problem and we have suffered the consequences in the form of misconfigured deployments that require manual analysis and intervention. Even adding in the artificial constraint of considering only applications written within the .NET technology stack doesn't make the solution any easier, as different versions of Visual Studio and MSbuild have offered their own take on the subject. More mature (read: old) applications might have a completely separate configuration file per environment. A more modern approach involves the introduction of environment-specific configuration transforms to ensure that static, insensitive config settings can be replaced in a single file at compile time based on a given configuration parameter. I actually find this very convenient, but it still doesn't help with the introduction of sensitive data, and changing the same value (say, an external service endpoint shared across several applications) can still require making the same change in multiple places.

So what would I prefer? A configuration solution meeting the following criteria:

 * Able to be used across multiple, disparate technology stacks.
 * Able to encapsulate environment specific data
 * Able to store sensitive information (secure)
 * Able to be utilized regardless of environment

These criteria all-but scream HTTP API. However, the mechanism by which a client adds or retrieves configuration data isn't the only factor at play. Ideally, the configuration manager would be fast, scalable, and provide some mechanism for redundancy. Fortunately, I didn't have too look for long before coming across something promising. A colleague pointed me toward [Consul](http://consul.io), made at least in part by a few of the same people responsible for the terribly useful environment scaffolding tool [Vagrant](https://www.vagrantup.com/). This is already looking promising.

### What is Consul?

They do a much better job explaining things on their own site, but in short it is a tool for managing different services in a given environment. This includes:

* Service discovery
 * Register a service through Consul, and clients can discover said service through DNS or plain HTTP
* Health Checks
 * Used to determine the status of a clustered service, individual node, etc.
* Multi-DC
 * Support for multiple datacenters out-of-box.
* Key/Value Store
 * A hierarchical key/value store, exposed via HTTP. Suggested uses: dynamic configuration, feature flagging, even leader election of services.

The last bullet is obviously the subject of this post, but each could easily warrant its own. After perusing the documentation for awhile I was ready to get started.

### The Setup

Unsurprisingly, it didn't take long for to find a sample Vagrant file that could easily be modified into a simple [demo](https://github.com/jmrgn/config-consul). For the sake of simplicity, I'll only be working with one consul agent working in server mode rather than using a cluster.

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

```
C:\projects\config-consul [master]> vagrant up
C:\projects\config-consul [master]> vagrant ssh consul1
```

And we're off to the races. I've successfully spun up a VM, installed consul, registered it against the above IP and made it accessible via the **consul1** hostname. Starting up a Consul agent in server mode is as easy as a simple command:


```
vagrant@consul1:~$ consul agent -server -bootstrap-expect 1 -data-dir /tmp/consul
```

Now that the basics are taken care of, it's time to move on to consul's configuration management system.


### The Key/Value Store

The K/V store is supported out-of-box with the above installation. As the "Getting Started" documentation on the Consul website states, this can be tested by performing a simple curl against the HTTP API:

```
vagrant@consul1:~/consul_demo$ curl -v http://localhost:8500/v1/kv/?recurse | python -m json.tool
* About to connect() to localhost port 8500 (#0)
*   Trying 127.0.0.1... connected
> GET /v1/kv/?recurse HTTP/1.1
> User-Agent: curl/7.22.0 (x86_64-pc-linux-gnu) libcurl/7.22.0 OpenSSL/1.0.1 zlib/1.2.3.4 libidn/1.23 librtmp/2.3
> Host: localhost:8500
> Accept: */*
>
< HTTP/1.1 404 Not Found
< X-Consul-Index: 1
< X-Consul-Knownleader: true
< X-Consul-Lastcontact: 0
< Date: Wed, 17 Dec 2014 04:20:37 GMT
< Content-Length: 0
< Content-Type: text/plain; charset=utf-8
<
* Connection #0 to host localhost left intact
* Closing connection #0
vagrant@consul1:~/consul_demo$
```

A simple GET returns a 404, meaning the Key/Value store is empty.

```
vagrant@consul1:~/consul_demo$ curl -X PUT -d 'test1' http://localhost:8500/v1/kv/web/consultest1
true
vagrant@consul1:~/consul_demo$ curl -X PUT -d 'test2' http://localhost:8500/v1/kv/web/consultest2?flags=1
true
vagrant@consul1:~/consul_demo$ curl -X PUT -d 'test3' http://localhost:8500/v1/kv/web/consultest3/?flags=2
true
```

This adds three values to the keystore - test1, test2, test3 - with keys consultest1, consultest2, consultest3 respectively. The can be verified by running the previous recursive command.

```
vagrant@consul1:~/consul_demo$ curl -v http://localhost:8500/v1/kv/?recurse | python -m json.tool
* About to connect() to localhost port 8500 (#0)
*   Trying 127.0.0.1... connected
> GET /v1/kv/?recurse HTTP/1.1
> User-Agent: curl/7.22.0 (x86_64-pc-linux-gnu) libcurl/7.22.0 OpenSSL/1.0.1 zlib/1.2.3.4 libidn/1.23 librtmp/2.3
> Host: localhost:8500
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: application/json
< X-Consul-Index: 70
< X-Consul-Knownleader: true
< X-Consul-Lastcontact: 0
< Date: Wed, 17 Dec 2014 14:52:57 GMT
< Content-Length: 311
<
* Connection #0 to host localhost left intact
* Closing connection #0

[
   {
      "CreateIndex":48,
      "ModifyIndex":68,
      "LockIndex":0,
      "Key":"web/consultest1",
      "Flags":0,
      "Value":"dGVzdDE="
   },
   {
      "CreateIndex":69,
      "ModifyIndex":69,
      "LockIndex":web/consultest2",
      "Flags":1,
      "Value":"dGVzdDI=""},
   {
      "CreateIndex":70,
      "ModifyIndex":70,
      "LockIndex":0,
      "Key":"web/consultest3/",
      "Flags":2,
      "Value":"dGVzdDM="
   }
]
```

Above, we can see three new values added as base64 encoded strings. In two of the above requests, I set a flag value, a generic integer that can be used for any client need. The modify index is used to enable atomic key modification. Modification of a key is possible without using it, like so:

```
vagrant@consul1:~/consul_demo$ curl -X PUT -d 'newval' http://localhost:8500/v1/kv/web/consultest3
vagrant@consul1:~/consul_demo$ curl -v http://localhost:8500/v1/kv/web/consultest3
* About to connect() to localhost port 8500 (#0)
*   Trying 127.0.0.1... connected
> GET /v1/kv/web/consultest3 HTTP/1.1
> User-Agent: curl/7.22.0 (x86_64-pc-linux-gnu) libcurl/7.22.0 OpenSSL/1.0.1 zlib/1.2.3.4 libidn/1.23 li
> Host: localhost:8500
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: application/json
< X-Consul-Index: 88
< X-Consul-Knownleader: true
< X-Consul-Lastcontact: 0
< Date: Wed, 17 Dec 2014 15:04:17 GMT
< Content-Length: 104
<
* Connection #0 to host localhost left intact
* Closing connection #0
[{"CreateIndex":88,"ModifyIndex":88,"LockIndex":0,"Key":"web/consultest3","Flags":0,"Value":"bmV3dmFs"}]
```

Or using the optional check-and-set parameter.

```
vagrant@consul1:~/consul_demo$ curl -X PUT -d 'newval' http://localhost:8500/v1/kv/web/consultest3?cas=87
false
```

The above request failed due to a check-and-set value (87) below the ModifyIndex (88) of the key being modified. Consul's Kev/Value API even provides the ability to wait until a given ModifyIndex has been met until a given time limit has been met. Additional documentation can be found [here](https://consul.io/docs/agent/http.html).

Deleting an key is just as easy:

```
vagrant@consul1:~/consul_demo$ curl -X DELETE http://localhost:8500/v1/kv/web/consultest3/
vagrant@consul1:~/consul_demo$ curl http://localhost:8500/v1/kv/web?recurse | python -m json.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   311  100   311    0     0  18023      0 --:--:-- --:--:-- --:--:-- 19437
[
    {
        "CreateIndex": 48,
        "Flags": 0,
        "Key": "web/consultest1",
        "LockIndex": 0,
        "ModifyIndex": 68,
        "Value": "dGVzdDE="
    },
    {
        "CreateIndex": 69,
        "Flags": 1,
        "Key": "web/consultest2",
        "LockIndex": 0,
        "ModifyIndex": 69,
        "Value": "dGVzdDI="
    }
```

And a recursive delete:

```
vagrant@consul1:~/consul_demo$ curl -X DELETE http://localhost:8500/v1/kv/web/?recurse
vagrant@consul1:~/consul_demo$ curl http://localhost:8500/v1/kv/web?recurse | python -m json.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
No JSON object could be decoded
```

### Security

My examples above didn't utilize any sort of authentication mechanism. According to the Consul documentation, it supports a gossip mechanism (powered by [http://www.serfdom.io/docs/internals/security.html](Serf)) and Transport Layer Security using asymmetric encryption. More information [here](http://www.consul.io/docs/internals/security.html).

### Conclusions

Overall, I'm impressed enough with Consul's Key/Value HTTP API to take it to the next level. Now that I've got the basics down for the capabilities and usage of the API it's time to experiment around with how to utilize it to make uniform, centralized configuration management easier for the host of applications my team develops against.
