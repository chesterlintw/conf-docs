OpenBMC: An Open Framework of Embedded Linux In Server Management
===

**OpenBMC is an open-source project supported by LINUX Foundation. It's originated
from Facebook's hackthon event in 2014, which aims to create an open infrastructure
for BMC firmware development. Unlike the traditional BMC LINUX solutions have lots
of proprietary codes, openBMC can be a standard and all designs or changes must be
reviewed based on common principles, which enhance reliability and compatibity of
firmware with other systems or services.Since there are more and more cloud providers
participate in OpenBMC development, it's a good opportunity to talk about this new
framework and how it would affect server management in the future.**

## 1. Baseband Management Controller (BMC)
BMC is a micro-system embedded in server boards, which is widely adopted by data
center products. It connects with a variety of components inside a server so that
system managment software can monitor, configure and manage hardware and firmware
resources via BMC. BMC can also apply firmware update to these components, which
increases the efficiency of data center management so that new settings or security
patches can be deployed quickly and massively.

### 1.1 Hardware Interfaces
BMC has strong connectivity with hardware components, which uses hundreds of IO
pins to connect with many devices inside the server. It also has common channels
such as dedicated/shared LANs, I2C, SPI, SDIO, USB, ADC, LPC, eSPI, PCIe, .. and
so on. To communicate with devices, BMC supports different protocols such as KCS,
BT, SMBus, PMBus, MCTP, NC-SI or PLDM.

### 1.2 Software Interfaces.
   * **IPMI 2.0** [1] is one of major managment protocol used in BMC firmware
     since 1998. It can runs on different channels such as LAN, SMBUS, KCS, BT,
     UART and so on.
   * **Redfish** [2]  is a new management API set targets to replace IPMI. It's
     based on RESTful and aims to provide simple but secured methods for Software
     Defined Data Center (SDDC) managment. It consists of HTTP request methods,
     JSON and schema so that management clients can understand how to use APIs
     since different vendors might have different designs.
   * **SNMP, SMASH-CLI over SSH, WEB or Propietary protcols** owned by vendors.

### 1.3 Management targets
CPU, GPU, onboard SoCs/FPGA/CPLD, NVMe/SATA SSD, storage controllers, onboard LAN,
network adapters, thermal control, PSUs, voltage/current sensors, power management,
UEFI or other firmwares' status and configuration, device hotplug, ... etc.

## 2. OpenBMC
A common BSP framework based on ycoto, which specifically demonstrates a reference
design of BMC LINUX system. In the past, server vendors could only choose BSPs
from a few firmware vendors, or designed their own BSPs, which still have lots of
proprietary designs for managing server resources. Beisdes, these BSPs could also
be outdated and insecure since they might not regularly merge with with upstream
versions. OpenBMC can be a model to unify firmware infrastructure and its instances
can work across hterogeneous systems therefore resources among different platforms
can still be managed based on standard.

## 3. Software Stack
![Alt text](./pics/meta-layers.png)

### 3.1 Yocto & Poky
**Yocto** is an automation build framework based on OpenEmbedded[3], which focuses
on embedded Linux distribution. It offers tools, packages, hierarchical layers
and programmable scripts for developers to quickly create and customize their
own systems. **Poky** is a reference distribution of the Yocto Project[4]. It's
the core that openbmc uses as the build framework. Despite OpenBMC is derived
from yocto/poky but it's not listed in yocto's repository list.


### 3.2 BitBake
A automatic build tool written in Python, which is mainly used by *Poky* for
handling cross-compilation prorcess of embedded Linux or open source packages.
To create a build target, developers need to implement the following items:

#### 3.2.1 classes: (*.bbclass)
A class file has common variables that can be shared among different recipes,
and it can also have python scripts to specifically define build actions, such
as compile, install, image & signature generation, ... etc.

#### 3.2.2 recipe-*
The recipes are used to customize each feature or package's build parameters,
such as source path [local/SCM/etc], dependencies on specific modules, branch
name or commit hash, lincense information, patch files, .. and so on. Baed on
these information BitBake can undestand how to deal with build requirements.
Each layer can have its own recipes or overwrite upper-layers' recipes.
*Examples: recipe-bsp/u-boot, recipe-core/systemd, recipe-kernel,
recipes-phosphor/images, .. etc.*

