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
   * **SNMP, SMASH-CLI over SSH, WEB or propietary protcols** owned by vendors.

### 1.3 Management Targets
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

## 3. Framework
![meta-layers](./pics/layers.png)

### 3.1 Yocto & Poky
**Yocto** is an automation build framework based on OpenEmbedded[3], which focuses
on embedded Linux distribution. It offers tools, packages, hierarchical layers
and programmable scripts for developers to quickly create and customize their
own systems. **Poky** is a reference distribution of the Yocto Project[4]. It's
the core that openbmc uses as the build framework. Despite OpenBMC is derived
from yocto/poky but it's not listed in yocto's repository list.


### 3.2 BitBake
An automatic build tool written in Python, which is mainly used by *Poky*. Like
GNU make, it controls the building flows of embedded linux distributions and
related open source packages, such as downloading required tool-chains, source
packages, finding CPU-arch and code dependencies, cross-compiling, image creations
..., etc. In OpenBMC, BitBake needs the following items in order to build a machine
target:

#### 3.2.1 meta layers:
An OpenBMC firmware consists of multiple layers:

  * **/meta-openembedded:** The OpenEmbedded core used in OpenBMC

  * **/poky:** A modified poky which is compatible with OpenBMC.

  * **Major BMC SoC layers:**
    * meta-aspeed: ASPEED ast2400, ast2500 [armv6l], ast2600 [armv7]
    * meta-nuvoton: NUVOTON ncpm7xx [armv7]
    * meta-fsp2: IBM FSP2 [ppc44x]

    These layers can choose upstream kernel and u-boot since most of
    codes have been merged into upstream recent years. However different
    hardware platforms can have different device-tree blobs (dtb) despite some
    machines use the same SoC. The reasons could be:
    * Specific thermal components and sensors.
    * Specific GPIO-pin definitions.
    * Peripheral bus settings and purposes can be different. E.g, I2C.
    * CPLD / FPGA control which could differ among platforms.
    * Enabling / Disabling hardware components based on different applications.

    To fulfill the custom requirements, the hardware vendors should have their
    .dts files to describe their platforms' device trees. See more details in
    linux-kernel source: /arch/arm/boot/dts/\*bmc\*.dts\*

  * **meta-phorsphor:** A major layer which contains the code-base that a BMC
    firmware needs, such as ipmi, network managment, logging, sensor porting,
    certificate, image handling, etc.

  * **Cloud/HW vendors:** meta-facebook, meta-ibm, meta-google, meta-microsoft,
    meta-intel, meta-qualcomm, meta-quanta, meta-inventec, meta-lenovo ..., etc.

  * **Host CPU-ARCH layers:** meta-x86, meta-openpower, meta-arm.

  * **Host Platform layers:** meta-ibm/meta-romulus, meta-ibm/meta-z,
    meta-intel/meta-s2600wf, meta-microsoft/meta-olympus..., etc.

#### 3.2.2 recipes
Recipes are used to customize each feature or package's build parameters,
such as source path [local/SCM/etc], dependencies on specific modules, branch
name or commit hash, lincense information, patch files, .. and so on. Baed on
these information BitBake can undestand how to deal with build requirements.
Each layer can have its own recipes or overwrite upper-layers' recipes. For
example, A platform can overwrite a part of recipes-phosphor inherited from
meta-phorsphor or other layers if it has customized implementation, such as
GPIO definitions, sensors, LED settings ..., etc.
*Other examples: recipe-bsp/u-boot, recipe-core/systemd, recipe-kernel,
recipes-phosphor/images, .. etc.*

A recipe might have **.bb and .bbappend files**. These files contain detail build
parameters a recipe will need to build a package. Each recipe could have its own
.bb files or it can use .bbappend files to add changes on upper-layers' recipes
or even overwrite them. Specific python scripts can also be included in .bb or
.bbappend in order to add more variants on the build process.

#### 3.2.3 classes: (*.bbclass)
A bbclass contains common variables which can be shared among different recipes,
and it can also have python scripts to specifically define build actions, such
as compile, install, image & signature generation, ... etc.

#### 3.2.4 meta-layer/conf
Some layers have config files under the conf folder, and BitBake has to read these
files in order to understand the preffered options, depending layers, recipes and
machine name that this layer will need before starting a build process.

