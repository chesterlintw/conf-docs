OpenBMC: An Open Framework of Embedded Linux In Server Management
===

**OpenBMC is an open-source project supported by LINUX Foundation. It's originated
from Facebook's hackthon event in 2014, which aims to create an open infrastructure
for BMC firmware development. Unlike the traditional BMC LINUX solutions have lots
of proprietary codes, openBMC can be a standard and all designs or changes must be
reviewed based on common principles, which enhance  reliability and compatibity of
firmware with other systems or services.Since there are more and more cloud providers
participate in OpenBMC development, it's a good opportunity to talk about this new
framework and how it would affect server management in the future.**

## Baseband Management Controller (BMC)
BMC is a micro-system embedded in server motherboards. It connects with a variety
of hardware devices, sensors, bridge chips and peripheral buses. System managment
software can monitor, configure and manage hardware resources via BMC, or even
apply updates on different components such as UEFI, VBIOS, Power Management SoCs,
NICs, CPLD/FPGA, ... and so on.


### Hardware Interfaces
BMC has strong connectivity with hardware components, which uses hundreds of IO
pins to connect with many devices inside the server. It also has common channels
such as dedicated/shared LANs, I2C, SPI, USB, ADC, LPC, eSPI, PCIe, .. and so on.
To communicate with devices, BMC supports different protocols such as SMBus, MCTP,
NC-SI or PLDM.

### Software Interfaces.
   * **IPMI 2.0** [1] is one of major managment protocol used in BMC firmware
     since 1998. It can runs on different channels such as LAN, SMBUS, KCS, BT,
     UART and so on.
   * **Redfish** [2]  is a new management API set targets to replace IPMI. It's
     based on RESTful and aims to provide simple but secured methods for Software
     Defined Data Center (SDDC) managment. It consists of HTTP request methods,
     JSON and schema so that management clients can understand how to use APIs
     since different vendors might have different designs.
   * **SNMP, Web Interface or Propietary protcols** owned by vendors.

### Management targets
CPU, GPU, NVMe/SATA SSD, LAN chips, thermal states of devices and the whole system,
FAN control, PSUs, circuit status, power capping, system on/off & chassis control,
BIOS/UEFI running status & configurations, device hotplu, .. etc.

## OpenBMC
A common BSP framework based on ycoto, which specifically demonstrates a reference
design of BMC LINUX system. In the past there was no standard for BMC LINUX
construction so hardware vendors could only choose BSPs from a few firmware vendors,
or even designed their own BSP in order to construct their BMC firmware solutions,
which still have lots of proprietary interfaces and methods for managing server
resources. Beisdes, these downstream code-base could also be outdated and insecure
since it might be difficult to merge proprietary codes with upstream versions.

OpenBMC can be an ideal and reliable solution for unifying firmware infrastructure
and its firmware instances can work across hterogeneous systems so that different
platforms' resource can still be managed based on standard.


## Software Stack
![Alt text](./pics/meta-layers.png)

### Yocto & Poky
**Yocto** is an automation build framework based on OpenEmbedded[3], which focuses
on embedded Linux distribution. It offers tools, packages, hierarchical layers
and programmable scripts for developers to quickly create and customize their
own systems. **Poky** is a reference distribution of the Yocto Project[4]. It's
the core that openbmc uses as the build framework. Despite OpenBMC is derived
from yocto/poky but it's not listed in yocto's repository list.

### BitBake
A automatic build tool written in Python, which is mainly used by *Poky* for
handling cross-compilation prorcess of embedded Linux or open source packages.
To create a build target, developers need to implement the following items:

* ### classes: (*.bbclass)
  A class file has common variables that can be shared among different recipes,
  and it can also have python scripts to specifically define build actions, such
  as compile, install, image & signature generation, ... etc.
* ### recipe-*
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

