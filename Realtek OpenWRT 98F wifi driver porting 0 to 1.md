[TOC]
# Realtek OpenWRT 98F wifi driver porting

## OpenWRT Introduction

### What is OpenWRT

The OpenWrt Project is a Linux operating system targeting embedded devices, especially for wireless router device.

### Why need OpenWRT

OpenWrt provides a fully writable filesystem with package management, there are several thousand packages to extend the functionality of your device, and it provide UCI(Unified Configuration Interface) system and luci web user interface to configure the openwrt system in a convinent way. As a open source project, it's matained by many people so it's rubost and secure

### Documentations of OpenWRT

[OpenWRT Quick Start Guide](https://openwrt.org/docs/guide-quick-start/begin_here)  
This documentation will guide you how to use a OpenWRT installed device, such as how to access WEB UI, how to turn on/off wifi, how to upgrade firmware etc...

[OpenWRT User Guide](https://openwrt.org/docs/guide-user/start)  
This documentation will tell you how to control OpenWRT system by modify the system configurations.

[OpenWRT Developer Guide](https://openwrt.org/docs/guide-developer/start)  
This documentation will guid you how to download OpenWRT SDK, modify the SDK and build your personalized OpenWRT image, this one is very important for us.

#### OpenWRT Packages

One of the things that we've attempted to do with OpenWrt's template system is make it incredibly easy to port software(user space application or kernel module) to OpenWrt. If you look at a typical package directory in OpenWrt you'll find three things:

- package/Makefile
- package/patches
- package/files

The patches directory is optional and typically contains bug fixes or optimizations to reduce the size of the executable. The files directory is optional. It typically includes default config or init files.  
The package makefile is the important item because it provides the steps actually needed to download and compile the package.  
Looking at one of the package makefiles, you'd hardly recognize it as a makefile. Through what can only be described as blatant disregard and abuse of the traditional make format, the makefile has been transformed into an object oriented template which simplifies the entire ordeal.  
Here for example, is package/bridge/Makefile:  

```makefile
include $(TOPDIR)/rules.mk
PKG_NAME:=hostapd
PKG_LICENSE:=BSD-3-Clause
PKG_VERSION:=2016-06-15
PKG_RELEASE:=2
PKG_SOURCE:=$(PKG_NAME)-$(PKG_VERSION).tar.bz2
PKG_SOURCE_URL:=git://w1.fi/srv/git/hostap.git
PKG_SOURCE_SUBDIR:=$(PKG_NAME)-$(PKG_VERSION)
PKG_SOURCE_PROTO:=git

include $(INCLUDE_DIR)/package.mk

define Package/hostapd/Default
  SECTION:=net
  CATEGORY:=Network
  TITLE:=IEEE 802.1x Authenticator
  URL:=http://hostap.epitest.fi/
  DEPENDS:=$(DRV_DEPENDS) +hostapd-common +libubus
endef

define Package/hostapd/description
 This package contains a full featured IEEE 802.1x/WPA/EAP/RADIUS
 Authenticator.
endef

define Build/Configure
	$(Build/Configure/rebuild)
	$(if $(wildcard ./files/hostapd-$(CONFIG_VARIANT).config), \
		$(CP) ./files/hostapd-$(CONFIG_VARIANT).config $(PKG_BUILD_DIR)/hostapd/.config \
	)
	$(CP) ./files/wpa_supplicant-$(CONFIG_VARIANT).config $(PKG_BUILD_DIR)/wpa_supplicant/.config
endef

define Package/hostapd/install
	$(call Install/hostapd,$(1))
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/hostapd/hostapd $(1)/usr/sbin/
endef

$(eval $(call BuildPackage,hostapd))
```

BuildPackage variables  
As you can see, there's not much work to be done; everything is hidden in other makefiles and abstracted to the point where you only need to specify a few variables.  

- PKG_NAME - The name of the package, as seen via menuconfig and ipkg
- PKG_VERSION - The upstream version number that we're downloading
- PKG_RELEASE - The version of this package Makefile
- PKG_LICENSE - The license(s) the package is available under, SPDX form.
- PKG_BUILD_DIR - Where to compile the package
- PKG_SOURCE - The filename of the original sources
- PKG_SOURCE_URL - Where to download the sources from (directory)
- PKG_CONFIG_DEPENDS - specifies which config options influence the build configuration and should trigger a rerun of Build/Configure on change
- PKG_SOURCE_PROTO - the protocol to use for fetching the sources (git, svn, etc).
- PKG_SOURCE_URL - source repository to fetch from. The URL scheme must be consistent with PKG_SOURCE_PROTO (e.g. git://), but most VCS accept http:// or https:// URL nowadays.
- PKG_SOURCE_VERSION - must be specified, the commit hash or SVN revision to check out.
- PKG_SOURCE_SUBDIR - where should the temporary source checkout should be stored, defaults to $(PKG_NAME)-$(PKG_VERSION)

At the bottom of the file is where the real magic happens, “**BuildPackage**” is a macro setup by the earlier include statements. BuildPackage only takes one argument directly – the name of the package to be built, in this case “**hostapd**”. All other information is taken from the define blocks. This is a way of providing a level of verbosity, it's inherently clear what the DESCRIPTION variable in Package/hostapd is, which wouldn't be the case if we passed this information directly as the Nth argument to BuildPackage.  

BuildPackage defines  
Package/  
matches the argument passed to buildroot, this describes the package the menuconfig and ipkg entries. Within Package/ you can define the following variables:  

- SECTION - The type of package (currently unused)
- CATEGORY - Which menu it appears in menuconfig
- TITLE - A short description of the package
- DESCRIPTION - (deprecated) A long description of the package
- URL - Where to find the original software
- DEPENDS - (optional) Which packages must be built/installed before this package. See below for the syntax.

Build/Configure (optional)  
You can leave this undefined if the source doesn't use configure or has a normal config script, otherwise you can put your own commands here or use “$(call Build/Configure/Default,)” as above to pass in additional arguments for a standard configure script.  

Build/Compile (optional)  
How to compile the source; in most cases you should leave this undefined, because then the default is used, which calls make. If you want to pass special arguments to make, use e.g. “$(call Build/Compile/Default,FOO=bar)”  

Package/install  
A set of commands to copy files into the ipkg which is represented by the $(1) directory. As source you can use relative paths which will install from the unpacked and compiled source, or $(PKG_INSTALL_DIR) which is where the files in the Build/Install step above end up.  

Creating packages for kernel modules  
A kernel module is an installable program which extends the behavior of the linux kernel. A kernel module gets loaded after the kernel itself, (e.g. using insmod).  

A kernel program can be compile as two method:  
1. compile it into the kernel as a built-in.
2. compile it as a loadable kernel module.  

in Realtek original SDK, wifi driver complied as (1), but in OpenWRT, we have to compiled it as a kernel module.
our wifi driver actually is a kernel module, but in our SDK we compile it into kernel, but in OpenWRT we have to seperate it from kernel and build it as a kernel module.  

Here for example, is rtl8192cd in package/kernel/mac80211/Makefile:  
```makefile
define KernelPackage/rtl8192cd
  $(call KernelPackage/mac80211/Default)
  TITLE:=Realtek 8192cd wireless module support
  DEPENDS+= @(TARGET_realtek||TARGET_rtkmips||TARGET_rtkmipsel) +kmod-mac80211 +@DRIVER_11N_SUPPORT
  FILES:=$(PKG_BUILD_DIR)/drivers/net/wireless/rtl8192cd/rtl8192cd.ko
  AUTOLOAD:=$(call AutoProbe,rtl8192cd)
endef

define KernelPackage/rtl8192cd/description
 This module adds support for Realtek rtl8192cd wireless adapters
endef

config-$(call config_package,rtl8192cd) += RTL8192CD
$(eval $(call KernelPackage,rtl8192cd))
```

##### add a kernel package

In OpenWRT, package will be declare in package/kernel/xxxx/Makefile, below is a example about add rtl8192cd driver into OpenWRT driver:  
```makefile
define KernelPackage/rtl8192cd 
  $(call KernelPackage/mac80211/Default)
  TITLE:=Realtek 8192cd wireless module support
  DEPENDS+= @(TARGET_realtek||TARGET_rtkmips||TARGET_rtkmipsel) +kmod-mac80211 +@DRIVER_11N_SUPPORT
  FILES:=$(PKG_BUILD_DIR)/drivers/net/wireless/rtl8192cd/rtl8192cd.ko
  AUTOLOAD:=$(call AutoProbe,rtl8192cd)
endef

define KernelPackage/rtl8192cd/description
 This module adds support for Realtek rtl8192cd wireless adapters
endef

config-$(call config_package,rtl8192cd) += RTL8192CD
ifeq ($(CONFIG_PACKAGE_kmod-rtl8192cd),y)
  MAKE_OPTS += "EXTRA_CFLAGS += -I$(PKG_BUILD_DIR)/drivers/net/wireless/rtl8192cd \
				-I$(PKG_BUILD_DIR)/drivers/net/wireless/rtl8192cd/phydm \
				-I$(PKG_BUILD_DIR)/drivers/net/wireless/rtl8192cd/WlanHAL \
				-I$(PKG_BUILD_DIR)/drivers/net/wireless/rtl8192cd/WlanHAL/RTL88XX \
				-I$(PKG_BUILD_DIR)/drivers/net/wireless/rtl8192cd/WlanHAL/HalMac88XX \
				-DDM_ODM_SUPPORT_TYPE=0x1"
endif
$(eval $(call KernelPackage,rtl8192cd))
```

## Realtek OpenWRT 

To support OpenWRT, our engineers already add SoC platform (97F/98F) and port linux with drivers into OpenWRT SDK, and here is a wiki about Realtek OpenWRT you can refer, I will focus on how to port wifi driver into OpenWRT.

## OpenWRT WiFi Porting

what we need:   
1. A OpenWRT SDK with certain platform and linux(platform and linux porting will be done by other teams)  
2. A verified WiFi driver of certain platform. (can work in original SDK)






















## Add rtl8192cd into package/kernel/mac80211


## Build ethernet driver

The wifi driver is depended on ethernet driver, so we have to build ethernet driver first.
To build ethernet driver, "kmod-ca_packages, kmod-rtl8198f-fleetconntrack, kmod-switch-rtl8367rb, PACKAGE_flash, rtk_nv, diagshell" must be enabled in menuconfig.

- How to confirm ethernet driver is built successfully:
    Flash image into the board, if you see "RTK FleetConntrack Driver Init was done", means that your ethernet driver is ok and can be loaded into kernel.
![RTK FleetConntrack Driver Init was done](https://github.com/robotwxy/pics/raw/master/RTK%20FleetConntrack%20Driver%20Init%20was%20done.PNG)

## Build wifi driver

First, download the patch for openwrt/package/kernel/mac80211: [wifi_driver_patch.zip](https://github.com/robotwxy/pics/raw/master/wifi_driver_patch.zip), apply the patch into openwrt.

Second, enter menuconfig, enable **kmod-rtl8192cd** in Kernel Modules/Wireless Drivers.
```sh
make V=s -j4.
```
If you meet a error like this: 
![Package kmod-rtl8192cd is missing dependencies for the following libraries:](https://github.com/robotwxy/pics/raw/master/Package%20kmod-rtl8192cd%20is%20missing%20dependencies%20for%20the%20following%20libraries.PNG)

You can modify the file openwrt/include/package-ipkg.mk:
from
``` c
ifneq ($(PKG_NAME),toolchain)
  define CheckDependencies
	@( \
		rm -f $(PKG_INFO_DIR)/$(1).missing; \
		( \
			export \
				READELF=$(TARGET_CROSS)readelf \
				OBJCOPY=$(TARGET_CROSS)objcopy \
				XARGS="$(XARGS)"; \
			$(SCRIPT_DIR)/gen-dependencies.sh "$$(IDIR_$(1))"; \
		) | while read FILE; do \
			grep -qxF "$$$$FILE" $(PKG_INFO_DIR)/$(1).provides || \
				echo "$$$$FILE" >> $(PKG_INFO_DIR)/$(1).missing; \
		done; \
		if [ -f "$(PKG_INFO_DIR)/$(1).missing" ]; then \
			echo "Package $(1) is missing dependencies for the following libraries:" >&2; \
			cat "$(PKG_INFO_DIR)/$(1).missing" >&2; \
			false; \
		fi; \
	)
  endef
endif
``` 
to
``` c
ifneq ($(PKG_NAME),toolchain)
  define CheckDependencies
	@( \
		rm -f $(PKG_INFO_DIR)/$(1).missing; \
		( \
			export \
				READELF=$(TARGET_CROSS)readelf \
				OBJCOPY=$(TARGET_CROSS)objcopy \
				XARGS="$(XARGS)"; \
			$(SCRIPT_DIR)/gen-dependencies.sh "$$(IDIR_$(1))"; \
		) | while read FILE; do \
			grep -qxF "$$$$FILE" $(PKG_INFO_DIR)/$(1).provides || \
				echo "$$$$FILE" >> $(PKG_INFO_DIR)/$(1).missing; \
		done; \
		if [ -f "$(PKG_INFO_DIR)/$(1).missing" ]; then \
			echo "Package $(1) is missing dependencies for the following libraries:" >&2; \
			cat "$(PKG_INFO_DIR)/$(1).missing" >&2; \
		fi; \
	)
  endef
endif
``` 
After that wifi driver should be able to build.

After wifi driver is built, we should enbale **hostapd** and **wpa_supplicant** in menuconfig and make again.

## Try wifi driver

After flash the iamge into the board, use below commands to insert wifi driver module:
```sh
insmod /lib/modules/4.4.0/compat.ko 
insmod /lib/modules/4.4.0/cfg80211.ko  
insmod /lib/modules/4.4.0/mac80211.ko
insmod /lib/modules/4.4.0/rtl8192cd.ko
```
After wifi driver module is inserted without errors, up the wlan and check the wlan.
```sh
ifconfig wlan0 up
ifconfig
```
### Try hostapd to test wifi driver AP mode.
*To exclude other factors, suggest use **static IP** for both AP and client, and make sure no other process(such as wpa_supplciant and hostapd) are using the wlan.  
First, enter /tmp and write a hostapd.conf, example:
```sh
driver=nl80211
interface=wlan0
ctrl_interface=/var/run/hostapd
bridge=br-lan
ssid=hostapd

auth_algs=1
wpa=0

hw_mode=g
channel=11
ieee80211n=1

ignore_broadcast_ssid=0
country_code=US
wmm_enabled=1
```
Then use following command to run hostapd:
```sh
ifconfig wlan0 down up
hostapd hostapd.conf &
```
After hostapd started without errors, you should be able to find a "**hostapd_2_0-none-2g**" wifi in your client, and the client can join it, after join client can ping the AP.  
### Try wpa_supplicant to test wifi driver client mode.
*To exclude other factors, suggest use **static IP** for both AP and client, and make sure no other process(such as wpa_supplciant and hostapd) are using the wlan.  
To use wpa_supplicant, first, enter /tmp and write a wpa_supplicant.conf, example:
```sh
ctrl_interface=/var/run/wpa_supplicant
update_config=1
ap_scan=1
network={
        ssid="wpa_test"
        key_mgmt=NONE
}
```
Make sure your ssid and encryption are the same with your AP.
Type following commands to use wpa_supplicant:
```sh
ifconfig wlan0 down up
wpa_supplicant -bbr-lan -Dnl80211 -iwlan0 -c wpa_supplicant.conf &
#wait for client connect to the AP
brctl addif br-lan wlan0
```
Now you can ping from cilent to your AP.
After exit wpa_supplicant, please type following command to delete wlan from bridge:
```sh
brctl delif br-lan wlan0
```
# Test plan for 98f openwrt wifi driver
### AP mode test
*To do the AP mode test, please make sure your wifi driver module is loaded, refer to [Test wifi driver](#try-wifi-driver), and the test procedure is similiar to [Try hostapd to test wifi driver AP mode](#try-hostapd-to-test-wifi-driver-ap-mode), only difference is the hostapd configure file.
#### Encryption
##### None encryption
hostapd-none.conf:
```sh
driver=nl80211
interface=wlan0
ctrl_interface=/var/run/hostapd
bridge=br-lan
ssid=hostapd-none

#none encryption
auth_algs=1
wpa=0
#none encryption

hw_mode=g
channel=11
ieee80211n=1

ignore_broadcast_ssid=0
country_code=US
wmm_enabled=1
```
**Pass criteria**  
The client can connect to the AP and then can ping to AP  
**Result**  
<font color=#8FBC8F>pass</font>
##### WPA encryption
hostapd-wpa.conf:
```sh
driver=nl80211
interface=wlan0
ctrl_interface=/var/run/hostapd
bridge=br-lan
ssid=hostapd-wpa

#wpa encryption
auth_algs=3
wpa=3
wpa_passphrase=12345678
wpa_key_mgmt=WPA-PSK
wpa_pairwise=CCMP
rsn_pairwise=CCMP
wpa_group_rekey=86400
wpa_gmk_rekey=86400
#wpa encryption

hw_mode=g
channel=11
ieee80211n=1

ignore_broadcast_ssid=0
country_code=US
wmm_enabled=1
```
**Pass criteria**  
The client can connect to the AP and then can ping to AP  
**Result**  
<font color=#8FBC8F>pass</font>
#### SSID
change the ssid in configuration file.  
hostapd-none.conf:
```sh
driver=nl80211
interface=wlan0
ctrl_interface=/var/run/hostapd
bridge=br-lan
#change ssid here
ssid=hostapd
#change ssid here
auth_algs=1
wpa=0

hw_mode=g
channel=11
ieee80211n=1

ignore_broadcast_ssid=0
country_code=US
wmm_enabled=1
```
**Pass criteria**  
After ssid in configuration file changed, the ssid shows in client will change.   
**Result**  
<font color=#8FBC8F>pass</font>
#### Channel
type 
```sh
iw phy0 info
```
to show the available channels, then change the channel in configuration file.  
hostapd-none.conf:  
```sh
driver=nl80211
interface=wlan0
ctrl_interface=/var/run/hostapd
bridge=br-lan
ssid=hostapd

auth_algs=1
wpa=0

hw_mode=g
#change channel here
channel=11
#change channel here
ieee80211n=1

ignore_broadcast_ssid=0
country_code=US
wmm_enabled=1
```
**Pass criteria**  
After ssid in configuration file changed, the wifi's channel will be changed.   
**Result**  
<font color=#8FBC8F>pass</font>
### Client mode test
*To do the AP mode test, please make sure your wifi driver module is loaded, refer to [Test wifi driver](#try-wifi-driver), and the test procedure is similiar to [Try hostapd to test wifi driver AP mode](#try-wpa_supplicant-to-test-wifi-driver-client-mode), only difference is the hostapd configure file.
#### Connect to none encryption AP
wpa_supplicant-none.conf:
```sh
ctrl_interface=/var/run/wpa_supplicant
update_config=1
ap_scan=1
network={
        ssid="wpa_test"
        key_mgmt=NONE
}
```
**Pass criteria**  
The client can connect to the AP and can ping to AP.   
**Result**  
<font color=#8FBC8F>pass</font>
#### Connect to wep encryption AP
wpa_supplicant-wep.conf:
```sh
ctrl_interface=/var/run/wpa_supplicant
update_config=1
ap_scan=1
network={
        ssid="wpa_test"
        key_mgmt=NONE
        wep_key0="12345"
}
```
**Pass criteria**  
The client can connect to the AP and can ping to AP.    
**Result**  
<font color=#8FBC8F>pass</font>
#### Connect to wpa encryption AP
wpa_supplicant-wpa.conf:
```sh
ctrl_interface=/var/run/wpa_supplicant
update_config=1
ap_scan=1
network={
        ssid="wpa_test"
        key_mgmt=WPA-PSK
		psk="12345678"
}
```
**Pass criteria**  
The client can connect to the AP and can ping to AP.   
**Result**  
<font color=#8FBC8F>pass</font>
