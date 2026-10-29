## InstallingK8s
    Installing k8s using kubeadm
## Install container first 
    mhasifkamal@k8s:~$ date
    Thu Oct 29 11:11:31 PM IST 2026

## Check port is open in order for Kubernetes components to communicate with each other.
    mhasifkamal@k8s:~$ nc 127.0.0.1 6443 -zv -w 2
    nc: connect to 127.0.0.1 port 6443 (tcp) failed: Connection refused

## Disable swap
    mhasifkamal@k8s:~$ sudo swapoff -a
    mhasifkamal@k8s:~$ free -mh
                total        used        free      shared  buff/cache   available
    Mem:           1.6Gi       319Mi       1.1Gi       5.2Mi       274Mi       1.3Gi
    Swap:             0B          0B          0B
    mhasifkamal@k8s:~$ cat /etc/fstab
    # /etc/fstab: static file system information.
    #
    # Use 'blkid' to print the universally unique identifier for a
    # device; this may be used with UUID= as a more robust way to name devices
    # that works even if disks are added and removed. See fstab(5).
    #
    # <file system> <mount point>   <type>  <options>       <dump>  <pass>
    # / was on /dev/ubuntu-vg/ubuntu-lv during curtin installation
    /dev/disk/by-id/dm-uuid-LVM-9XVoZtgTFodSmlVurEsEafIXLDkHU7quIXzQ1VzuOKFbhZGmhYfUmgMlP4Kmhu3e / ext4 defaults 0 1
    # /boot was on /dev/vda2 during curtin installation
    /dev/disk/by-uuid/11e3d738-0663-42e8-85c3-e3552d7fdd20 /boot ext4 defaults 0 1
    # /boot/efi was on /dev/vda1 during curtin installation
    /dev/disk/by-uuid/9CD9-DEB8 /boot/efi vfat defaults 0 1
    /swap.img	none	swap	sw	0	0
    mhasifkamal@k8s:~$ sudo vi /etc/fstab


![Image] (Screenshot 2026-10-29 at 11.22.18 PM.png)