* ### meta-* layers:
  **An openBMC firmware consists of multiple layers:**

  **/meta-openembedded:** The OpenEmbedded core used in OpenBMC

  **/poky:** A modified poky which is compatible with OpenBMC.

  **Major BMC SoC layers:**
    * meta-aspeed: ast2400, ast2500 ..., etc. [armv6l]
    * meta-nuvoton: ncpm7xx [armv7]

    Both astxx and ncpmxx can choose upstream kernel & u-boot. However different
    hardware platforms have different device-tree blobs (dtb) despite some of
    them uses the same SoC. For example,
    * The purpose of GPIO pins can be different.
    * I2C clock freq / I2C MUX addresses can be different.
    * Specific CPLD / FPGA control which could differ among platforms.
    * Specific thermal components or sensor measuring.
    * Enabling / Disabling hardware components based on different applications.

  **meta-phorsphor:** A major layer which contains the code-base that a BMC
  firmware needs, such as ipmi, network managment, logging, .

  **Cloud/HW vendors:** meta-facebook, meta-ibm, meta-google, meta-microsoft,
  meta-intel, meta-qualcomm, meta-quanta, meta-inventec, meta-lenovo ..., etc.

  **Host CPU-ARCH layers:** meta-x86, meta-openpower, meta-arm.

  **Host Platform layers:** meta-ibm/meta-romulus, meta-ibm/meta-z,
  meta-intel/meta-s2600wf, meta-microsoft/meta-olympus..., etc.

### YAML descriptors of D-Bus interfaces
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

### Major services
**systemd** manages and controls all services in openbmc. Most of daemons in
openbmc are systemd-based and all communications happen on systmd's D-Bus.

**phorphor-dbus-monitor**: A event monitor on phorsphor-dbus. It has to listen
and react to events, or invoke corresponding handlers if registered. *E.g, A
service raised a PropertiesChanged event so that other services could be aware
of it and even took actions accordingly.*

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

**Others:** phorsphor-log-manager, phorsphor-network-manager, phospher-network-manager..., etc.

### OpenBMC REST API
Apart from Redfish, OpenBMC designs its RESTful API as primary management
interface. Per the request methods [GET/POST/PUT/PATCH] via HTTPS, users can
access and control resources that openBMC is managing. Like Redish, this API also
uses JSON to compose reqeusts or responses.

### Binary utilities in firmware
Most of tools are from busybox, however there are still some tools from util-linux,
such as mount,umount, fsck and sulogin.

### Firmware image layout and boot-flow
As a fitImage, openbmc firmware image consists of several parts, such as u-boot
code & env, kernel image, initramfs and FDTs. To check firmware integrity, some
of them must be verified by sha256 hash values before booting to the kernel.
Besides, the default file paritions used in LINUX are described as MTD in this DT
file:
> kernel-src/arch/arm/boot/dts/openbmc-flash-layout.dtsi

however it's still able to add or overwrite partitions. To link all paritions
into one image, either static or UBI is selectable. [TBD]

**Firmware-Boot-Flow:** SoC-BootROM-> U-Boot -> verify sha256 sig of kernel,
initramfs and fdt -> running initramfs -> start systemd -> start all bmc-fw daemons.


### Firmware Security [TBD]
  * **Signed firmware image** has been implemented [ref]

  * **phosphor-certificate-manager**: This service provides a unified interface
    to manage all kinds of certificates, including https, user authentication...,
    etc.

  * **User Management:** OpenBMC relies on PAM module to authenticate users, and
    it avoids transmitting passwords on D-BUS for security concerns. It also
    support Group & Privilege Roles in order to distinguish privileged and
    non-privileged users. [ref]
      * Groups: ssh, ipmi, redfish and web.
      * Privileges: admin, operator, user, callback.
  * HTTP/Redfish
  * Security Response Team

### Market opportunities for SUSE
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
7.

---
## Notes
* <TODO>: Manifest in image layout? /etc/default/obmc? phosphor-hwmon: Sensors / Hardware Monitors? ObjectMapper in D-Bus & Hwmon?