**.bb or .bbappend files:** These files contains detail build parameters that
recipes will need. Each recipe could have its own .bb files or it can use
.bbappend files to add changes on upper-layers' recipes or even overwrite it.
Specific scripts are also possible like we have mentioned in *classes*.

#### 3.2.3 meta-* layers:
An openBMC firmware consists of multiple layers:

  * **/meta-openembedded:** The OpenEmbedded core used in OpenBMC

  * **/poky:** A modified poky which is compatible with OpenBMC.

  * **Major BMC SoC layers:**
    * meta-aspeed: ASPEED ast2400, ast2500 [armv6l], ast2600 [armv7]
    * meta-nuvoton: NUVOTON ncpm7xx [armv7]
    * meta-fsp2: IBM FSP2 [ppc44x]

    Both astxx and ncpmxx can choose upstream kernel & u-boot. However different
    hardware platforms have different device-tree blobs (dtb) despite some of
    them uses the same SoC. For example,
    * The purpose of GPIO pins can be different.
    * I2C clock freq / I2C MUX addresses can be different.
    * Specific CPLD / FPGA control which could differ among platforms.
    * Specific thermal components or sensor measuring.
    * Enabling / Disabling hardware components based on different applications.

  * **meta-phorsphor:** A major layer which contains the code-base that a BMC
    firmware needs, such as ipmi, network managment, logging, .

  * **Cloud/HW vendors:** meta-facebook, meta-ibm, meta-google, meta-microsoft,
    meta-intel, meta-qualcomm, meta-quanta, meta-inventec, meta-lenovo ..., etc.

  * **Host CPU-ARCH layers:** meta-x86, meta-openpower, meta-arm.

  * **Host Platform layers:** meta-ibm/meta-romulus, meta-ibm/meta-z,
    meta-intel/meta-s2600wf, meta-microsoft/meta-olympus..., etc.

### 3.3 The running services within OpenBMC
**systemd** manages and controls all services in openbmc. Most of daemons in OpenBmc
are systemd-based with specific prefix like **xyz.openbmc_project.**, **obmc-**
or **phosphor-**", and all systemd units are under /lib/systemd/system/.

**D-Bus**: In OpenBMC, inter-service communication relies on D-Bus. When users
interacts with an OpenBMC firmware, such as polling machine/sensor status, changing
configs, managing user settings ... etc, all data-flows also happen on D-Bus.
**phorphor-dbus-monitor** is designed to listen and react all BMC-related events,
and it can invoke event handlers if registered. *For example, a service raised a
PropertiesChanged event so that other services could be aware of it and even took
actions accordingly.* You can try "busctl --system" on BMC console for more details.

**bmcweb** provides a variety of web services via http such as web interface,
redfish, IKVM, virtual media, .. etc. It integrates **crow** as its http server,
which is based on boost & C++11. **phorsphor-webui** includes a set of WebUI files
[html/css/js] packed by webpack so that users can access BMC via web. It also
contains RESTful calls based on AngularJS in order to let the client side can
communicate with BMC firmware via REST API.

**obmc-ikvm:** Remote Keyboard/Video/Mosue Interface based on noVNC.

**ipmid:** **phosphor-host-ipmid** is the major ipmi daemon for handling most of
ipmi commands and communication packets between BMC and host via KCS or BT channel.
**phopsphor-net-ipmid** relies on phosphor-host-ipmi but it's responsible for
dealing with ipmi requests from LAN channels.

**Others:** phorsphor-log-manager, phorsphor-network-manager, phospher-network-manager
..., etc.

### 3.4 YAML descriptors of D-Bus interfaces
OpenBMC offers two types of yaml: *.interface.yaml* and *.error.yaml*. YAML is
used to define schema of service interfaces, such as properties, methods,
enumerations or error codes that services will need in DBus communication.In the
repo of phorsphor-dbus-interface[5], all kinds of .interface.yaml files are
defined here, such as sensor interface,, power control interface, .. etc. OpenBMC
hierarchially presents every interface by the following rule:
> xyz.openbmc_project.<category>.<subcategory>....<function>

