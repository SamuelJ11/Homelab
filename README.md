# Homelab

## OVERVIEW AND PROJECT GOALS

* When I was first gifted the Dell R740XD server by a dear professor who has since moved back to China, I knew I had to put it to good use.

* With 40 logical cores and 188 GB of RAM, I knew the possibilities were endless, but I really had no idea what to do with this as it was massively overkill for anything I ever needed.

* Originally, I decided the purpose of the Dell R740XD machine was to function as a VPN server and a file server, so that I may mount a network file share to any of my devices' filesystems and access my data from anywhere in the world. So far, I have not strayed too far from this original goal.

- See [homelab server](server-setup/images/server.jpg) and [homelab architecture](server-setup/architecture.txt) for a visual representation of my overall setup.

* I will continually update this README as I add new services to this machine. With that being said, equipped with nothing but the internet and sheer determination, I set out to build and document my first homelab...

## CONFIGURATIONS

### Creating the WireGuard VPN

* After wiping the original OS and replacing it with Proxmox, the first thing I needed to do was configure a secure VPN to my home network.

- See [Proxmox Setup](server-setup/images/proxmox.png) for what the web interface for Proxmox looks like when your physical setup is complete.

* I decided to host an LXC container running WireGuard for its customizability and zero dependence on third-party authentication providers or centralized management platforms.

* The first thing I did was configure the WireGuard VPN configuration file on the server LXC container:

```bash
[Interface]

Address = 10.0.0.1/24

SaveConfig = true

PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE;

PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE;

ListenPort = 51820

PrivateKey = XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXAAA=

[Peer]

PublicKey = p8F7QN6rmS9rnsgXFKEt91kvYuGwcnOY7cURz8x7DWU=

AllowedIPs = 10.0.0.2/32
```

* Next, while still inside the container, I ran the following command:

```bash
root@wireguard:~# ip route add 172.16.0.0/16 via 192.168.4.11 dev eth0
```

* This line will be explained in the Cisco CML section.

- Then I did the same on my laptop:

```bash
[Interface]

PrivateKey = XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXAAA=

Address = 10.0.0.2/24

MTU = 1420

#DNS = 1.1.1.1  <--- comment this out to use your public wifi's DNS resolver

[Peer]

PublicKey = 8Tsc+Fk1HKA0VM1tNIhTRqy2USny3sryiu7+4hnlaDE=

AllowedIPs = 10.0.0.0/24, 192.168.4.0/24, 172.16.0.0/16

Endpoint = [PUBLIC IP]:51820

PersistentKeepalive = 21
```

* See [WireGuard Dashboard Setup](server-setup/images/Wireguard_Dashboard.png) for an example of the web GUI for WireGuard after successful implementation.

### Configuring the NFS Server

* Next, I needed to set up the NFS (Network File System) server on the Dell machine and configure my PC as the sole NFS client.

- It should be noted for context that during the Proxmox installation, I selected ZFS (RAID 0) for the root filesystem, which creates a default ZFS pool named `rpool`.

- A dataset in ZFS is a logical filesystem created inside a storage pool.

- ZFS is an extremely efficient filesystem that uses a copy-on-write resource management technique and comes with built-in volume management capabilities.

* I began by creating the dataset named `storage`, which I mounted to `/media/storage`. The output of the `zfs list` command on the server is shown below:

```bash
root@sams-server:~# zfs list

NAME                           USED  AVAIL  REFER  MOUNTPOINT

rpool                         18.9G  1.74T   104K  /rpool

rpool/ROOT                    3.20G  1.74T    96K  /rpool/ROOT

rpool/ROOT/pve-1              3.20G  1.74T  3.20G  /

rpool/data                    8.36G  1.74T    96K  /rpool/data

rpool/data/subvol-100-disk-0  1.08G  2.92G  1.08G  /rpool/data/subvol-100-disk-0

rpool/data/vm-101-disk-0       104K  1.74T   104K  -

rpool/data/vm-101-disk-1      7.28G  1.74T  7.28G  -

rpool/storage                 7.15G   493G  7.15G  /media/storage

rpool/var-lib-vz               124M  1.74T   124M  /var/lib/vz
```

* The storage dataset was created via the `zfs create rpool/storage` command.

- Now that the dataset that will hold my files is created, I needed NFS to actually export this directory:

```bash
root@sams-server:~# zfs set mountpoint=/media/storage rpool/storage
```

