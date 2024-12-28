# NetworkManager

```sh
# is NetworkManager running?
nmcli -t -f RUNNING general

# get general status
nmcli general

# show devices
nmcli dev status

# show all connections
nmcli con show

# show the specified connection
nmcli con show "Wired connection 1"

# no auto connect
nmcli con mod "Wired connection 1" connection.autoconnect no
cat /etc/sysconfig/network-scripts/ifcfg-eth0 | grep ONBOOT
# ONBOOT=no

# change boot protocol to DHCP
nmcli con mod "Wired connection 1" ipv4.method auto
cat /etc/sysconfig/network-scripts/ifcfg-eth0 | grep BOOTPROTO
# BOOTPROTO=dhcp

# config static IP
nmcli con mod "Wired connection 1" \
  ipv4.method 'manual' \
  ipv4.addresses '192.168.250.100/24' \
  ipv4.gateway '192.168.250.1' \
  ipv4.dns '8.8.8.8,114.114.114.114' \
  ipv4.ignore-auto-dns 'true'

# disable IPv6
nmcli con mod "Wired connection 1" ipv6.method ignore

# add DNS server
nmcli con mod "Wired connection 1" +ipv4.dns 8.8.4.4
cat /etc/sysconfig/network-scripts/ifcfg-eth0 | grep DNS

# delete DNS server
nmcli con mod "Wired connection 1" -ipv4.dns 8.8.4.4
nmcli con mod "Wired connection 1" -ipv4.dns 8.8.8.8,114.114.114.114

# nmcli interactive connection editor
nmcli con edit "Wired connection 1"

# monitor
nmcli con monitor "Wired connection 1"

# add connection (static)
nmcli con add \
    con-name eth2 \
    type ethernet \
    ifname eth2 \
    ipv4.method manual \
    ipv4.address '192.168.0.10/24' \
    ipv4.gateway '192.168.0.1'

# add connection (DHCP)
nmcli con add \
    con-name eth3
    type ethernet \
    ifname eth3 \
    ipv4.method auto

# activate a connection
nmcli con up eth2

# show active connections
nmcli con show --active

# deactivate a connection
nmcli con down eth2

# delete a connection
nmcli con del eth2

# show host name
nmcli general hostname

# change host name
nmcli general hostname linux

# default router
nmcli con mod "Wired connection 1" ipv4.never-default yes

# reload connection
nmcli con reload
```
