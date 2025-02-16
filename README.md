<h1 align="center"> Linux-based CPE for Telia's (AS3249) "Koduinternet" service </h1>

<h2 id="table-of-contents"> :book: Table of Contents</h2>
<details open="open">
  <summary>Table of Contents</summary>
  <ol>
    <li><a href="#about-the-project"> ➤ About The Project</a></li>
    <li><a href="#hardware"> ➤ Hardware</a></li>
    <li><a href="#configuring-nic-nvram"> ➤ Configuring NIC NVRAM</a></li>
    <li><a href="#details-nic-nvram"> ➤ Details of Configuring NIC NVRAM</a></li>
    <li><a href="#modifying-the-nic-driver"> ➤ Modifying the NIC driver</a></li>
    <li><a href="#rooting-the-sfp-ont"> ➤ Rooting the SFP ONT</a></li>
    <li><a href="#modifying-the-sfp-ont-serial"> ➤ Modifying the SFP ONT serial</a></li>
    <li>
      <a href="#debian-10-config"> ➤ Debian 10 config</a>
      <ul>
        <li><a href="#udev-usb_modeswitch-config">udev and usb_modeswitch config</a></li>
        <li><a href="#network-config">network config</a></li>
        <li><a href="#iptables">iptables</a></li>
        <li><a href="#dnsmasq">dnsmasq</a></li>
        <li><a href="#radvd">radvd</a></li>
        <li><a href="#igmpproxy">igmpproxy</a></li>
        <li><a href="#ntp-config">NTP config</a></li>
        <li><a href="#hostapd">hostapd</a></li>
        <li><a href="#linux-pam-config">Linux PAM config</a></li>
      </ul>
    </li>
    <li><a href="#bandwidth-tests"> ➤ Bandwidth tests</a></li>
    <li>
      <a href="#hardware-mods"> ➤ Hardware mods</a>
      <ul>
        <li><a href="#nic-fan-replacement">Replacing the Dell N20KJ stock fan with Noctua NF-A4x10 fan</a></li>
      </ul>
    </li>
    <li><a href="#acknowledgements"> ➤ Acknowledgements</a></li>
    <li><a href="#license"> ➤ License</a></li>
  </ol>
</details>

<h2 id="about-the-project"> :pencil: About The Project</h2>
<p align="justify">
Debian based router on conventional PC hardware and SFP GPON ONT replacing the <a href="https://pood.telia.ee/ruuterid-ja-digiboksid/ruuter-Telia-X2-Technicolor-DGA4330/DGA4330-TELIA">Technicolor</a> or <a href="https://pood.telia.ee/ruuterid-ja-digiboksid/ruuter-Genexis-Pure-ED500/ED500A-TELIA">Genexis</a> router and stand-alone <a href="https://pood.telia.ee/vorguseadmed-ja-lisad/v%C3%B5rguseade-Huawei-HG8010H/HG8010H-TELIA">Huawei ONT</a> provided by ISP.
</p>

