# Home Lab: Kali Linux + Metasploitable 2

## What this is

A small lab for practicing on a deliberately vulnerable machine, kept fully isolated from my actual network. Running on a Windows host with 8GB RAM, using VirtualBox.

- **Attacker box:** Kali Linux, static IP 192.168.56.102
- **Target box:** Metasploitable 2, static IP 192.168.56.101
- **Network:** VirtualBox Internal Network ("labnet"), not connected to the internet or my home network

I went with an internal network instead of NAT or bridged mode on purpose. Metasploitable is full of unpatched vulnerabilities by design, so I didn't want it reachable from anywhere outside the lab, or able to reach out either.

## Setting it up

I already had Kali running as a VM (2GB RAM). Metasploitable was a bit different to get going, it doesn't come as an OVA you can just import like Kali does. It's a zipped VMDK file, so I had to create a new VM manually in VirtualBox and point it at the extracted disk file instead of using the usual import wizard. Gave it 1GB RAM since it's old and doesn't need much.

Set both VMs' network adapters to Internal Network, and made sure the network name was typed identically on both ("labnet"). Found out the hard way that this is case sensitive, if the names don't match exactly, VirtualBox just silently creates two separate networks instead of connecting the machines.

## What went wrong (and how I fixed it)

This ended up being the most useful part, honestly. Documenting it here because working through it taught me more than the setup itself did.

**No IP address after booting either VM.** Turns out Internal Network mode in VirtualBox doesn't run DHCP, unlike NAT or Bridged mode. Nothing hands out addresses automatically, so both machines needed a static IP set by hand.

**ifconfig said it worked but nothing actually changed (on Kali).** I ran the standard `ifconfig eth0 <ip> netmask <mask> up` command, no error came back, but checking with `ip a` afterward showed the interface still had no address. Switched to the newer `ip` command instead:
```
sudo ip addr add 192.168.56.102/24 dev eth0
sudo ip link set eth0 up
```
That one actually stuck.

**"Network is unreachable" when I tried to ping.** This happened before Kali's interface was properly up. Useful to know in hindsight, that specific error means the problem is on your own machine's interface, not the target or the path to it.

**Then "destination host unreachable" once Kali was fixed.** Different error, different meaning, this one pointed at the other end not responding. Sure enough, Metasploitable's address hadn't actually taken either. Same fix as Kali sorted it, and the ping finally went through.

**Static config didn't survive a reboot on Metasploitable.** To stop having to set the IP by hand every time I booted the VMs, I added a static block to `/etc/network/interfaces` on both. After rebooting Metasploitable, the boot log showed "Configuring network interfaces... [fail]" and eth0 came up with no address again.

Turned out the original default config was still in the file, I'd added my static block underneath it instead of replacing it, so there were two conflicting `eth0` entries. The network service didn't know which one to use and just failed. Deleted the old DHCP block, left only:
```
auto eth0
iface eth0 inet static
    address 192.168.56.101
    netmask 255.255.255.0
```
Ran `sudo ifdown eth0 && sudo ifup eth0` to apply it without a full reboot, and it worked straight away.

## Where it landed

Both VMs now boot up already networked, with static IPs that survive a restart. No more manual setup between sessions.

## What I took from this

- Internal Network mode in VirtualBox means you're on your own for IP assignment, there's no DHCP unless you set one up.
- Don't trust a command just because it didn't throw an error. Check the actual state afterward.
- The exact wording of a network error tells you where to look. "Network unreachable" means check your own machine first. "Host unreachable" means check the other end.
- When you're editing a config file, look for old conflicting entries before adding new ones. Appending without checking what's already there is how this whole thing broke.

## Next

Now that the lab's stable, next step is running through some TryHackMe rooms against it and writing each one up separately in this repo.
