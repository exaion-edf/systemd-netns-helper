# Systemd network namespace helper

## What is it useful for?

This tool helps building services that are isolated on different network
interfaces.

Let’s say you have a server running nginx, with a frontoffice interface
(eth0) and an admin interface (eth1). You want ssh and system services
to do everything on eth1, while nginx is running on eth0, with its own
routing table.

## How to configure

### Base configuration

Configure networking for the system, leaving alone interfaces you want
to keep for specific services.

In our example, this means in `/etc/network/interfaces`:

	auto lo
	iface lo inet loopback
	
	auto eth1
	iface eth1 inet static
	   ...

### Per-service network configuration

For each service you want running on a specific interface, create a specific
*interfaces* file in `/etc/netns/service/network/`. In our example, that
would be `/etc/netns/nginx/network/interfaces/` containing:

	auto lo
	iface lo inet loopback
	
	auto eth0
	iface eth0 inet static
	   ...

It also works with VLAN interfaces:

	auto eth1.124
	iface eth1.124 inet static
	   ...

If you need static DNS configuration, don’t forget to use *nameserver* stanzas
or create a `/etc/netns/service/resolv.conf` accordingly.

> Note: for the system, you can use whatever network management tool you want
> (networkd, ifupdown…). But for now only ifupdown is supported to configure
> the network *inside* the namespace.

### Per-service systemd configuration

We will create a drop-in configuration directory for the service we are 
modifying in `/etc/systemd/system/myservice.service.d`. The contents of the 
drop-in file is always the same, so we can just use a symlink.

In our example:

	mkdir /etc/systemd/system/myservice.service.d
	ln -s /usr/share/systemd-netns-helper/netns.conf /etc/systemd/system/myservice.service.d/

### Re-launching the service

Just stop the service and restart it after taking into account our
modifications:

	systemctl stop nginx.service
	systemctl daemon-reload
	systemctl start nginx.service

### Controlling the behavior

All processes launched by the wrapped service should now run in the netns:

	ip netns pids nginx

The output will show all the PIDs that have been wrapped.

## What if I want to run several services on the same interface?

Configure one service in the way we described. Pick the service that
needs to **start first**. We will call it `service1`.

For other services (e.g. service2), instead of using a symlink, create
`/etc/systemd/system/service2.service.d` containing the `netns.conf` file
as follows:

	[Unit]
	BindsTo=systemd-netns@service1.service
	After=systemd-netns@service1.service
	
	[Service]
	NetworkNamespacePath=/run/netns/service1

If you want these services to be correctly restarted whenever you
restart the master service, you can also add a file in
`/etc/systemd/system/service1.service.d/deps.conf`:

	[Service]
	ExecStartPost=+/bin/systemctl restart service2

## What if I want to communicate with the namespace?

We can automatically create a pair of veth devices to that effect.

In `/etc/network/interfaces` create one end of the link. Do not use the
`auto` stanza, instead use `link-netns` this way:

	link-netns service1 link0 link1
	iface link0 inet static
	  address 100.64.0.0/31

In `/etc/netns/service1/network/interfaces`, just create the other
end of the link:

	auto link1
	iface link1 inet static
	  address 100.64.0.1/31

Now you can communicate with the namespace, and route or proxy your
packets as you see fit.

## What if I want to do something insane with IPsec tunnels?

The real reason why this program was written is to run IPsec gateways in
complex network setups with several routing tables. The thing you want to
do in this case is to **put IPsec connections in a separate namespace**
from the IPsec server.

This tool will do that for you if you create a new namespace, let’s call
it `tunnel`. You will write:

 * A `/etc/netns/tunnel/network/interfaces` the same way you did for
   the service.
 * An `xfrm` file in the same directory as the `interfaces` file for the
   service namespace.

For example, if you are using it with charon-systemd, the xfrm file would
be located in `/etc/netns/strongswan/network/xfrm` and look like the
following:

	[ipsec0]
	IfId=17
	DestNetNS=tunnel
	Prefixes=10.143.12.0/24

	[ipsec1]
	IfId=18
	DestNetNS=tunnel2
	UnderlayDev=ens5
	Prefixes=10.143.14.0/23, 10.143.18.0/27
	BlockP2P=true

This example creates two XFRM interfaces, named *ipsec0* and *ipsec1*.
For each one the settings mean:

 * IfId (mandatory): interface ID of the XFRM interface. This is the
   setting you would find in your `swanctl.conf` file, named
   *if\_id\_out* and *if\_id\_in*.
 * Prefixes (mandatory): comma-separated list of network prefixes that
   need to be routed in your XFRM interface. This would be the IP
   prefixes of your IPsec pools or subnets that need to be routed in
   the tunnel.
 * DestNetNS: namespace to which you want to move the tunneled traffic.
   Defaults to *xfrm\_myservice*.
 * UnderlayDev (optional): if you want hardware-accelerated IPsec, set
   this to the physical interface that holds the IPsec service (in the
   namespace allocated to strongswan).
 * BlockP2P (optional): setting this will add the route to your prefixes
   in a specific routing table that is only available to the upstream
   link. If the next hop is a firewall (which is recommended), the
   firewall will filter all traffic between VPN hosts.

## Caveats

 * Missing IPv6 support (should be easy)
 * Missing networkd support (impossible without netns support in networkd)
