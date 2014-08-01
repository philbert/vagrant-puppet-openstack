vagrant-puppet-openstack
=====================

This Vagrantfile will bring up a 5-node Openstack Icehouse environment (Puppet Master, Controller, Compute, Network and Storage nodes) on virtualbox. It uses Puppet Enterprise and the puppetlabs-openstack modules on Centos 6.5. 

I explicitly install all of the versions of the openstack dependencies that I have verified to work. Verficiation has really just been limited to basic smoke testing: everything that is necessary to create an instance, ssh to it and verify it can reach the outside world.

The complete provisioning after 'vagrant up' takes a long time (like 1hr+). At the end of it, all the required puppet modules should be installed, ip addresses and networks configured, nodes and classes added, agents run and converged. That all sounds great, but you still won't have a working system yet. For that you'll need to read the notes below.

Host machine configuration
------------
This project was build with the following host machine configuration:
* OSX Mavericks (10.9.4)
* Vagrant (1.6.2)
* Virtualbox (4.3.12)
* Enable ip forwarding and setup NAT for instances to allow internat access. See below how to do this.

Login information
------------
The puppet master is available on https://192.168.11.3 
These login details are created by vagrant-pe_build
user: admin@puppetlabs.com
pass: puppetlabs

Openstack horizon dashboard is available on https://192.168.11.4
These login details are configured in the openstack.yaml hiera file
user: test
pass: abc123

Additional host setup for NAT and IP forwarding
-------------------
Because the Openstack external network is actually setup as a host-only network in virtualbox, the only way to give created instances internet access is to setup NAT for them. At the moment, I don't have a better way to do this than to manually set this up on the host machine :( I think it might be possible to specify the external network to use a bridged network, but then you're probably going to run into issues trying to use static ips on a network with dhcp enabled (unless you're on a network that doesn't have dhcp, but when is that ever the case?).

If you have a better way to do this please let me know or changeit and send me a PR!

So here is how to setup ip forwarding and NAT on your (OSX) host:
Enable ip forwarding:
<code>sudo sysctl -w net.inet.ip.forwarding=1</code>
Add this line to the file /etc/pf.conf on your host after the line: 'nat-anchor "com.apple/"'
<code>nat on { en0 } from 192.168.22.101/32 to any -> { (en0) }</code>
Flush and reload the new rules:
<code>sudo pfctl -F all -f /etc/pf.conf</code>
Enable the packet filter:
<code>sudo pfctl -e</code>

There are a couple of caveats with this method:
<ol>
<li>If you are connecting to the net on an interface other than en0, then make sure you change the nat rule to the correct interface</li>
<li>If you have more than once instance you will need to add another rule for that too, or fiddle around to get a correct CIDR that only your instances are on. You can't just forward everything on 192.168.22.0/24 because some traffic on that network is destined for node-node communication. I never bothered to do this because I recognise there are limits to how much I can torture my poor little laptop</li>
<li>If you reboot your host, then you'll need to run the commands above again</li>
<li>If you're on linux, you can do the same with iptables. If you're on Windows... I'm sorry</li>
</ol>

Idiosyncrasises of the environment
-------------------
Either due to the behaviour of Openstack or how the puppet modules configure the Openstack services, there are a bunch of issues that you will run into with this system that you need to know how to detect and fix to make it work properly. I will try to document here all the main things that I've found (at least with this version and method of building the system). If you find more then let me know, or update this readme and send me a PR.

