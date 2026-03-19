# BeagleBadge Notes

Badge specific:
1. Binaries: https://drive.google.com/drive/folders/1rmY-gFMv-ymjwGoOkSwclhZjPUB8hQpV?usp=drive_link
2. Schematics: beaglebadge-schematics-v7.zip
3. EPaper Display
  - ProductPage/ Datasheet - https://www.seeedstudio.com/4-2-Monochrome-ePaper-Display-with-400x300-Pixels-p-5784.html?srsltid=AfmBOopF93IiVaO_vagog1pz3k_gwRO49O3mmZIpQ5-UwafveXr4R2zu
  - Required Breakout Adapter - https://www.seeedstudio.com/ePaper-Breakout-Board-p-5804.html

4. Hackathon page: https://grippy98.github.io/Badge-Hackathon-Tracker/
5. Badge app store: https://github.com/Grippy98/badge-app-store
6. Python badge launcher https://github.com/Grippy98/badge-slop/tree/linux-wip

7. Processor SDK documentation links:
   - Debian: https://texasinstruments.github.io/processor-sdk-doc/processor-sdk-debian-AM62LX/esd/docs/master/devices/AM62LX/debian/index.html
   - Arago:
     https://texasinstruments.github.io/processor-sdk-doc/processor-sdk-linux-AM62LX/esd/docs/master/devices/AM62LX/index.html
     https://texasinstruments.github.io/processor-sdk-doc/processor-sdk-linux-AM62LX/esd/docs/master/boards/beagle/BeagleBadge/BeagleBadge.html

SoC specific:
1. Datasheet: https://www.ti.com/lit/ds/symlink/am62l.pdf?ts=1773929968154&ref_url=https%253A%252F%252Fconfluence.itg.ti.com%252F
2. Trm: https://www.ti.com/lit/ug/sprujb4a/sprujb4a.pdf?ts=1773929986205&ref_url=https%253A%252F%252Fconfluence.itg.ti.com%252F
3. Errata: https://www.ti.com/lit/er/sprz582b/sprz582b.pdf?ts=1773929995715&ref_url=https%253A%252F%252Fconfluence.itg.ti.com%252F
4. Pinmux tool: https://www.ti.com/tool/SYSCONFIG

--------------------------------------------------------------------------------------------------------------------------------------------------------------

My complete build instructions for badge:

1. TFA: https://github.com/TexasInstruments/arm-trusted-firmware.git branch: ti-master
   make -j$(nproc) CROSS_COMPILE=$CC_64 ARCH=aarch64 PLAT=k3 TARGET_BOARD=am62l-badge SPD=opteed
   cp build/k3/am62l-badge/release/bl1.bin build/k3/am62l-badge/release/bl31.bin $bins

2. U-boot: https://git.ti.com/git/ti-u-boot/ti-u-boot.git branch: ti-u-boot-2025.01
   make ARCH=arm -j$(nproc) CROSS_COMPILE=aarch64-none-linux-gnu- am62lx_badge_defconfig O=out/
   make ARCH=arm -j$(nproc) CROSS_COMPILE=aarch64-none-linux-gnu-  BL31=$bins/bl31.bin BL1=$bins/bl1.bin TEE=$bins/bl32.bin BINMAN_INDIRS=$bins O=out/
   cp $out/tiboot3.bin $out/tispl.bin $out/u-boot.img $boot

3. Linux: https://git.ti.com/git/ti-linux-kernel/ti-linux-kernel.git branch: ti-linux-6.12.y
   make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu-
   cp arch/arm64/boot/dts/ti/k3-am62l3-badge.dtb $root/boot/dtb/ti

4. Yocto:
   git clone https://git.ti.com/git/arago-project/oe-layersetup.git tisdk
   cd tisdk
   ./oe-layertool-setup.sh -f configs/arago-scarthgap-config.txt
   cd build
   . conf/setenv
   export MACHINE=beaglebadge-ti
   ARAGO_SYSVINIT=1 bitbake -k tisdk-tiny-image

5. Debian:
   # Make sure docker is installed
   sudo apt-get install docker.io qemu-user-static binfmt-support
   git clone https://github.com/TexasInstruments/armbian-build.git
   cd armbian-build
   git checkout 2025.12-beaglebadge
   wget http://swubn04.india.englab.ti.com/armbian-files/armbian-proxy.patch
   git apply armbian-proxy.patch
   tmp_proxy_ip=$(nslookup webproxy.ext.ti.com | grep "Address: " | cut -c 10-) && echo $tmp_proxy_ip
   sed -i "s|172.16.172.136|${tmp_proxy_ip}|g" lib/functions/rootfs/distro-agnostic.sh
   sed -i "s|172.16.172.136|${tmp_proxy_ip}|g" lib/functions/host/docker.sh
   sed -i "s|webproxy.ext.ti.com|${tmp_proxy_ip}|g" lib/functions/general/python-tools.sh
   ./compile.sh build BOARD=beaglebadge BRANCH=vendor-edge BUILD_MINIMAL=yes KERNEL_CONFIGURE=no RELEASE=trixie GIT_SKIP_SUBMODULES=yes SKIP_ARMBIAN_REPO=yes
