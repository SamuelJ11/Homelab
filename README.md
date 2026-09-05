# Homelab

## OVERVIEW AND PROJECT GOALS

    • When I was first gifted the Dell R740XD server by a dear professor who has since moved back to China, I knew I had to put it to good use.  

    • With 40 logical cores and 188 GB of RAM, I knew the possibilities were endless, but I really had no idea what to do with this as it was massively overkill for anything I ever needed.

    • Originally I decided the purpose of the Dell R740XD machine was to function as a VPN server and a file server, so that I may mount a network file share to any of my devices' filesystems and access my data from anywhere in the world; so far I have not strayed too far from this original goal. 

        * to get a visual reference of my setup so far, please refer to the "architecture.txt" file in the "Homelab-Infrastructure" directory

    • I will continually update this README as I add new services to this machine.  With that being said, equipped with nothing but the internet and sheer determination, I set out to build and document my first homelab ...

## CONFIGURATIONS

### Creating the Wireguard VPN

    • After wiping the original OS and replacing it with Proxmox, the first thing I needed to do was configure a secure VPN to my home network.  

    • I decided to host an LXC container running Wireguard for its customizablilty and zero dependence on third-party authentication providers or centralized management platforms.

    • The first thing I did was configure the Wireguard VPN on the server LXC container:

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

    • Then I did the same on my laptop:

        [Interface]
        PrivateKey = XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXAAA=
        Address = 10.0.0.2/32
        MTU = 1420
        DNS = 1.1.1.1 

        [Peer]
        PublicKey = 8Tsc+Fk1HKA0VM1tNIhTRqy2USny3sryiu7+4hnlaDE=
        AllowedIPs = 0.0.0.0/0
        Endpoint = [PUBLIC IP]:51820
        PersistentKeepalive = 21 

### Configuring NFS

    • Next, I needed to set up the nfs (network filesystem) server on the Dell machine, and configure my PC as the sole nfs client.

        * it should be noted for context that during the Proxmox installation, I selected zfs (RAID 0) for the root file system, which creates a default zfs pool named "rpool"
        * a dataset in zfs is a logical filesystem created inside a storage pool
        * zfs is an extremely efficient filesystem that uses a copy-on-write resource management technique and comes with built in volume managment capabilites
    
    • I began by creating the dataset named "storage", which I mounted to /media/storage; the output of the "zfs list" command on the server is shown below:

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

        * the storage dataset was created via the "zfs create rpool/storage" command

    • Now that the dataset that will hold on my files is created, I needed nfs to actually export this directory:

        root@sams-server:~# zfs set mountpoint=/media/storage rpool/storage 

    • To prevent the dataset from consuming the entire 1.74 TB system pool should it grow too large (not that it ever will), I set a 500 GB cap, or "quota" on the dataset:

        root@sams-server:~# zfs set quota=500G rpool/storage

    • The last server-side configuration was setting up the host to export the dataset exclusively to your local subnet by modifying the /etc/exports configuration file:

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

    • After exporting the share (/media/storage) and starting the service, I was good to go:

        root@sams-server:~# exportfs -rv
        root@sams-server:~# systemctl enable --now nfs-kernel-server

    • On the NFS client (My PC), I installed both nfs and autofs, so that the filesytem will be mounted automatically:

        samuel@sam-pc:~$ sudo apt install nfs-common autofs

    • Next, I edited the autofs master file which specifies which map files the autofs daemon should read:

        #
        # Sample auto.master file
        # This is a 'master' automounter map and it has the following format:
        # mount-point [map-type[,format]:]map [options]
        # For details of the format look at auto.master(5).
        #
        /mnt /etc/auto.nfs --timeout=180 browse

        * here, the /mnt tells autofs that it manages the entire /mnt parent directory as a base folder
        * /etc/auto.nfs is the actual map file containing the relative paths to build inside /mnt
        * the timeout specifies how long the nfs share should be mounted before it is unmounted due to inactivity
        * the "browse" keyword I found was very helpful as it keeps the subfolder (like /mnt/nfs) visible in ls or the GUI file manager even when the NFS share is currently unmounted

    • The last file to edit was the map file itself (/etc/auto.nfs):

        nfs -fstype=nfs,rw,soft,intr 192.168.4.27:/media/storage

        * where nfs is the local mount on my PC for the nfs file share, and 192.168.4.27:/media/storage is the server IP address and filepath to the actual files being exported

    • Of course we can't forget ...

        samuel@sam-pc:~$ sudo systemctl enable --now autofs


## 3. CISCO CML