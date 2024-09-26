`sudo apt-get install libvirt-daemon libvirt-clients qemu-kvm`
`vagrant box add generic/debian9 `
`vagrant init generic/debian9`
```
Vagrant.configure("2") do |config|
  config.vm.box = "generic/debian9"
end
```
`sudo systemctl start libvirtd`
`sudo apt-get install nftables`
`sudo nano /etc/nftables.conf`
```
#!/usr/sbin/nft -f
table inet filter {
  chain input {
    type filter hook input priority 0;

    # Allow established/related connections
    ct state {established, related} accept

    # Drop invalid connections
    ct state invalid drop

    # Allow loopback interface
    iifname lo accept

    # Allow ICMP (for IPv4 and IPv6)
    ip protocol icmp accept
    meta l4proto ipv6-icmp accept

    # Allow SSH
    tcp dport ssh accept

    # Allow libvirt NAT traffic
    iif virbr0 accept

    # Default reject
    reject with icmpx type port-unreachable
  }
}
```
sudo systemctl start nftables
sudo systemctl enable nftables
sudo nft list ruleset

sudo virsh net-list --all
sudo systemctl start libvirtd
sudo systemctl enable libvirtd
vagrant up