A bit of a disclaimer: Some of the things I mention below may not be totally correct. I'm writing this with my current level of understanding of Openstack (and puppet) and I still have a lot to learn.
<ol>
<li>Always check that Openstack thinks all your services, compute agents and network agents are up and running on http://192.168.11.4/dashboard/admin/info/. If any of them are in State down, log into the host and restart the service.</li>
<li>If you log into the Network node and run <code>ip netns</code> it will likely return nothing on a freshly build system. If it is working you should see two items qdhcp-<UUID> and qrouter-<UUID>. From what I understand, namespaces are initialised by the kernel, so the only thing that I've found to fix this issue is a reboot of the network node</li>
<li>If you reboot the network node, you may also end up with with some network abnormalities on the compute node. I have typically observed this as DHCP discovery message being sent from a created instance being swallowed on the compute node. Continue reading for debugging and fixing.</li>
<li>If you reboot the compute or network nodes, or start up the system from being shutdown, you will likely end up with some missing network configuration in iptables that should be setup by the agents such as openvswitch, l3 and dhcp. You can tell this by running <code>iptables -S</code> (and knowing what to look for).</li>
<li>On the compute node, if you are missing the chain neutron-openvswi-sg-chain, rules for the tap interface (and a load of others), then you need to <code>service neutron-openvswitch-agent restart</code>. More than likely you will also need to <code>service openstack-nova-compute restart</code> as well.</li>
<li>On the network node it is a similar thing if you are missing the neutron-openvswi-sg-chain (plus a bunch of others), you need to <code>service neutron-openvswitch-restart</code>.</li>
<li>Check that iptables is also setup correctly for the router on the network node. Run <code>ip netns</code> to list the namespaces and then check <code>ip netns exec qrouter-<UUID> iptables -S</code>. If you are missing rules for "neutron-l3-agent-BLAH" then you need to <code>service neutron-l3-agent restart</code>. If you are missing rules for "neutron-vpn-agen-BLAH" then you need to <code>service neutron-vpn-agent restart</code>
<li>If you create an instance and it does not appear to get an ip address, there are two most likely causes: 1. DHCP discovery messages are not making their way to the DHCP server, or 2. DHCP offers are not getting back from the DHCP server to the instance. The way to determine on which side the problem is run <code>tcpdump -i br-int -vvv -s 1500 '((port 67 or port 68) and (udp[8:1] = 0x1))'</code> which checks for all DHCP-packets on the br-int integration bridge interface. Run this on both the compute node and the network node at the time the instance is trying to send the discovery message (to see when the instance is doing this look for "Sending discover..." in the instance console log).</li>
<li>If you are not seeing DHCP discovery messages on the br-int interface of the compute node then you should restart the openstack-nova-compute and neutron-openvswitch-agent</li>
<li>If you are not seeing DHCP discovery messages on the br-int interface of the network node then you should restart the neutron-openvswitch-agent and neutron-l3-agent</li>
<li>If you are not seeing DHCP offer responses on the br-int interface of the network node then you should restart the neutron-dhcp-agent</li>
<li>If you are still having problems getting an ip address to the instance, but you've checked all the above, then you've likely hit something that I've not seen. Submit me an issue and I'll see if I can help!</li>
<li>If you have problems with the Horizon user interface with an error page (might be a 50x or 40x) saying "Oops something went wrong" then check to see if the rabbitmq-server is running on the control node</li>
<li>After allocating an external IP to the instance you should be able to log into the cirros test instance with <code>ssh cirros@192.168.22.101</code>, however you probably won't have internet access from the instance unless you followed my directions above to setup packet forwarding and NAT to the instance.</li>
</ol>

Additional Resources
-------------------

- Good, but a bit outdated, link to explain some of the typical problems you can envounter with Openstack networking
<http://docs.openstack.org/openstack-ops/content/network_troubleshooting.html>
- Nice diagram about packet flow through iptables
<http://vinojdavis.blogspot.ie/2010/04/packet-flow-diagrams.html>
- Some explanations about networking works on virtualbox
<https://blogs.oracle.com/fatbloke/entry/networking_in_virtualbox1>
- Pretty good information about trying to get Openstack running on virtualbox
<https://blogs.oracle.com/ronen/entry/diving_into_openstack_network_architecture>
- Indepth information about how openvswitch works that you really need to know if you're trying to debug Openstack network issues
<http://docs.openstack.org/admin-guide-cloud/content/under_the_hood_openvswitch.html>
- Where I found how to setup PF and NAT on OSX for virtualbox
<https://forums.virtualbox.org/viewtopic.php?f=8&t=47959>

###HAPPY OPENSTACKING!