["Configuring NIC NVRAM"](#configuring-nic-nvram) and ["Modifying the NIC driver"](#modifying-the-nic-driver) steps are needed if `2500BASE-X` link between the NIC and SFP ONT is desired instead of `1000BASE-T` link. ["Rooting the SFP ONT"](#rooting-the-sfp-ont) and ["Modifying the SFP ONT serial"](#modifying-the-sfp-ont-serial) steps are mandatory because Telia authenticates ONT by its serial number.

<h2 id="hardware"> :clipboard: Hardware</h2>

```
CPU: Intel Core i7-4790K @ 4.00GHz (Cooler Master Hyper 212 EVO cooler)
MB: ASRock H97 Anniversary
RAM: Crucial 8 GiB CT102464BA160B.C16FER
SSD: Corsair Force LS 2.5" 60 GB SATA3
NIC: Dell N20KJ; Broadcom BCM57810 (wan0 and lan0; driver bnx2x; firmware /usr/lib/firmware/bnx2x/bnx2x-e2-7.13.1.0.fw; NF-A4x10 fan)
NIC: integrated; Realtek RTL8111/8168/8411 (lan1; driver r8169; firmware /usr/lib/firmware/rtl_nic/rtl8168g-2.fw)
NIC: TP-LINK TG-3468; Realtek RTL8111/8168/8411 (lan2; driver r8169; firmware /usr/lib/firmware/rtl_nic/rtl8168e-2.fw)
NIC: TP-LINK TG-3468; Realtek RTL8111/8168/8411 (lan3; driver r8169; firmware /usr/lib/firmware/rtl_nic/rtl8168e-2.fw)
WNIC: Asus PCE-AC88; Broadcom BCM4366(v4) (wlan0; driver brcmfmac; firmware /usr/lib/firmware/brcm/brcmfmac4366c-pcie.bin)
LTE modem: Huawei E3372s-153 (hw ver: CL1E3372SM Ver.A; firmware ver: 22.286.03.02.07)
SFP ONT: Huawei MA5671A
PSU: Chieftec CTG-500-80P
case: Chieftec 1E0-500A-CT04
```

![router with side panel removed](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/router_with_side_panel_removed.jpg)


<h2 id="configuring-nic-nvram"> :small_blue_diamond: Configuring NIC NVRAM</h2>

Both the <a href="https://github.com/tonusoo/koduinternet-cpe/blob/main/docs/hw_415752.pdf">Huawei MA5671A SFP ONT</a> and <code>Dell N20KJ</code> NIC using the <code>Broadcom BCM57810</code> chipset support the 2500BASE-X mode. According to <a href="https://github.com/tonusoo/koduinternet-cpe/blob/main/docs/netxtreme_ii.pdf">Broadcom NetXtreme II Network Adapter User Guide</a>, the 2500BASE-X is a term used by Broadcom to describe 2.5 Gbit/s operation, where electricals are leveraged from IEEE 802.3ae-2002 (XAUI).


By default, the `Dell N20KJ` branded NIC supports 1GigE and 10GigE modes. One needs to configure the NIC's NVRAM with [QLogic BCM577xx/BCM578xx diagnostics utility named eDiag](https://github.com/tonusoo/koduinternet-cpe/raw/main/uefi_ediag.tgz) in order to enable the 2500BASE-X mode. For example, the NIC can be passed through to a VM where `ediag_x64.efi` UEFI binary in engineering mode(`-b10eng`) is executed:

![UEFI shell in qemu VM](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/UEFI_shell_in_qemu_VM.jpg)

Command for creating the VM in the upper `tmux` pane was:
```
# PCIe passthrough requires IOMMU support.
# "OVMF.fd" UEFI is from the "ovmf" package.
qemu-system-x86_64 \
    -name eDiag-VM \
    --enable-kvm \
    -m 1G \
    -bios /usr/share/ovmf/OVMF.fd \
    -serial telnet:127.0.0.1:5016,server,nowait \
    -nographic \
    -drive file=fat:rw:UEFI/uefi_ediag/x64/,format=raw \
    -device vfio-pci,host=01:00.0,rombar=0 \
    -device vfio-pci,host=01:00.1,rombar=0
```

As seen on the lower `tmux` pane, the first ports is set to `2.5G` mode:
![UEFI eDiag in qemu VM](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/UEFI_eDiag_in_qemu_VM.jpg)

`2500BASE-X` mode was enabled and set as a default for the first port with `eDiag` CLI commands below:
```
device 1
nvm cfg
7
35=70
36=70
56=6
59=6
save
exit
```

Correct link settings for the first port can be seen below:
![UEFI eDiag in qemu VM 2.5G settings](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/UEFI_eDiag_in_qemu_VM_2_5G_settings.jpg)

<h2 id="details-nic-nvram"> :small_blue_diamond: Configuring NIC NVRAM</h2>

Hello Martin, I had questions after reading configuring-nic-nvram.

The questions I had were:

* What does qemu mean when it says that /dev/vfio/35 does not exist?

When qemu complains that some integer under /dev/vfio/ does not exist, it means that you have not yet correctly configured the NIC for use by the vfio-pci kernel module.  Continue reading.

* How do I keep bnx2x from loading in the hypervisor so that the device can be accessed properly in the guest VM?

In order to keep the bnx2x module, which is installed by default on debian, from loading at boot time, consider this:

```
$ cat /etc/modprobe.d/00-netextreme-vfio.conf 
install bnx2x /bin/true
options vfio-pci ids=14e4:168e
softdep bnx2x pre: vfio-pci
```

* How does one configure iommu for a 14e4:168e ?

To enable iommu, append some command line arguments to the kernel line.
```
$ grep iommu /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="kvm-intel.nested=1 intel_iommu=on vfio_pci.ids=15b3:1003 modprobe.blacklist=bnx2x bnx2x.blacklist=1 rd.driver.blacklist=bnx2x brd rd_size=33554432 console=tty0 console=ttyS1,38400n8 cgroup_enable=memory swapaccount=1"
```

* What even is a /sys/bus/pci/drivers/vfio-pci/new_id and why do we echo make and model to it?

I don't know what `/sys/bus/pci/drivers/vfio-pci/new_id` is, but IBM tells me that in order to bind the driver to a device on the pci(e) bus, the manufacturer id and product id without colon separation should be echoed to it.  Probably right after the system boots up is best.

```
$ grep vfio-pci /etc/rc.local
echo 14e4 168e > /sys/bus/pci/drivers/vfio-pci/new_id
```

* Do I need to update my initrd or grub configuration for these changes to take effect on the next reboot?

Good thinking!  Yes, update your grub configuration and re-build your initrd.  While you're at it, you might as well install the latest kernel.  And power all the way off and remove the power cables from the PSU for the count of five and then boot back up again.

```
$ sudo -i
# update-grub ; apt-get update ; apt-get install --reinstall -t bookworm-backports linux-image-amd64 ; update-initramfs -u
# shutdown -h now
```

* How do I confirm that the device is ready for passthrough to qemu?

When your device is correctly configured, `lspci -v` will show `vfio-pci` in the `Kernel driver in use` field of its output.

```
$ lspci -n -v -s 02:00
02:00.0 0200: 14e4:168e (rev 10)
        Subsystem: 14e4:1006
        Physical Slot: 5
        Flags: bus master, fast devsel, latency 0, IRQ 16, NUMA node 0, IOMMU group 35
        Memory at db000000 (64-bit, prefetchable) [size=8M]
        Memory at db800000 (64-bit, prefetchable) [size=8M]
        Memory at dc010000 (64-bit, prefetchable) [size=64K]
        Expansion ROM at d9c00000 [virtual] [disabled] [size=512K]
        Capabilities: <access denied>
        Kernel driver in use: vfio-pci
        Kernel modules: bnx2x

02:00.1 0200: 14e4:168e (rev 10)
        Subsystem: 14e4:1006
        Physical Slot: 5
        Flags: bus master, fast devsel, latency 0, IRQ 17, NUMA node 0, IOMMU group 35
        Memory at da000000 (64-bit, prefetchable) [size=8M]
        Memory at da800000 (64-bit, prefetchable) [size=8M]
        Memory at dc000000 (64-bit, prefetchable) [size=64K]
        Expansion ROM at d9c80000 [virtual] [disabled] [size=512K]
        Capabilities: <access denied>
        Kernel driver in use: vfio-pci
        Kernel modules: bnx2x
```

* How do I attach to that serial terminal?

```
$ telnet 127.0.0.1 5016
```

* What does the scrollback buffer of the uefi serial console look like?

https://gist.github.com/cjac/e17d47676d1dfc26fbf65fe15116295c



<h2 id="modifying-the-nic-driver"> :small_blue_diamond: Modifying the NIC driver</h2>

`bnx2x` driver was [patched](https://github.com/tonusoo/koduinternet-cpe/blob/main/patches/bnx2x_link.patch) in order to support the 2500BASE-X mode and ignore the Tx fault:

![bnx2x_link.c diff](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/bnx2x_link_diff.jpg)

Without ignoring the Tx fault, the Ethernet link would drop down if the fiber to `MA5671A SFP ONT` is not connected and ONT is not successfully registered by the OLT.

Example of taking the patched driver into use:

![patching the bnx2x driver](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/patching_the_bnx2x_driver.jpg)

For troubleshooting purposes, the driver can be loaded in debug mode with `modprobe -v bnx2x debug=0x4102034`.

`2500BASE-X` link between the `MA5671A SFP ONT` and `Dell N20KJ` NIC observed from the Linux router:

![MA5671A and 2500BASE-X link from r1](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/MA5671A_and_2500BASE-X_link_from_r1.jpg)

The same link mode seen from the non-rooted `MA5671A SFP ONT` `minishell` with the `lanpsg`(LAN port status get) command where `link_status` value `5` means `2500BASE-X`:

![MA5671A and 2500BASE-X link from non-rooted MA5671A](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/MA5671A_and_2500BASE-X_link_from_non-rooted_MA5671A.jpg)

Link mode checked from the rooted `MA5671A SFP ONT`:

![MA5671A and 2500BASE-X link from rooted MA5671A](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/MA5671A_and_2500BASE-X_link_from_rooted_MA5671A.jpg)

In case the `2500BASE-X` mode is not desired, then `Huawei MA5671A SFP ONT` links also in `1000BASE-T` mode:

![MA5671A and 1000BASE-T link from r1 and rooted MA5671A.jpg](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/MA5671A_and_1000BASE-T_link_from_r1_and_rooted_MA5671A.jpg)

Once the link is established, the SFP is accessible at 192.168.1.10 via SSH. User is `root` and password is `admin123`.


<h2 id="rooting-the-sfp-ont"> :small_blue_diamond: Rooting the SFP ONT</h2>

One option to root the `Huawei MA5671A SFP ONT` is to short the `GND`(pin `4`) and `Serial Data Input`(pin `5`) of the [flash memory](https://github.com/tonusoo/koduinternet-cpe/blob/main/docs/Infineon-S25FL129P_128-Mbit_3.0_V_Flash_Memory-DataSheet-v11_00-EN.pdf) and power on the SFP. The SFP will boot into bootrom CLI and one can transfer a modified boot loader([1224ABORT.bin](https://github.com/tonusoo/koduinternet-cpe/raw/main/1224ABORT.bin)) to flash chip with XMODEM file transfer protocol.

`Huawei MA5671A SFP ONT` with wires soldered to `GND` and `SI` pins:
![MA5671A with soldered wires to flash](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/MA5671A_with_soldered_wires_to_flash.jpg)

`Huawei MA5671A SFP ONT` connected to [Reveltronics adapter board](https://www.reveltronics.com/en/products/sfp-qsfp-xfp):
![MA5671A connected to reveltronics board](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/MA5671A_connected_to_reveltronics_board.jpg)

As seen above, the `RXD`, `TXD` and `GND` of the USB to TTL (3.3V) adapter are connected to transceivers adapter board.

Minicom(`115200 8N1`) screenshot of the `Huawei MA5671A SFP ONT` bootup where `GND` and `SI` pins of the flash were shorted:

![FALCON CLI after modified U-Boot upload](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/falcon_cli_after_modified_u-boot_upload.jpg)

Typing the `7` in the `ROM:` prompt started the XMODEM file transfer mode and `Ctrl + c` aborted the autoboot. Setting the variables below will keep the bootloader open after the SFP module restart:
```
FALCON => setenv bootdelay 5
FALCON => setenv asc 0
FALCON => setenv preboot "gpio input 105;gpio input 106;gpio input 107;gpio input 108;gpio set 3;gpio set 109;gpio set 110;gpio clear 423;gpio clear 422;gpio clear 325;gpio clear 402;gpio clear 424"
FALCON => saveenv
```

After rebooting the module, the `minishell` was replaced with `ash` shell with `sed -i  "s|/opt/lantiq/bin/minishell|/bin/ash|g" /etc/passwd`.

[1224ABORT.bin](https://github.com/tonusoo/koduinternet-cpe/raw/main/1224ABORT.bin)(MD5 `10e94a4b4acdc82dec20c7904b69e5c0`) is a slightly modified U-Boot version 1.22.4 binary image for `MA5671A`. Hex dumps diff between the `1224ABORT.bin` and [unmodified U-boot for this module](https://github.com/minhng99/alcatel_lucent-lantiq_falcon/blob/master/mtd/mtd0)(`Huawei MA5671A` is based on `Alcatel-Lucent G-010S-P`) can be seen below:

![1224ABORT.bin hex dump compared to vanilla U-boot](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/1224ABORT_hex_dump_compared_to_vanilla_U-boot.jpg)

Boot messages over serial of the rooted `MA5671A SFP ONT` can be seen [here](https://github.com/tonusoo/koduinternet-cpe/blob/main/docs/rooted_MA5671A_boot_messages.txt).

<h2 id="modifying-the-sfp-ont-serial"> :small_blue_diamond: Modifying the SFP ONT serial</h2>

GPON OLT does not register the new SFP ONT if its serial number differs from the serial number of the `Huawei HG8010H ONT` installed by Telia. In other words, the serial number of the ONT is used for authentication. Here is a screenshot of the OLT CLI from [Huawei forum](https://forum.huawei.com/enterprise/en/sfp-mini-ont-ma5671a-ploam-password/thread/686439-100181) showing the status of the ONT:

![Huawei GPON OLT CLI](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/Huawei_GPON_OLT_CLI.jpg)

The serial of the `Huawei HG8010H` provided by Telia can be seen from the web interface:

![HG8010H status](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/HG8010H_status.jpg)

Default username is `root` and password is `admin`. One could also log in as a privileged user named `telecomadmin` with password `admintelecom`. For example, this allows one to enable telnet access to `Huawei HG8010H ONT` by downloading its configuration file from the web interface, modifying the `TELNETLanEnable` parameter and uploading the configuration. The same serial from `Huawei HG8010H` CLI:

![HG8010H telnet CLI](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/HG8010H_telnet_cli.jpg)

The hex representation of the serial is also printed on the label under the `Huawei HG8010H` chassis and additionally on its cardboard box.

`MA5671A SFP ONT` stores the base64 encoded serial number in U-Boot environment(`/dev/mtd1` flash partition named `uboot_env`) variable called "sfp_a2_info". The "sfp_a2_info" variable contains multiple base64 encoded fields/lines which are separated with `@` character. For example, the MAC address is on the ninth field and GPON ONT password is on the fifth field. However, only the serial needs to be changed and this is on the sixth, 45 bytes long base64 encoded field:

![MA5671A serial in sfp_a2_info](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/MA5671A_serial_in_sfp_a2_info.jpg)

As seen above, the output of the `fw_printenv sfp_a2_info` was transferred to `r1` Debian machine where the sixth base64 field was decoded to binary and dumped to hex. In order to change the serial number for example from `HWTCa1b1c1d1` to `HWTCa2b2c2d2`, it's necessary to prepare a new value of "sfp_a2_info" variable:

![MA5671A new sfp_a2_info prepared](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/MA5671A_new_sfp_a2_info_prepared.jpg)

As a final step, the modified "sfp_a2_info" variable needs to be written to `/dev/mtd1` with `fw_setenv` and SFP ONT has to be rebooted so U-Boot reads the updated variable:

![MA5671A new_sfp_a2_info written](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/MA5671A_new_sfp_a2_info_written.jpg)

As mentioned before, the default password for `root` user is `admin123`. It's recommended to back up the `/dev/mtd1` and output of `fw_printenv` before the changes.

`MA5671A SFP ONT` registration status can be checked with the `onu ploamsg` command. Value of the `curr_state` has to be `5` which means `O5` or `operation-state`:

![MA5671A in operation state](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/MA5671A_in_operation_state.jpg)

The same `O5` value was seen on the screenshot of the `Huawei HG8010H ONT` [web interface](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/HG8010H_status.jpg). On lower `tmux` pane the optical Tx and Rx power reported by the `MA5671A SFP ONT`(plugged into `wan0` port) can be seen.

<h2 id="debian-10-config"> :small_blue_diamond: Debian 10 config</h2>

Configuration leans towards traditional or legacy way of doing things, e.g `iptables` over `nftables`, `udev` rules in `/etc/udev/rules.d/` for renaming network interfaces instead of `/etc/systemd/network/*.link` files, etc. This is an intentional design choice.

List of packages installed using `apt`:
```
firmware-bnx2x
firmware-realtek
firmware-brcm80211
usb-modeswitch
bridge-utils
radvd
hostapd
libpam-yubico
libpam-google-authenticator
```

<h3 id="udev-usb_modeswitch-config"> :black_small_square: udev and usb_modeswitch config</h3>

`Huawei E3372s-153` USB LTE modem has to be mode switched from `157d`(CD/DVD drive) to [modem mode](https://wiki.dd-wrt.com/wiki/index.php/3G_/_3.5G#Huawei) which is `14dc` for this particular device.

This is taken care of by following `udev` rule in `/usr/lib/udev/rules.d/40-usb_modeswitch.rules` file installed by `usb-modeswitch-data` package:
```
ATTRS{idVendor}=="12d1", ATTRS{manufacturer}!="Android", ATTR{bInterfaceNumber}=="00", ATTR{bInterfaceClass}=="08", RUN+="usb_modeswitch '/%k'"
```
The rule above executes the `/usr/lib/udev/usb_modeswitch` script with a "kernel name" for the device as an argument, e.g `/usr/lib/udev/usb_modeswitch 3-2:1.0`. The script starts an instance of the `/lib/systemd/system/usb_modeswitch@.service`, which in turn executes the `usb_modeswitch_dispatcher` binary with a `--switch-mode` option for a specified device. `usb_modeswitch_dispatcher` reads the configuration from the `/usr/share/usb_modeswitch/configPack.tar.gz` tarball:
```
root@r1:~# tar -xOzf /usr/share/usb_modeswitch/configPack.tar.gz 12d1:157d
# Huawei E3331, E3372
TargetVendor=0x12d1
TargetProductList="14db,14dc"
HuaweiNewMode=1
root@r1:~#
```
.. and calls the `usb_modeswitch` with the content of the appropriate configuration file as a value of the `--long-config` argument. However, the default configuration does not work if the `cdc_mbim` driver is loaded. As a workaround, `/etc/usb_modeswitch.d/12d1:157d` file is created with `NoMBIMCheck=1` configuration option added:
```
root@r1:~# cat /etc/usb_modeswitch.d/12d1:157d
# Huawei E3331, E3372
TargetVendor=0x12d1
TargetProductList="14db,14dc"
HuaweiNewMode=1
NoMBIMCheck=1
root@r1:~#
```
With the configuration above the `usb_modeswitch` called by `usb_modeswitch_dispatcher` makes the switch from `157d` to `14dc`:
```
root@r1:~# lsusb -d 12d1:
Bus 003 Device 042: ID 12d1:14dc Huawei Technologies Co., Ltd. E33372 LTE/UMTS/GSM HiLink Modem/Networkcard
root@r1:~#
```

`udev` rule named `70-persistent-net.rules` ensures the `wan*`, `wwan*`, `lan*` and `wlan*` interfaces naming convention:

![ip_link_brief](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/ip_link_brief.jpg)

Configuration files: [/etc/usb_modeswitch.d/12d1:157d](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/usb_modeswitch.d/12d1%3A157d), [/etc/udev/rules.d/70-persistent-net.rules](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/udev/rules.d/70-persistent-net.rules)


<h3 id="network-config"> :black_small_square: network config</h3>

Telia's service works in a way that the Internet traffic is untagged and IPTV traffic is in the VLAN(IEEE 802.1q) 4. Router has a `wan0` interface for untagged Internet traffic and a VLAN interface named `wan0.4` for IPTV. Both interfaces receive the IPv4 address and netmask via DHCP. `wan0` will get a publicly routable IPv4 address and IPTV interface will get a private IPv4 address. In addition, several v4 routes are installed by DHCP. Few /24 publicly routable networks and the 10/8 are routed via the IPTV interface. Default route is via the Internet interface `wan0`. IPv6 part works in a way that Telia delegates a /56 prefix for LAN hosts from their /32 allocation using a DHCPv6. Internet facing interface will not get an IPv6 address and routing works using the IPv6 link local addresses:
```
root@r1:~# # v6 default route via VRRP virtual router
root@r1:~# ip -6 r sh default
default via fe80::200:5eff:fe00:1 dev wan0 proto ra metric 1024 expires 3286sec hoplimit 64 pref medium
root@r1:~#
```

Additionally, the `wan0` interface has a manually configured address of `192.168.1.200/24` for accessing the `Huawei MA5671A SFP ONT`. `dhclient` enter hook script named [get-static-ipv4-addrs](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/dhcp/dhclient-enter-hooks.d/get-static-ipv4-addrs) and exit hook script named [restore-static-ipv4-addrs](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/dhcp/dhclient-exit-hooks.d/restore-static-ipv4-addrs) ensure that this address does not get flushed by `dhclient`.

`lan*` and `wlan*` interfaces are part of the Linux bridge named `br0`:
```
root@r1:~# bridge link
2: lan2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state forwarding priority 32 cost 4
3: lan3: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 master br0 state disabled priority 32 cost 4
4: lan1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 master br0 state disabled priority 32 cost 100
6: lan0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 master br0 state disabled priority 32 cost 100
47: wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state forwarding priority 32 cost 100
root@r1:~#
```
The `br0` interface has a manually configured IPv4 address `192.168.0.1/24`. IPv6 address on `br0` is managed by [dhclient exit hook script](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/dhcp/dhclient-exit-hooks.d/ipv6-pd-br0).

In addition, there is a `wwan0`(USB CDC Ethernet device) interface facing the Telia's mobile broadband network with static address of `192.168.8.200/24` and corresponding floating default route:
```
root@r1:~# ip r sh default dev wwan0
default via 192.168.8.1 metric 100
root@r1:~#
```
Switching the IPv4 connectivity to mobile network and back to fiber by adjusting the default routes metric is managed by [isp-switch](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/systemd/system/isp-switch.service) `systemd` service which calls the [isp-switch](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/usr/local/sbin/isp-switch) script. IPv6 is not supported by Telia for the mobile broadband service.

LAN hosts get the IPv4 address of the DNS server(`dnsmasq` running in the router) from DHCPv4 server(also `dnsmasq`). By default, the domain name server sent to DHCPv4 clients is set to the address of the interface running `dnsmasq` which is 192.168.0.1:
```
root@r1:~# ip -4 --brief a sh dev br0
br0              UP             192.168.0.1/24
root@r1:~#
```
IPv6 address of the DNS server(`dnsmasq` running in the router) is propagated by NDP "Router Advertisement" messages sent by `radvd`. RDNSS(Recursive DNS Server) statement in `radvd` configuration file `/etc/radvd.conf` is set automatically by the `dhclient` exit hook script [ipv6-pd-br0](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/dhcp/dhclient-exit-hooks.d/ipv6-pd-br0). This script will use the IPv6 [link-local address](https://datatracker.ietf.org/doc/html/rfc8106#section-5.1)(found by `ip -6 a` or [mac_to_ip6_ll_addr](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/usr/local/bin/mac_to_ip6_ll_addr) script) of the `br0` interface as a value for the `RDNSS` statement for `radvd`:
```
root@r1:~# ip --brief -6 addr show dev br0 scope link
br0              UP             fe80::a7:29ff:fea6:ec61/64
root@r1:~#
root@r1:~# grep RDNSS /etc/radvd.conf
    RDNSS fe80::a7:29ff:fea6:ec61 { };
root@r1:~#
```

[nodnsupdate](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/dhcp/dhclient-enter-hooks.d/nodnsupdate) `dhclient` enter hook ensures that local [/etc/resolv.conf](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/resolv.conf) is not overwritten by DHCP client. As a safety net, modifications of the [/etc/resolv.conf](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/resolv.conf) were disabled with `chattr +i /etc/resolv.conf`:
```
root@r1:~# lsattr /etc/resolv.conf
----i---------e---- /etc/resolv.conf
root@r1:~#
```

Configuration files and scripts: [/etc/network/interfaces](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/network/interfaces), [/etc/systemd/system/networking.service.d/override.conf](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/systemd/system/networking.service.d/override.conf), [/etc/dhcp/dhclient.conf](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/dhcp/dhclient.conf), [/etc/dhcp/dhclient-exit-hooks.d/ipv6-pd-br0](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/dhcp/dhclient-exit-hooks.d/ipv6-pd-br0), [/usr/local/bin/mac_to_ip6_ll_addr](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/usr/local/bin/mac_to_ip6_ll_addr), [/etc/dhcp/dhclient-enter-hooks.d/get-static-ipv4-addrs](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/dhcp/dhclient-enter-hooks.d/get-static-ipv4-addrs), [/etc/dhcp/dhclient-exit-hooks.d/restore-static-ipv4-addrs](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/dhcp/dhclient-exit-hooks.d/restore-static-ipv4-addrs), [/etc/dhcp/dhclient-enter-hooks.d/nodnsupdate](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/dhcp/dhclient-enter-hooks.d/nodnsupdate), [/etc/dhcp/dhclient-enter-hooks.d/unset-dhcp-options](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/dhcp/dhclient-enter-hooks.d/unset-dhcp-options), [/etc/resolv.conf](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/resolv.conf), [/etc/modprobe.d/bnx2x.conf](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/modprobe.d/bnx2x.conf), [/etc/sysctl.conf](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/sysctl.conf), [/etc/systemd/system/isp-switch.service](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/systemd/system/isp-switch.service), [/usr/local/sbin/isp-switch](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/usr/local/sbin/isp-switch)

<h3 id="iptables"> :black_small_square: iptables</h3>

Notes on `iptables` and `ip6tables` rules:

* `dhclient` in DHCPv4 mode uses raw sockets(fallback UDP socket for sending unicast packets is also opened) and thus no firewall rule is needed
* important ICMP messages like "frag needed" or "TTL exceeded" are accepted thanks to conntrack "RELATED" state match
* certain ICMP messages sent by the router including "Echo Reply" or "Destination Unreachable" are rate limited by adjusting the kernel parameters in [/etc/sysctl.conf](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/sysctl.conf)
* `recent`(used in SSH chain) module internals are seen in the /proc/net/xt_recent/SSH file
* while `igmpproxy` is using raw sockets on the downstream interface, then in regard to upstream interface the `igmpproxy` simply acts as a normal multicast client(calls `setsockopt()` with IP_ADD_MEMBERSHIP) and thus there is a need to accept IGMP membership query messages sent by ISP. This also means that the IPTV UDP datagrams are sent towards the router application layer and dropped in the filter table INPUT chain.
* IGMP messages can not be tracked by the conntrack module. At least without an helper module. IGMP messages are sent to IPv4 multicast address and thus the `conntrack` module expects a reply sourced from a multicast address in order to move from UNREPLIED state to ESTABLISHED state. However, multicast address is never used as a src IP.
* `igmpproxy` subscribes to 224.0.0.2(IGMP "Leave group" messages) on a downstream interface and packets sent to this address are subject to INPUT chain rules. That's the reason for `-A INPUT -d 224.0.0.2/32 -i br0 -p igmp -j ACCEPT` rule. Details can be seen in [netfilter user mailinglist message](https://marc.info/?l=netfilter&m=168393431101974&w=2).
* dhclient in DHCPv6 mode is able to use ordinary UDP sockets thanks to link-local addresses and does not need to use raw sockets. This means that a firewall rule for DHCPv6 traffic is needed.
* ICMP6 "echo request" messages are rate limited by the `limit` module. Newer kernel versions have the `net.ipv6.icmp.ratemask` which would allow to rate limit the replies to "echo request" messages by adjusting the `net.ipv6.icmp.ratelimit`. ICMP6 "destination unreachable" messages are rate limited according to `net.ipv6.icmp.ratelimit`.
* important ICMP6 messages like "packet too big" or "time exceeded" or "destination unreachable" are accepted thanks to `conntrack` "RELATED" state match
* RA messages sent by radvd to `ff02::1` multicast addr via LAN-facing interface are looped back by the IP layer for local delivery. This is a default behavior and can be controlled by IPV6_MULTICAST_LOOP(`man 7 ipv6`). Those messages are dropped.

Rules are loaded by `iptables-restore` and `ip6tables-restore` utilities before bringing the `br0` bridge interface up and after taking the `br0` interface down. This is configred in `/etc/network/interfaces` file.

Configuration files: [/usr/local/etc/IPv4_fw_rules](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/usr/local/etc/IPv4_fw_rules), [/usr/local/etc/IPv4_default_fw_rules](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/usr/local/etc/IPv4_default_fw_rules), [/usr/local/etc/IPv6_fw_rules](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/usr/local/etc/IPv6_fw_rules), [/usr/local/etc/IPv6_default_fw_rules](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/usr/local/etc/IPv6_default_fw_rules)

<h3 id="dnsmasq"> :black_small_square: dnsmasq</h3>

As `dnsmasq` is configured with `interface=br0` and `bind-interfaces`, then `setsockopt()` with `SO_BINDTODEVICE` option is called by `dnsmasq` when started and socket is bound to a `br0` interface. [Under the hood](https://github.com/torvalds/linux/blob/ba0ad6ed89fd5dada3b7b65ef2b08e95d449d4ab/net/core/sock.c#L667), the socket is bound to an ifindex of `br0` interface. As restarting the `networking.service` deletes and recreates the network bridge `br0` and the new `br0` has an [incremented ifindex](https://github.com/torvalds/linux/blob/80e62bc8487b049696e67ad133c503bf7f6806f7/net/core/dev.c#L9553), then `dnsmasq.service` has to be restarted if the `networking.service` is restarted:
```
root@r1:~# systemctl show dnsmasq | grep PartOf=
PartOf=networking.service
root@r1:~#
root@r1:~# systemctl show networking | grep ConsistsOf=
ConsistsOf=hostapd.service igmpproxy.service dnsmasq.service
root@r1:~#
```

`dnsmasq` is configured not to read `/etc/resolv.conf` and use the upstream DNS servers specified in its configuration file:
```
root@r1:~# grep ^server /etc/dnsmasq.conf
server=2620:fe::fe
server=2620:fe::9
server=9.9.9.9
server=149.112.112.112
root@r1:~#
```
Configuration files: [/etc/dnsmasq.conf](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/dnsmasq.conf), [/etc/systemd/system/dnsmasq.service.d/override.conf](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/systemd/system/dnsmasq.service.d/override.conf)

<h3 id="radvd"> :black_small_square: radvd</h3>

`radvd` configuration and necessary reloads for activating the configuration is automatically handled by [dhclient exit hook script](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/dhcp/dhclient-exit-hooks.d/ipv6-pd-br0) and [expired prefixes removal script](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/usr/local/sbin/rm-expired-prefixes) executed by `cron` at every minute.

The general idea is that it's needed to keep advertising the prefixes with no valid or preferred lifetime left towards LAN at least as long as the last non-zero valid lifetime of the prefix in order to ensure that all the hosts in LAN pick up the prefix deprecation. Such setup ensures that the prefix deprecation is seen even by devices which were for example in suspended to RAM state at the time of the delegated prefix change.

For each "prefix" statement in radvd.conf file the script finds a timestamp in the future when this prefix could be removed from the radvd.conf file in order to avoid stale entries.

Example of `/etc/radvd.conf`:
![radvd_conf](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/radvd_conf.jpg)

Scripts: [/etc/dhcp/dhclient-exit-hooks.d/ipv6-pd-br0](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/dhcp/dhclient-exit-hooks.d/ipv6-pd-br0), [/usr/local/sbin/rm-expired-prefixes](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/usr/local/sbin/rm-expired-prefixes)

<h3 id="igmpproxy"> :black_small_square: igmpproxy</h3>

[Patched](https://github.com/tonusoo/koduinternet-cpe/blob/main/patches/igmpproxy.patch)(issue [#95](https://github.com/pali/igmpproxy/issues/95)) version of `igmpproxy` is installed. [igmpproxy_0.2.1-1+relaxed-timeout1_amd64.deb](https://github.com/tonusoo/koduinternet-cpe/blob/main/igmpproxy_0.2.1-1%2Brelaxed-timeout1_amd64.deb) package was built with commands below:
1. `apt source igmpproxy`
2. `cd igmpproxy-0.2.1/`
3. `patch --verbose src/igmpproxy.c ~/koduinternet-cpe/patches/igmpproxy.patch`
4. `dch --local +relaxed-timeout` with "relaxed pselect() timeout in main loop" note for the changelog
5. `cd . && dpkg-buildpackage -b`

Package can be installed with `dpkg -i igmpproxy_0.2.1-1+relaxed-timeout1_amd64.deb` and `apt-mark hold igmpproxy` prevents the package from being automatically upgraded.

Restarting the `networking.service` will call the `/lib/bridge-utils/ifupdown.sh` script which deletes(`brctl delbr ..`) and recreates(`brctl addbr ..`) the network bridge `br0`. However, the new `br0` interface will have an incremented ifindex(`cat /sys/class/net/br0/ifindex` or `ip l sh dev br0`) and thus the `setsockopt()` calls by `igmpproxy` will fail. Example where interface with ifindex 17(21 in octal) no longer existst:
```
setsockopt(3, SOL_IP, IP_MULTICAST_IF, "\217U\0\0\300\250\0\1\21\0\0\0", 12) = -1 EADDRNOTAVAIL (Cannot assign requested address)
```
That's the reason why `igmpproxy.service` is configured to be restarted if `networking.service` is restarted:
```
root@r1:~# systemctl show igmpproxy | grep PartOf=
PartOf=networking.service
root@r1:~#
root@r1:~# systemctl show networking | grep ConsistsOf=
ConsistsOf=hostapd.service igmpproxy.service dnsmasq.service
root@r1:~#
```

As an example, here is the multicast routing table(populated by `igmpproxy`) and switch multicast group membership table(populated by IGMP snooping) at the time when host connected to `lan0` is running `mplayer -vf yadif -volume 80 -fs -ao pulse udp://@239.3.1.106:1234`:
```
root@r1:~# ip mroute
(10.0.2.28,239.3.1.106)          Iif: wan0.4     Oifs: br0  State: resolved
root@r1:~#
root@r1:~# bridge -s mdb
33: br0  lan0  239.3.1.106  temp   251.57
root@r1:~#
```

IGMP snooping is enabled by default:
```
root@r1:~# cat /sys/devices/virtual/net/br0/bridge/multicast_snooping
1
root@r1:~#
```
`force_igmp_version` for `wan0.4` is not touched, i.e it's in IGMPv3 mode and falls back to IGMPv1/v2 mode if needed.

Configuration files: [/etc/igmpproxy.conf](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/igmpproxy.conf), [/etc/systemd/system/igmpproxy.service.d/override.conf](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/systemd/system/igmpproxy.service.d/override.conf)

<h3 id="ntp-config"> :black_small_square: NTP config</h3>

By default, the `dhclient` `/etc/dhcp/dhclient-exit-hooks.d/timesyncd` script builds the `/run/systemd/timesyncd.conf.d/01-dhclient.conf` configuration file for `systemd-timesyncd` SNTP client and instructs it to use the NTP servers provided by Telia's DHCP server. Instead, the default Debian NTP pool compiled to `systemd-timesyncd` is used by creating a `/etc/systemd/timesyncd.conf.d/01-dhclient.conf` symlink pointing to `/dev/null`. Output of `timedatectl`:

![timedatectl output](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/timedatectl_output.jpg)


<h3 id="hostapd"> :black_small_square: hostapd</h3>

`Asus PCE-AC88`(`Broadcom BCM4366/4` SoC) WNIC is configured to work as `IEEE 802.11a/n/ac` AP. For the `IEEE 802.11ac` mode, the channel width is up to 80 MHz(5170 - 5250 MHz). `hostapd` adds the WNIC named `wlan0` to bridge `br0`. Connectivity between the Wi-Fi clients is allowed. Output of [hostapd_cli](https://manpages.debian.org/testing/hostapd/hostapd_cli.1.en.html) `status` and `get_config` commands can be seen below:

![hostapd_cli_output](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/hostapd_cli_output.jpg)

Firmware blob loaded by `brcmfmac` driver is `brcmfmac4366c-pcie.bin`:
```
root@r1:~# md5sum /lib/firmware/brcm/brcmfmac4366c-pcie.bin
e3bb4457082aa54769253e9168db2269  /lib/firmware/brcm/brcmfmac4366c-pcie.bin
root@r1:~#
root@r1:~# strings /lib/firmware/brcm/brcmfmac4366c-pcie.bin | tail -1
4366c0-roml/pcie-ag-splitrx-fdap-mbss-mfp-wnm-osen-wl11k-wl11u-txbf-pktctx-amsdutx-ampduretry-chkd2hdma-proptxstatus-11nprop-obss-dbwsw-ringer-dmaindex16-bgdfs-murx-wwdfs Version: 10.28.2 (r769115) CRC: 6924533a Date: Mon 2018-11-05 03:22:36 PST Ucode Ver: 1128.17924 FWID: 01-d2cbb8fd
root@r1:~#
```
This is provided by `firmware-brcm80211` package, but the same firmware version can also be downloaded from [Linux kernel git repo](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/tree/brcm/brcmfmac4366c-pcie.bin):
```
root@r1:~# curl -s https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain/brcm/brcmfmac4366c-pcie.bin | md5sum
e3bb4457082aa54769253e9168db2269  -
root@r1:~#
```
For firmware related troubleshooting purposes, the `brcmfmac` driver can be loaded with `modprobe -v brcmfmac debug=0x1416` when built with debugging support:

![brcmfmac_with_debug_functions](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/brcmfmac_with_debug_functions.jpg)

Configuration files: [/etc/hostapd/hostapd.conf](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/hostapd/hostapd.conf), [/etc/systemd/system/hostapd.service.d/override.conf](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/systemd/system/hostapd.service.d/override.conf)

<h3 id="linux-pam-config"> :black_small_square: Linux PAM config</h3>

Diff with `Debian 10` default Linux PAM `common-auth` enabling 2FA(user password and OTP):
```
root@r1:~# diff -u /etc/pam.d/common-auth{~,}
--- /etc/pam.d/common-auth~     2023-06-01 21:22:43.894184345 +0300
+++ /etc/pam.d/common-auth      2023-06-01 21:48:15.777645409 +0300
@@ -13,8 +13,16 @@
 # pam-auth-update to manage selection of other modules.  See
 # pam-auth-update(8) for details.

+# If pam_yubico.so returns PAM_SUCCESS, then pam_google_authenticator.so is skipped.
+# All other return codes are ignored and pam_google_authenticator.so is used.
+# pam_yubico.so has "forward_pass"(pam_set_item(pamh, PAM_AUTHTOK, onlypasswd)) enabled by default.
+auth   [success=1 default=ignore]      pam_yubico.so id=12345 authfile=/etc/authorized_yubikeys urllist=https://api.yubico.com/wsapi/2.0/verify
+
+# pam_google_authenticator.so is able to detect if user password is entered before TOTP.
+auth   required        pam_google_authenticator.so forward_pass
+
 # here are the per-package modules (the "Primary" block)
-auth   [success=1 default=ignore]      pam_unix.so nullok_secure
+auth   [success=1 default=ignore]      pam_unix.so nullok_secure try_first_pass
 # here's the fallback if no module succeeds
 auth   requisite                       pam_deny.so
 # prime the stack with a positive return value if there isn't one already;
root@r1:~#
```
YubiKeys token IDs are mapped to username in `/etc/authorized_yubikeys` file. `$HOME/.google_authenticator` is created by `google-authenticator` utility and it includes the secret key, emergency scratch codes, rate-limit configuration, etc. SSH keyboard interactive authentication is enabled in `/etc/ssh/sshd_config` by changing the value of `ChallengeResponseAuthentication` to `yes`.

Configuration files: [/etc/pam.d/common-auth](https://github.com/tonusoo/koduinternet-cpe/blob/main/conf/etc/pam.d/common-auth)

<h2 id="bandwidth-tests"> :small_blue_diamond: Bandwidth tests</h2>

Statistics of `wan0` when host connected to `lan2` was running a TCP bidirectional test against `iperf3` server in another ISP network:

![iptraf-ng wan0 stats](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/iptraf-ng_wan0_stats.jpg)

Statistics of all the interfaces when hosts connected to `lan2` and `lan3` were running a TCP bidirectional test against `iperf3` server in another ISP network and smartphone connected to Wi-Fi was running Ookla's Speedtest:

![iptraf-ng general stats](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/iptraf-ng_general_stats.jpg)

[Speedtest.net](https://www.speedtest.net/) results when executed from a host connected to `lan2`:

![speedtest from host in lan](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/speedtest_from_host_in_lan.jpg)

[Speedtest.net](https://www.speedtest.net/) results when executed from a smartphone connected to `r1` Wi-Fi network:
![speedtest from wifi client in lan](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/speedtest_from_wifi_client_in_lan.jpg)


<h2 id="hardware-mods"> :small_blue_diamond: Hardware mods</h2>
<h3 id="nic-fan-replacement"> :black_small_square: Replacing the Dell N20KJ stock fan with Noctua NF-A4x10 fan</h3>

Stock `ADDA AD0412HB-K96` fan was replaced with silent, `Noctua NF-A4x10` fan:

![Dell N20KJ with NF-A4x10 fan](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/Dell_N20KJ_with_NF-A4x10_fan.jpg)

`Noctua NF-A4x10` is connected to motherboard chassis fan connector where it is picked up by `lm-sensors`:

![NIC fan RPM by lm-sensors](https://github.com/tonusoo/koduinternet-cpe/blob/main/imgs/NIC_fan_RPM_by_lm-sensors.jpg)

`Dell N20KJ` fan fault detection was disabled in `eDiag` engineering mode(`ediag_x64.efi -b10eng`) with commands below:
```
nvm cfg
option 4 (board i/o)
83=1 (disabled)
save
exit
```

<h2 id="acknowledgements"> :small_blue_diamond: Acknowledgements</h2>

Users [upnatom](https://www.dslreports.com/profile/1426739) and [JAMESMTL](https://www.dslreports.com/profile/1901978) in [DSLReports forum](https://www.dslreports.com/forums/all) ["Bypassing the HH3K up to 2.5Gbps using a BCM57810S NIC"](https://www.dslreports.com/forum/r32230041-Internet-Bypassing-the-HH3K-up-to-2-5Gbps-using-a-BCM57810S-NIC) thread.
User **anon23891239** and others in [OpenWRT forum](https://forum.openwrt.org/) ["Support MA5671A SFP GPON"](https://forum.openwrt.org/t/support-ma5671a-sfp-gpon/48042/) thread. [README](https://github.com/tonusoo/koduinternet-cpe/blob/main/README.md) is inspired by ["Human-Activity-Recognition"](https://github.com/ma-shamshiri/Human-Activity-Recognition) repo.

<h2 id="license"> :small_blue_diamond: License</h2>

[MIT License](https://github.com/tonusoo/koduinternet-cpe/blob/main/LICENSE)
