# depenguin.me mfsbsd-script
depenguin.me installer script for mfsBSD image to install FreeBSD 13.2 (with zfs-on-root) using qemu

https://depenguin.me

## Install FreeBSD-13.2 on a dedicated server from a Linux rescue environment

### 1. Boot into rescue console

You must be logged in as root. Prepare file path or URL of SSH public key.

### 2. Download and run installer script
Boot your server into rescue mode, then download and run the custom [mfsBSD-based installer](https://github.com/depenguin-me/depenguin-builder) for FreeBSD-13.2, with root-on-ZFS.

    wget https://depenguin.me/run.sh && chmod +x run.sh && \
      ./run.sh [ -d ] [ -r ram ] [ -m <url of own mfsbsd image> ] authorized_keys ...

The "-d" parameter will send the qemu process to the background.

The "-r" parameter allows setting qemu memory for low memory systems, default is `8G` for `8GiB`.

The "-m" parameter allows using a custom mfsbsd ISO.

You must specify at least one authorized_keys source, both URLs and local files are supported.

    note: run.sh on the website is a symlink to the depenguinme.sh script

Example invocations:

    ./run.sh https://example.org/mypubkey
    ./run.sh /tmp/my_public_key

### 3. Connect via SSH
Wait until the script reports SSH to be available (takes a few minutes), then connect:

    ssh -i /path/to/privkey -p 1022 mfsbsd@your-host-ip

Once logged in, you can `sudo su -` to root without a password. You cannot login as root.

If you have trouble with the ssh connection, wait 2 minutes and try again.

### 4. [Optional] Some UEFI systems may need edits to /etc/fstab
Before FreeBSD 13.2, `bsdinstall` added the efi boot partition to /etc/fstab using the device name visible in QEMU, which may differ (see issues [10](https://github.com/depenguin-me/depenguin-run/issues/10#issuecomment-1225893163) and [57](https://github.com/depenguin-me/depenguin-run/issues/57#issuecomment-1280604676)). This can be rectified by changing the device name of the efi partition to `/dev/gpt/efiboot0` (you can check the gpt label using `gpart show -l`):

```
# Device                Mountpoint      FStype  Options         Dump    Pass#
/dev/efiboot0            /boot/efi       msdosfs rw              2       2
/dev/mirror/swap.eli            none    swap    sw              0       0
```

_Note: Since FreeBSD 13.2, bsdinstall uses the gpt label when creating /etc/fstab by default (see [this commit](https://cgit.freebsd.org/src/commit/?id=7919c76dbdd20161247d1bfb647110d87ca5ee0f)), so editing it manually is not required anymore._

### 5. [Optional] Disable serial ports

FreeBSD hangs on some ASUS boards on boot if serial ports are enabled (see issue [10](https://github.com/depenguin-me/depenguin-run/issues/10)). To work around this problem, you can either disable serial ports in the BIOS or, more easily, disable them in /boot/loader.conf:

```
hint.uart.0.disabled="1"
hint.uart.1.disabled="1"
```

### 6. Install FreeBSD-13.2 using unattended bsdinstall
Copy the file `depenguin_settings.sh.sample` to `depenguin_settings.sh` and edit for your server's details.

    cp depenguin_settings.sh.sample depenguin_settings.sh
    nano depenguin_settings.sh

Configure your specifics, note that Hetzner DNS is in this example, you might need other servers listed.

    conf_hostname="depenguintest"
    conf_interface="igb0"
    conf_ipv4="1.2.3.4"
    conf_ipv6="abcd:xxxx:yyyy:zzzz::p"
    conf_gateway="6.7.8.9"
    conf_nameserveripv4one="185.12.64.1"
    conf_nameserveripv4two="185.12.64.2"
    conf_nameserveripv6one="2a01:4ff:ff00::add:1"
    conf_nameserveripv6two="2a01:4ff:ff00::add:2"
    conf_username="myusername"
    conf_pubkeyurl="http://url.host/keys.txt"
    conf_disks="ada0 ada1" # or ada0 | or nvme0n1 | or nvme0n1 nvme1n1
    conf_disktype="mirror" # or stripe for single disk
    run_installer="1" # set to 1 to enable installer 

### 7. Run the depenguin bsdinstall script
This script will update the `INSTALLERCONFIG` file used by `bsdinstall` with the values set above.

    ./depenguin_bsdinstall.sh 

When complete the mfsbsd VM will shutdown automatically.

### 8. Reboot
Switch to the rescue console session and press `ctrl-c` to end qemu. Then type `reboot`. 

### 9. Connect to your new server
After a few minutes to boot up, connect to your server via SSH:

    ssh YOUR-USER@ip-address

Check DNS is available and then perform initial system configuration such as:

    freebsd-update fetch
    freebsd-update install

End
