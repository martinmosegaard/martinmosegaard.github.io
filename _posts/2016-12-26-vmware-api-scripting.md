---
layout: post
title:  VMware API Scripting
date:   2016-12-26
categories: vmware
---

*First published on <http://www.praqma.com/stories/vmware-api-scripting/>*
![Servers](/images/multipleserver.jpg){: .pic .medium .center}

VMware vSphere is widely used for virtualization in the enterprise. Thus you may face the task of managing
infrastructure on this platform and of course this should be done *as code*. This post will give you a jumpstart into
scripting the vSphere API.

## Scripting the vSphere API

The VMware vSphere product contains two major components:

- ESXi, the free hypervisor.
- vCenter Server, for managing a bunch of ESXi hosts.

Both ESXi and vCenter Server provide a web service that you can access by using the vSphere API. The functionality
exposed allows you to do everything you can do in the vSphere clients. For example power on virtual machines or create
inventory objects like resource pools or network port groups.

The vSphere API has bindings for multiple languages. This post uses the Ruby binding, `rbvmomi`.
Like its relatives, `pyvmomi` and `govmomi`, it is available as open source on GitHub with both guides and samples
for getting started.

<br/>

![VMware API scripting](/images/vmware-api-scripting.png){: .pic .medium .center}


## The Inventory and the MOB

When you start working with the vSphere API, it will be helpful if you are already familiar with the vSphere inventory
model from one of the vSphere clients. You may have seen the four main inventory hierarchies: Hosts & clusters,
VMs & templates, Datastores, and Networks.

While these hierarchies reflect many parts of the underlying model, they also hide away some details. For example, a VM
always belongs to a resource pool (or vApp, which is a kind of resource pool), even if you cannot see it in the
client. Things like these are meant to make it easier to use the client.

You can get a more realistic view of the object model by using the Managed Object Browser (or MOB). It is a web
application available for both ESXi and vCenter Server.It allows you to explore all details of the objects on a
server and even invoke methods on them. I find it to be a good companion along with the API documentation when writing
scripts.

The MOB is available at the `mob` context of the server, for example: `https://hostname.company.com/mob`. But be
aware that you [may need to enable it](http://www.virtuallyghetto.com/2015/02/quick-tip-vsphere-mob-is-disabled-by-default-in-esxi-6-0.html){: target="\_blank"}.
*Hint:* In the new web UI that comes with ESXi 6.5, you can enable the MOB directly from the Help menu.


## Running ESXi in VirtualBox

To get started with the vSphere API for no cost at all, on a single computer, let's try running ESXi in VirtualBox.
ESXi is free and so is VirtualBox, so why not?

Now this is a part where your mileage may vary. I have seen several blog posts stating this can be done, but with
various caveats. I got it working using the details in this post. But be warned that this is a route not supported by
VMware or Oracle. It involves the concept of *nested virtualization*, that is, running a VM in a VM. VMware products
are actually pretty good at nested virtualization, as long as it only involves VMware products - for example ESXi in
ESXi or in Workstation. But they are probably not interested in supporting VirtualBox. Oracle, on the other hand,
offers VirtualBox as a free, open source product. In this setting, it is [not easy](https://www.virtualbox.org/ticket/4032){: target="\_blank"}
to fund such advanced features.

To get an ESXi installer, all you have to do is create an account at VMware and download the ISO. I used the 6.5
version, `VMware-VMvisor-Installer-6.5.0-4564106.x86_64.iso`. Then I took the following steps:

- Set OS type to `Linux - Other Linux (64-bit)`.
- Set memory to 4 GB (4096 MB).
- Create a dynamically allocated disk of 20 GB.
- Set number of processors to 2.
- Change the network adapter to be a `Host-only Adapter`.
- Attach the ISO.
- Power on and start the installation. Ignore the warning about hardware virtualization not enabled.


## Results and Limitations

I was able to install and start ESXi in VirtualBox and then I could access the web client and the MOB.
But since it did not recognize hardware virtualization, I could not power on a 64-bit VM inside ESXi. That's OK.
I could still test my scripts by creating and powering on a 32-bit Alpine Linux VM inside ESXi.

Also notice that vCenter-only features will not work when using the API against a standalone ESXi host. For example
using clusters or cloning a VM template.

Finally, ESXi is installed with a 60-day trial license. After that, you will lose a lot of the API features without a
license.

## Now Some Code

Here is a tiny [rbvmomi](https://github.com/vmware/rbvmomi){: target="\_blank"} script that connects to the vSphere API:

```ruby
require 'rbvmomi'
require 'pry'

host     = ENV['vsphere_host'] || '192.168.99.100'
user     = ENV['vsphere_user'] || 'root'
password = ENV['vsphere_password'] || raise('vsphere_password not set')

vim = RbVmomi::VIM.connect host: host, user: user, password: password, insecure: true
dc = vim.serviceInstance.find_datacenter('ha-datacenter') || raise('datacenter not found')

puts "Datacenter [dc]: #{dc.name}"

Pry.start # For an interactive session
```

When you connect in an interactive session, you can browse the inventory in the same way as the MOB - or by following
the official SDK documentation. Here is an example where we find the first VM, see that it is running, and power it
off:

```sh
$ pry vsphere.rb
Datacenter [dc]: ha-datacenter
[1] pry(main)> dc.name
=> "ha-datacenter"
[2] pry(main)> dc.vmFolder.childEntity[0].name
=> "alpine"
[3] pry(main)> dc.vmFolder.childEntity[0].summary.runtime.powerState
=> "poweredOn"
[4] pry(main)> dc.vmFolder.childEntity[0].PowerOffVM_Task
=> Task("haTask-3-vim.VirtualMachine.powerOff-194292129")
```

This post showed you how to access the vSphere API directly. You may not always need this. If you use Chef for example,
you may be able to use their wrapper, [knife-vsphere](https://github.com/chef-partners/knife-vsphere){: target="\_blank"}. But, as with
most open source, it may be missing just that bit of functionality that you require.



## More Resources

If you found this interesting, you should check out:

- [Examples in Ruby](https://github.com/vmware/rbvmomi/tree/master/examples){: target="\_blank"}
- [More examples in Python](https://github.com/vmware/pyvmomi-community-samples){: target="\_blank"}
- [The automation resources on the virtuallyGhetto blog](http://www.virtuallyghetto.com/kickstart){: target="\_blank"}
