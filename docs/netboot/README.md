Netbooting Configuration for HW CI Devices
==========================================
*~10 min read*

Objective and Background
------------------------
This post explains how to configure the computer used for network booting devices. The goals are to have this happen automatically in the Hardware CI Lab. This is based on experiences booting FreeBSD on the Pine A64-LTS (64-bit ARM), the BeagleBone Black (32-bit ARM) and the Jetson TK1 (32-bit ARM) over the network.

Tools and Requirements
----------------------
### Assumptions
This post is based on devices that have the support for:
 * Ethernet connection
 * Serial connection

As the number of devices increases, you will want to use either a network switch or [bridge the connections](https://www.freebsd.org/doc/en/books/handbook/network-bridging.html) in order to easily add/remove devices.

### Packages
 * `net/isc-dhcp42-server`
 * `sysutil/u-boot-<device_name>`
     * If exists and preinstalled u-boot is insufficient

Getting Started
---------------
### Connecting to the Serial Port
You should integrate the device into [the power controller](/WaraH18ZQK2LgxyFFQxJOA) before starting. As a result, you should have added a line like 
```
console slot# {
    aliases <device_name>;
    include device serial; port #;
}
```

in which case connection to the device is as simple as using `console <device_name>`.

However, if you don't have a power controller/conserver setup then you can alternatively use `cu -l /dev/cuaU# -115200`. Where `cuaU#` is the device that was added when you plugged in the serial connection.
> **WARNING**
> 
> If you have permissions issues when using `cu` for the first time, then ensure you are part of the `dialer` group.

When everything is configured, you'll want to connect to the serial port before powering on the device.
### Cross compiling FreeBSD
While this isn't required for any given device, it is helpful to have a FreeBSD compiled and ready to install once everything is set up. Ultimately, this will be handled by Jenkins. 

> **NOTE**
> 
> This flow is based on the one that Jenkins uses. It is nice if you don't already know where you are going to be installing the files. If you do know you may want to look into using `make installworld distribution installkernel` instead of using the release folder.



In the project directory (i.e. `/project/<device_name>`), download FreeBSD using Git or SVN:
> **NOTE**
> 
> The below commands download FreeBSD-CURRENT but you may want to get a more stable build for initial testing.

```bash
git checkout https://github.com/freebsd/freebsd/ src 
```
or 
```bash
svnlite checkout https://svn.freebsd.org/base/head/ src 
```

let's say you want to add a new `arm64` device. These devices can all use the `GENERIC` kernel configuration (unlike some `arm` devices). However, since we need to netboot it, you'll need to create an NFS configuration. Save this file under `src/sys/<target_arch>/conf`.
> **NOTE**
> 
> In the case where you need to use a device specific config (i.e. `BEAGLEBONE` you can simply replace `GENERIC` with desired kernel configuration name).

```
#
# GENERIC-NFS -- Extends GENERIC to include NFS support
#
include GENERIC

ident   GENERIC-NFS

# NFS Support
options NFSCL
options NFSLOCKD

# NFS Root Support
options NFS_ROOT
options BOOTP_NFSROOT
options BOOTP_COMPAT
options BOOTP
options BOOTP_NFSV3
#options BOOTP_WIRED_TO=ue1
```

> **NOTE**
> 
> More information about the options used to enable NFS support can be found in `sys/conf/NOTES`


Then cross build FreeBSD you can run the following in the project directory (i.e. `/project/<device_name>`):

```bash
export BASEDIR=$(pwd)
export JFLAG=$(sysctl -n kern.smp.cpus) # may want to use a smaller number
export TARGET=<TARGET>
export TARGET_ARCH=<TARGET_ARCH>
export SRCCONF=/dev/null
export MAKECONF=/dev/null
export MAKEOBJDIRPREFIX=${BASEDIR}/obj

export KERNCONF=GENERIC-NFS # or whatever the kernel config was called
rm -fr ${MAKEOBJDIRPREFIX} # only include this line if you always want a clean build.

cd src

make -j ${JFLAG} -DNO_CLEAN \
    buildworld buildkernel \
    TARGET=${TARGET} \
    TARGET_ARCH=${TARGET_ARCH} \
    __MAKE_CONF=${MAKECONF} \
    SRCCONF=${SRCCONF}

cd release

sudo -E make clean
sudo -E make -DNOPORTS -DNOSRC -DNODOC packagesystem \
    TARGET=${TARGET} TARGET_ARCH=${TARGET_ARCH} \
    MAKE="make -DDB_FROM_SRC __MAKE_CONF=${MAKECONF} SRCCONF=${SRCCONF} KERNCONF=${KERNCONF}"

RELEASEDIR=${BASEDIR}/release
rm -fr ${RELEASEDIR}
mkdir -p ${RELEASEDIR}
mv ${MAKEOBJDIRPREFIX}/${BASEDIR}/src/${TARGET}.${TARGET_ARCH}/release/*.txz ${RELEASEDIR}
mv ${MAKEOBJDIRPREFIX}/${BASEDIR}/src/${TARGET}.${TARGET_ARCH}/release/MANIFEST ${RELEASEDIR}


cd ../..
```

Installing these "release" files will be discussed later once the NFS root directory is set up.

Initial Setup
-------------
### Configuring the TFTP Server
In `/etc/inetd.conf`, uncomment the line that starts with `tftp dgram udp wait root ....`.

Then, change the path at the end to the directory where you store bootable files. In my case, it was `/b/tftpboot`. For the remainder of this post, it will be referred to as `$TFTPROOT`

Run the following to enable inetd and start it.
```bash
sudo sysrc inetd_enable=YES
sudo service inetd start
```

#### Testing it out
Create a test file like `$TFTPROOT/testfile` with some dummy content and then run `tftp`. You should get a prompt like
```
tftp>
```
in this shell run something like the following to 
```
connect localhost
get testfile
```
You should see something like `Received ## bytes during #.# seconds in # blocks` which means it is working.

### Configuring the DHCP Server
You'll want to create a DHCP configuration. As below is a sample that adds three devices (on IPs `10.0.0.10` to `10.0.0.12`) and connects them to the local TFTP server (running at `10.0.0.1`). The following `/usr/local/etc/dhcpd.conf` should work out of the box, and all adding ~250 devices (which is probably a lot more than you'd need) on the addresses `10.0.0.2` to `10.0.0.254`.