For example,
> xyz.openbmc_project.Control.FanPwm  // FAN PWM control
>
> xyz.openbmc_project.Control.ACPIPowerState.interface.yaml //ACPI status
>
> xyz.openbmc_project.NVMe.Status.interface.yaml //NVMe status

While running the firmware build process, the **sdbus++**[6] tool converts all.yaml
files into coressponding server.cpp files, and all these server.cpp files would
be combined as one libphosphor_dbus.cpp, which will be compiled as
libphosphor_dbus.so.0 and will be used by lots of phorsphor-* services.

### 3.5 OpenBMC REST API
Apart from Redfish, OpenBMC designs its RESTful API [7][8] as primary management
interface. Per the request methods [GET/POST/PUT/PATCH] via HTTPS, users can
access and control resources that openBMC is managing. Like Redish, this API also
uses JSON to compose reqeusts or responses.


### 3.6 Binary utilities in firmware
Most of tools are from busybox, however there are still some tools from util-linux,
such as mount,umount, fsck and sulogin.


### 3.7 Image, partition and boot-flow
As a fitImage, an openbmc firmware image consists of several parts, such as u-boot
code & env, kernel image, initramfs and FDTs. To check firmware integrity, some
of them must be verified by sha256 hash values before booting to the kernel. The
default file paritions are based on MTD, which is written in this DT file:
> kernel-src/arch/arm/boot/dts/openbmc-flash-layout.dtsi

However it's still able to add or overwrite partitions. Apart from the static
layout, a UBI image format is also selectable.

**Firmware-Boot-Flow:** SoC-BootROM-> U-Boot -> verify sha256 sig of kernel,
initramfs and fdt -> running initramfs -> start systemd -> default.target ->
multi-user.target -> start all bmc-fw daemons.


### 4. Firmware Security
**Signed firmware image** [10] has been implemented based on OpenSSL. This design
prevents OpenBMC from updating unreliable image sources, which could cause firmware
vulnerabilities or backdoors.

**phosphor-certificate-manager**: This service provides a unified interface to
manage all kinds of certificates, including https, user authentication..., etc.

**User Management:** OpenBMC relies on PAM module to authenticate users, and it
avoids transmitting passwords on D-BUS for security concerns. It also support
Group & Privilege Roles in order to distinguish privileged and non-privileged
users. [9]
  * Groups: ssh, ipmi, redfish and web.
  * Privileges: admin, operator, user, callback.

**Networking**
TLS is enabled on the following functions: IPMI-RAKP, HTTPS for WEB, REST APIs,
IKVM/Virtual-Media, SOL..., etc. However OpenBMC hasn't integrated a firewall
application for users to filter network traffic.

**Security Response Team**: OpenBMC has a workflow for community members to report
any vulnerability to the response team.

### 5. Market opportunities for SUSE
SUSE Manager might be a good start to integrate openBMC APIs in order to achieve
micro management of physical resources in data-ceneter, such as power managment,
storage control, device hotplug, all server firmwares' update, critical failure
warning, ..., etc.

### References
1. IPMI 2.0: https://www.intel.la/content/www/xl/es/servers/ipmi/ipmi-technical-resources.html
2. Redfish API: https://www.dmtf.org/standards/redfish
3. Yocto: https://www.yoctoproject.org/docs/2.7/brief-yoctoprojectqs/brief-yoctoprojectqs.html
4. Poky: https://www.yoctoproject.org/software-item/poky/
5. phorsphor-dbus-interface:https://github.com/openbmc/phosphor-dbus-interfaces
6. sdbus++: https://github.com/openbmc/sdbusplus
7. https://github.com/openbmc/docs/blob/master/rest-api.md
8. https://github.com/openbmc/docs/blob/master/REST-cheatsheet.md
9. https://gerrit.openbmc-project.xyz/c/openbmc/openbmc/+/8949

---
## Notes
* <TODO>: phosphor-hwmon: Sensors / Hardware Monitors? ObjectMapper in D-Bus & Hwmon?
