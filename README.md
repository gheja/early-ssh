early-ssh
=========

early-ssh gives you an SSH server during the boot of your Linux system. It starts before the root filesystem is mounted so you can unlock your encrypted root filesystem interactively, you don't have to be at the console of your server. You can also scp files to your server so you can even start your key-encrypted system.

early-ssh is basically an update-iniramfs hook script and a boot script that starts dropbear SSH server in the initramfs during boot.

Features
--------

### SCP to the server
You can copy files to the server using scp from a remote host. This way you it is possible to copy key files to unlock key-based LUKS even for the root filesystem.

### Dedicated root password
It is possible to use a different root password than the one use on your system (it can be disabled as well) by adding a `SHADOW_OVERRIDE` option specifying a different shadow file.

See `PASSWD_OVERRIDE`, `SHADOW_OVERRIDE`, `GROUP_OVERRIDE` in config.

Example: `SHADOW_OVERRIDE="/etc/early-ssh/shadow"`

### Timeout to continue the boot
A timeout can be specified to continue the boot after a certain time.

See `TIMEOUT` in config.

Example: `TIMEOUT=600`

### Specify network interface by MAC address
If you have more than one network interface and they get scrambled on boot it is possible to specify the correct interface by MAC address instead of the name ("eth0" for example).

See `IFACE` in config.

Example: `IFACE="MAC 01:02:03:04:05:06"`

### Put your ssh server on a different port
You can move the early-ssh server on a different port.

See `PORT` in config.

Example: `PORT=2233`


A real-life example with RAID-1 and LUKS
----------------------------------------

content of zaphod-init.sh
```
#!/bin/sh

# load the RAID-1 module for mdadm
modprobe raid1

# create the md special devices
for i in 0 1 2 3; do mknod /dev/md${i} b 9 ${i}; done

# scan for all mdadm arrays (can be good if you forget to update your initramfs)
mdadm --examine --scan > /tmp/mdadm.conf

# assemble the RAID-1 arrays
mdadm --assemble --scan --config /tmp/mdadm.conf

# load the dm-crypt module for LUKS
modprobe dm-crypt

# do the LUKS openings (md0 is the /boot, so it is not encrypted!)
for i in 1 2 3; do cryptsetup --key-file /tmp/zaphod-md${i}.key luksOpen /dev/md${i} md${i}_crypt; done
```

An SSH session:
```
mobilem00:/mnt/pendrive# scp zaphod-init/* root@xxx.xxx.xxx.xxx:/tmp
root@xxx.xxx.xxx.xxx's password:
zaphod-init.sh                                 100%  338     0.3KB/s   00:00
zaphod-md1.key                                 100%  256     0.3KB/s   00:00
zaphod-md2.key                                 100%  256     0.3KB/s   00:00
zaphod-md3.key                                 100%  256     0.3KB/s   00:00

mobilem00:/mnt/pendrive# ssh root@xxx.xxx.xxx.xxx
root@xxx.xxx.xxx.xxx's password:

Welcome to early-ssh!

After you have finished everything, run the following to continue booting:
  finished


Please send your comments and bugreports to <xxx@xxx.xx>


BusyBox v1.1.3 (Debian 1:1.1.3-4) Built-in shell (ash)
Enter 'help' for a list of built-in commands.

~ # chmod 700 /tmp/zaphod-init.sh

~ # /tmp/zaphod-init.sh
mdadm: /dev/md0 has been started with 2 drives.
mdadm: /dev/md1 has been started with 2 drives.
mdadm: /dev/md2 has been started with 2 drives.
mdadm: /dev/md3 has been started with 2 drives.
key slot 0 unlocked.
Command successful.
key slot 0 unlocked.
Command successful.
key slot 0 unlocked.
Command successful.

~ # finished
Your session will be now terminated and the boot will be continued. Bye!

Connection to xxx.xxx.xxx.xxx closed by remote host.
Connection to xxx.xxx.xxx.xxx closed.
```


mdadm RAID assembly example [draft]
---------------------------

```
ssh root@1.2.3.4
modprobe raid1
mdadm --examine --scan > /tmp/mdadm.conf
mdadm --assemble --scan --config /tmp/mdadm.conf
finished
```

LUKS device unlocking example [draft]
-----------------------------

```
ssh root@1.2.3.4
modprobe dm_crypt
cryptsetup luksOpen /dev/sda2 sda2_crypt
finished
```

advanced setup (mdadm + drbd + LUKS + LVM) [draft]
------------------------------------

```
ssh root@1.2.3.4
modprobe raid1
mdadm --examine --scan > /tmp/mdadm.conf
mdadm --assemble --scan --config /tmp/mdadm.conf

modprobe drbd
hostname trish
ifconfig eth1 172.16.254.1 netmask 255.255.255.0 up
drbdadm up r0
drbdadm primary r0

modprobe dm_crypt
cryptsetup luksOpen /dev/drbd0 drbd0_crypt

pvscan
vgchange -a y

finished
```