## 4. Software Stack

### 4.1 The Running Services in OpenBMC
![service](pics/service.png)

**systemd** starts and manages all services in openbmc. Most of daemons in OpenBmc
are systemd-based which have specific prefix like **xyz.openbmc_project.**, **obmc-**
or **phosphor-**", and all systemd units are under /lib/systemd/system/.

**D-Bus**: In OpenBMC, inter-service communication relies on D-Bus. When users
interact with an OpenBMC firmware, such as polling machine/sensor status, changing
configs, managing user settings ... etc, all data-flows also happen on D-Bus.
**phorphor-dbus-monitor** is designed to listen and react all BMC-related events,
and it can invoke event handlers if registered. *For example, a service raised a
PropertiesChanged event so that other services could be aware of it and even took
actions accordingly.* You can try "busctl --system" on BMC console for more details.

**bmcweb** provides multiple services via https such as web interface, redfish,
IKVM, virtual media ..., etc. It integrates **crow** as its http server, which is
based on boost and C++11. **phorsphor-webui** includes a set of WebUI files
[html5/js/css] packed by webpack so that users can access BMC via web. It also
contains RESTful calls based on AngularJS in order to let the client side can
communicate with BMC firmware via REST API.

**obmc-ikvm:** Remote Keyboard/Video/Mosue Interface based on noVNC.

**ipmid:** **phosphor-host-ipmid** is the major ipmi daemon for handling ipmi
requests and communication packets between BMC and host via KCS or BT channel.
**phopsphor-net-ipmid** relies on phosphor-host-ipmi but it's responsible for
dealing with ipmi requests from LAN channels.

**Others:** phorsphor-log-manager, phorsphor-user-manager, phospher-network-manager
..., etc.

### 4.2 YAML Descriptors of D-Bus Interface
OpenBMC offers two types of yaml: *.interface.yaml* and *.error.yaml*. YAML is
used to define schema of service interfaces, such as properties, methods,
enumerations or error codes that services will need in DBus communication. The
repo phorsphor-dbus-interface[5] defines all kinds of .interface.yaml files, such
as sensor interface, power control interface ..., etc. OpenBMC hierarchially
presents every interface by the following rule:
> xyz.openbmc_project."category"."subcategory"...."function"

For example,
> xyz.openbmc_project.Control.FanPwm  // FAN PWM control
>
> xyz.openbmc_project.Control.ACPIPowerState.interface.yaml //ACPI status
>
> xyz.openbmc_project.NVMe.Status.interface.yaml //NVMe status

While running the firmware build process, the **sdbus++**[6] tool converts all
.yaml files to coressponding server.cpp files, and all server.cpp files would be
merged into a libphosphor_dbus.cpp, which will be compiled as libphosphor_dbus.so.0.
Then this shared library will be used by all OpenBMC services on D-BUS.

### 4.3 OpenBMC REST API
Apart from Redfish, OpenBMC has its RESTful API [7][8] as primary management
interface. Per the request methods of HTTPS [GET/POST/PUT/PATCH], users can access
and control resources that openBMC is managing. Like Redish, this API also uses
JSON as request and response format. Here is an example of enumerating the eth0
on BMC:

    chester@linux-8mug:~> curl -k -X GET https://root:0penBmc@${bmc}:${bmcport}/xyz/openbmc_project/network/eth0/enumerate
    {
      "data": {
        "/xyz/openbmc_project/network/eth0": {
          "AutoNeg": false,
          "DHCPEnabled": true,
          "DomainName": [],
          "IPv6AcceptRA": false,
          "InterfaceName": "eth0",
          "LinkLocalAutoConf": "xyz.openbmc_project.Network.EthernetInterface.LinkLocalConf.both",
          "MACAddress": "52:54:0:12:34:56",
          "NTPServers": [],
          "Nameservers": [],
          "Speed": 0
        },
        "/xyz/openbmc_project/network/eth0/ipv4/49e97e86": {
          "Address": "10.0.2.15",
          "Gateway": "",
          "Origin": "xyz.openbmc_project.Network.IP.AddressOrigin.DHCP",
          "PrefixLength": 24,
          "Type": "xyz.openbmc_project.Network.IP.Protocol.IPv4"
        },
        "/xyz/openbmc_project/network/eth0/ipv6/ff024971": {
          "Address": "fe80::5054:ff:fe12:3456",
          "Gateway": "",
          "Origin": "xyz.openbmc_project.Network.IP.AddressOrigin.LinkLocal",
          "PrefixLength": 64,
          "Type": "xyz.openbmc_project.Network.IP.Protocol.IPv6"
        }
      },
      "message": "200 OK",
      "status": "ok"
    }