* To prevent the dataset from consuming the entire 1.74 TB system pool should it grow too large (not that it ever will), I set a 500 GB cap, or "quota," on the dataset:

```bash
root@sams-server:~# zfs set quota=500G rpool/storage
```

* The next server-side configuration was setting up the host to export the dataset exclusively to my local subnet by modifying the `/etc/exports` configuration file:

```bash
# /etc/exports: the access control list for filesystems which may be exported
#               to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#

/media/storage 192.168.4.26(rw,sync,no_subtree_check)
```

* After exporting the share (`/media/storage`) and starting the service, I was good to go:

```bash
root@sams-server:~# exportfs -rv

root@sams-server:~# systemctl enable --now nfs-kernel-server
```

* To adhere to the principle of least privilege, I created the `nfsusers` group and added `samuel`; only this user/group has permission to view the NFS files.

```bash
root@sams-server:~# groupadd nfsusers
```

* Note that since the `samuel` user already existed, I simply added it to the `nfsusers` group and unlocked its Bash shell, since its shell had disabled login by default (`/usr/sbin/nologin`):

```bash
root@sams-server:~# usermod -aG nfsusers samuel

root@sams-server:~# chsh -s /bin/bash samuel
```

* We should note that the user `samuel` doesn't have a home directory yet, so we created it and assigned the proper ownership:

```bash
root@sams-server:~# mkhomedir_helper samuel

root@sams-server:~# chown -R samuel:nfsusers /home/samuel
```

* Note that the last command above is needed later when we configure SSH keys for the SSHFS client (my laptop).

- Finally, we update the permissions on the `/media/storage` dataset so that any user in the `nfsusers` group has read and write access:

```bash
root@sams-server:~# chown -R samuel:nfsusers /media/storage

root@sams-server:~# chmod -R 775 /media/storage
```

* This allows daily file operations over NFS without relying on root access.

### Configuring the NFS Client

* On the NFS client (my PC), I installed both NFS and autofs so that the filesystem will be mounted automatically:

```bash
samuel@sam-pc:~$ sudo apt install nfs-common autofs
```

* Next, I edited the autofs master file, which specifies which map files the autofs daemon should read:

```bash
#
# Sample auto.master file
#
# This is a 'master' automounter map and it has the following format:
# mount-point [map-type[,format]:]map [options]
# For details of the format look at auto.master(5).
#

/mnt /etc/auto.nfs --timeout=180 browse
```

* Here, `/mnt` tells autofs that it manages the entire `/mnt` parent directory as a base folder.

* `/etc/auto.nfs` is the actual map file containing the relative paths to build inside `/mnt`.

* The timeout specifies how long the NFS share should be mounted before it is unmounted due to inactivity.

* The `browse` keyword I found was very helpful as it keeps the subfolder (like `/mnt/nfs`) visible in `ls` or the GUI file manager even when the NFS share is currently unmounted.

- The last file to edit was the map file itself (`/etc/auto.nfs`):

```bash
nfs -fstype=nfs,rw,soft,intr 192.168.4.27:/media/storage
```

* Here, `nfs` is the local mount on my PC for the NFS file share, and `192.168.4.27:/media/storage` is the server IP address and filepath to the actual files being exported.

- Of course we can't forget:

```bash
samuel@sam-pc:~$ sudo systemctl enable --now autofs
```

### Configuring the SSHFS Client

* While NFS is lightning fast over a local network, it suffers greatly over a WAN or VPN with high latency due to it being a "chatty" protocol.

* Since I mostly use my laptop while away from home, I'll configure an SSHFS mount to get faster upload and download speeds to/from my file server.

* Regarding security and permissions, I restricted my laptop's remote access to the unprivileged `samuel` account rather than root.

  * If my laptop ever got compromised (either hacked or stolen), I want to be sure that the potential blast radius is limited strictly to the `/media/storage` dataset.

* Configuration was relatively simple:

  1. I copied my laptop's public SSH key to the `./ssh/authorized_keys` file on the server's `samuel` account.

  2. Next, I installed the `sshfs` utility on my laptop:

```bash
samuel@sam-laptop:~$ sudo apt install sshfs
```

```
3. Finally, I created a local mount point and established the remote connection:
```

```bash
samuel@sam-laptop:~$ mkdir ~/mnt/sftp

samuel@sam-laptop:~$ sshfs samuel@192.168.4.27:/media/storage ~/mnt/sftp -o reconnect,ServerAliveInterval=15
```

## 3. CISCO CML