> **NOTE**
> 
> More information about DHCPD configurations can be found in [the handbook](https://www.freebsd.org/doc/en/books/handbook/network-dhcp.html)


```
# dhcpd.conf
#
# Sample device configuration file for ISC dhcpd
#

option root-opts code 130 = string;

log-facility local7;

group {
    option root-opts "nolockd";
    option routers 10.0.0.1;
    
    subnet 10.0.0.0 netmask 255.255.255.0 {
      next-server 10.0.0.1;
    }

    #host pinea64 {
    #  hardware ethernet 02:ba:fb:ce:73:7e;
    #  fixed-address 10.0.0.11;
    #  filename "pinea64/install/boot/loader.efi";
    #  option root-path "/b/tftpboot/pinea64/install/";
    #}

    #host beaglebone {
    #  hardware ethernet c8:df:84:da:65:00;
    #  fixed-address 10.0.0.12;
    #  filename "beaglebone/install/boot/loader.efi";
    #  option root-path "/b/tftpboot/beaglebone/install/";
    #}
}
```

#### Option Description
`option root-opts code 130 = string;`
: Enables the `root-opts` option, so you can specify it later

`log-facility local7`
: Configures where logs will go. You could specify a file here instead like `/var/log/dhcpd.log`. (optional)

`subnet <subnet_id> netmask <netmask>`
:  Create a subnet with the id `<subnet_id>` and will include addresses based on `<netmask>`. You can determine how many addresses are included in the subnet by using. 
   > **NOTE**
   > 
   > You can use calculator tools like [this](http://www.subnet-calculator.com/) to see which range of addresses the subnet makes available. In my case, I was using "Network Class A" with 254 "Hosts per subnet". Note that the TFTP server also counts as a host, so this will allow you to add up to 253 devices.


`group`
:  Allows you to define options for a group of hosts/subnets. Saves you typing.

`option root-opts nolockd`
:  Disables the `lockd` service for NFS directories. 

`option routers <tftp_server_ip>`
:  This will be the host computer's IP on this subnet. This will make the TFTP server available for downloading files and also for using NFS. This should be the first host IP in the subnet.

`next-server <tftp_server_ip>`
:  The TFTP server that will be used on this subnet. This should match the value of `option routers`

`host <device_name>`
: Defines options and configutations for DHCP for a specific device

`hardware ethernet <MAC_address>`
:  The MAC address of the device. This can usually be found in u-boot by running `printenv ethaddr`. Otherwise, you may have to look for a sticker or manual for the device's MAC address.

`fixed-address <device_ip>`
: The IP address the device should use. This is arbitrary but should be with the subnet and shouldn't coincide with either the TFTP server (the lowest address, in my case `10.0.0.1`) or the broadcast address (the highest address, in my case `10.0.0.255`)

`option filename`
: The file that will be downloaded by default when `dhcp` is called from u-boot. For most devices, this should `${TFTPROOT}/<device_dir>/boot/loader.efi`

`option root-path`
:  Provides the `NFS` root directory to use for this device. This is where FreeBSD should be installed. From here on, this will be referred to as `$NFSROOTDIR`. For simplicity, this directory should be under the `$TFTPROOT` chosen earlier.
    > **NOTE**
    > 
    > The default scripts expect `$NFSROOTDIR` to be `$TFTPROOT/$DEVICE/install`



To start the dhcp service run the following
```bash
sudo sysrc dhcpd_enable=YES
sudo service dhcpd start
```
Additionally, you should restart the service after modifying this file (i.e. when you add a new device). 

### Configuring the NFS Server
You'll need a `/etc/exports` file to expose the NFS directories. Since all files under `/b` or `$TFTPROOT` should accessible over NFS and we also want these file systems to be modifiable by the device, the following simple `/etc/exports` file works well.
```
/b -alldirs -maproot=root
```
> **NOTE**
> 
> The [manpage](https://www.freebsd.org/cgi/man.cgi?query=exports&sektion=5) for `exports(5)` is helpful if you want more granular options such as limiting only certain IPs (devices) to use certain NFS directories and additionally making them read-only.



Run the following to enable and start the NFS service and its dependencies:
```bash
sudo sysrc rpcbind_enable=YES
sudo sysrc rpc_statd_enable=YES
sudo sysrc rpc_lockd_enable=YES
sudo sysrc nfs_server_enable=YES
sudo sysrc mountd_enable=YES

sudo service rpcbind start
sudo service nfsd start
```

#### Installing Files to the NFS Directory
Assuming you have files in `$RELEASEDIR` that are ready to be installed into `$NFSROOTDIR` for this device. You can install the files by running the following (based on what's done by Jenkins):
```bash
# clean up destdir
sudo mkdir -p ${NFSROOTDIR}
sudo chflags -R noschg ${NFSROOTDIR}
sudo rm -rf ${NFSROOTDIR}/*

# install files
for COMPONENT in ${RELEASEDIR}/*.txz; do
    sudo tar Jxf ${COMPONENT} -C ${NFSROOTDIR}    
done

# Make DTB files available to u-boot
sudo mkdir -p ${TFTPROOT}/dtb
sudo cp -r ${NFSROOTDIR}/boot/dtb ${TFTPROOT}/dtb
```

You may also want to add a `/etc/fstab`, `/boot/loader.conf` and `/etc/rc.conf` files for improvements to booting. See [freebsd-ci/scripts/test/extract-artifacts.sh](https://github.com/GeraldNDA/freebsd-ci/blob/master/scripts/test/extract-artifacts.sh) for examples.

New Device Setup
----------------
All that needs to be updated with new devices is the `dhcpd.conf` by adding a new `host` definition to the group and defining the required variables.

For example, adding the following would add a specific Pine A64-LTS device to the dhcpd.
```
host pinea64 {
  hardware ethernet 02:ba:fb:ce:73:7e;
  fixed-address 10.0.0.11;
  filename "pinea64/install/boot/loader.efi";
  option root-path "/b/tftpboot/pinea64/install/";
}
```

Device Specific Boot
--------------------
Depending on the device, you may have to use a slightly different process to load the `loader.efi` file and the correct DTB (Device Tree Blob) file. 

For devices that have u-boot preinstalled, you'll need to ensure `bootefi` is an available command. If so then `bootcmd_dhcp` should attempt to run `dhcp` or `tftpboot` with no arguments or a single argument (i.e. `$kernel_addr_r`) to download the EFI file and something like `dhcp $fdt_addr_r $fdtfile` or `tftpboot $fdt_addr_r $fdtfile` to load the DTB . If not, you can use the following lines to change the command to something similar to below to enable this behaviour.
```bash
setenv bootcmd_dhcp <any_initializatin_commands>\; if dhcp ${kernel_addr_r}\; then tftpboot ${fdt_addr_r} dtb/${fdt_file}\; if fdt addr ${fdt_addr_r}\; then bootefi ${kernel_addr_r} ${fdt_addr_r}\; fi\;  fi\;
```
> **WARNING**
> 
> Before changing this variable permanently (using `saveenv`) you may want to verify that any initialization that is normally done in this command (i.e. `usb start`) is still included.



If you're device does not have u-boot preinstalled or it doesn't have the `bootefi` command you'll need to update u-boot. This is different depending on the device, however, you'll probably won't have to recompile it. The new version of u-boot should be available from ports or packages at `sysutils/u-boot-<device_name>`. If so, the files needed for installing u-boot will be available in `/usr/local/share/u-boot/u-boot-<device_name>`.

The default u-boot installation is very likely to try `dhcp` booting last which will slow down the boot process. This can be reconfigured by modifying the `boot_targets` variable
```bash
setenv boot_targets dhcp $boot_targets
```
> **WARNING**
> 
> After changing any environment variables use `saveenv` to make it permanent (after reboot). Use with care!