### 4.4 Sensor Polling
The sensor poliing [9] of OpenBMC relies on LINUX HWmon interface [10]. A platform
layer can create sensor config files in recipes-phosphor/sensors/phosphor-hwmon,
which provides sensor information to hwmon routines, such as chip models, mmio
addresses, sysfs paths, sensor labels, indexes, polling intervals, GPIO control,
warning thresholds ..., and so on. Each sensor config should have a match item
in the platform's device-tree. By writing the SYSTEMD_ENVIRONMENT_FILE variable
in phosphor-hwmon_%.bbappend, the **xyz.openbmc_project.Hwmon@.service** can know
which sensor configs are required before launching hwmon routines.

### 4.5 Binary Utilities
Most of tools in firmware are from busybox, however there are still some tools
from util-linux, such as mount,umount, fsck and sulogin.


## 5. Images, Partitions and Boot-Flow
As a fitImage, an openbmc firmware image consists of several parts, such as u-boot
code & env, kernel image, initramfs and FDTs. To check firmware integrity, some
of them must be verified by sha256 hash values before booting to the kernel. The
default file paritions are based on MTD, which is written in this DT file:
> kernel-src/arch/arm/boot/dts/openbmc-flash-layout.dtsi

However it's still able to add or overwrite partitions. Apart from the static
layout, the UBI (Unsorted Block Images) format is also selectable.

**Firmware-Boot-Flow:** SoC-Bootrom-> U-Boot -> verify image signatures ->
boot kernel -> mount initramfs -> start systemd -> default.target ->
multi-user.target -> start all bmc-fw daemons.


## 6. Firmware Security
**Signed firmware image** [11] has been implemented based on OpenSSL. This design
prevents OpenBMC from updating unreliable firmware images, which could be exploited
as vulnerabilities or backdoors.

**phosphor-certificate-manager**: This service provides a unified interface to
manage all kinds of certificates, including https, user authentication ..., etc.

**User Management:** OpenBMC relies on PAM module to authenticate users, and it
avoids transmitting passwords on D-BUS for security concerns. It also support
Group & Privilege Roles in order to distinguish privileged and non-privileged
users. [12]

  * Groups: ssh, ipmi, redfish and web.
  * Privileges: admin, operator, user, callback.

**Networking**: **TLS** is enabled on the following functions: IPMI-RAKP, HTTPS
for WEB, REST APIs, IKVM/Virtual-Media, SOL..., etc. However OpenBMC hasn't
integrated a firewall application for users to filter network traffic. **X-Auth**
can be used in Redfish sessions as credentials for accessing Redfish URLs.

**Security Response Team**: OpenBMC has a workflow for community members
to report any vulnerability to the response team.

## 7. Market Opportunities for SUSE
SUSE Manager might be a good start to integrate openBMC APIs in order to achieve
micro management of physical resources in data-ceneter, such as power managment,
storage control, device hotplug, all server firmwares' update, critical failure
warning, ..., etc.

### References
1.  https://www.intel.la/content/www/xl/es/servers/ipmi/ipmi-technical-resources.html
2.  https://www.dmtf.org/standards/redfish
3.  https://www.yoctoproject.org/docs/2.7/brief-yoctoprojectqs/brief-yoctoprojectqs.html
4.  https://www.yoctoproject.org/software-item/poky/
5.  https://github.com/openbmc/phosphor-dbus-interfaces
6.  https://github.com/openbmc/sdbusplus
7.  https://github.com/openbmc/docs/blob/master/rest-api.md
8.  https://github.com/openbmc/docs/blob/master/REST-cheatsheet.md
9.  https://github.com/openbmc/docs/blob/master/sensor-architecture.md
10. https://www.kernel.org/doc/Documentation/hwmon/sysfs-interface
11. https://gerrit.openbmc-project.xyz/c/openbmc/openbmc/+/8949
12. https://github.com/openbmc/docs/blob/master/user_management.md

