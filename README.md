# jetson-tx1-imx219
Setting up a Jetson TX1 with the IMX219 image sensor on an J106 &amp; M100 motherboard. I need to edit the IMX219's modes, which involves rebuilding both the driver and the dtb files for the kernel.

# Note well: this procedure is a work in progress, and has not been extensively tested!
I am actively developing this system and am keeping my project notes here so that I can do a complete rebuild if necessary. I'm writing these notes publicly on the off-chance that someone else will find them useful. Please feel free to contribute if you feel so moved.

To Do:
- [ ] Guide on how to modify the IMX219's modes
- [ ] Make the IMX219 driver a module so that I don't have to rebuild the whole kernel each time I change it?
- [ ] Add links to the various parts and their documentation.
- [ ] Screenshots of JetPack?
- [ ] Maybe add targets to the Makefile to deal with JetPack and Tegra210_...tbz2?
- [ ] Try using the 24.2.3 BSP rather than 24.2.1.
- [ ] See if Auvidea's patches would work with the 28.2.1 BSP (seems unlikely--they supposedly made some big changes)

## Part 1: Parts & datasheets

## Part 2: Software environment
I'm using Auvidea's drivers and patches for the IMX219, which are compatible with NVIDIA's JetPack 2.3.1 development environment. JetPack 2.3.1 only runs on Ubuntu 14.04, so I've set that up in a VM on my Windows 10 machine.

 - [JetPack 2.3.1 Release Notes](https://developer.nvidia.com/embedded/jetpack-2_3_1)
 - [L4T 24.2.1 Release Notes](https://developer.nvidia.com/embedded/linux-tegra-r2421)

1. Using JetPack, install into a convenient directory (e.g. ~/JetPack-2.3.1)
   - Common
   - CUDA Toolkit for Ubuntu 14.04
   - OpenCV for Tegra

1. JetPack forces you to download their rootfs if you try to use it to download the Kernel and drivers, so we can just download the driver pack instead (link in the L4T release notes). If you want the default rootfs, go for it, otherwise:
   ```bash
   cd JetPack-2.3.1
   mkdir 64_TX1
   tar xf Tegra210_Linux_R24.2.1_aarch64.tbz2 -C 64_TX1
   ```

1. Copy the Makefile and patches into `JetPack-2.3.1./64_TX1/Linux_for_Tegra`. If you don't care about the Auvidea patches, just apply the first (patches/no-chromium.patch) to add the `--no-chrome` option to `apply_binaries.sh`:
   ```bash
   quilt push
   ```

1. At this point, if you want to use the stock kernel you can skip to Part 4 to generate the rootfs, or go straight to Part 5 if you just want to flash the dang thing already.


## Part 3: Patching & compiling the kernel
1. First we must download the kernel. This is done using the `source_sync.sh` script given the current L4T release tag (listed in the Tegra Linux Driver Package Release Notes pdf). I have added a make target for it as well:
   ```bash
   make sync
   ```qq
1. Next, use `quilt` to apply the remaining patches. These are Auvidea's patches for the J106 and M100 boards, with support for the IMX219 image sensor. I modified them a bit to normalize them and clean them up.
   ```bash
   quilt push -a
   ```
   The last patch adds a file `p2371-2180-j106.conf`, which sets the environment variables for flashing the board. To choose it replace the `jetson-tx1.conf` link:
   ```bash
   ln -snf p2371-2180-j106.conf jetson-tx1.conf
   ```
1. Configure the kernel for building.
   ```bash
   make config
   make menuconfig
   ```
   The default `config` target is Auvidea's, set in an environment variable at the top of `Makefile`. Use the `menuconfig` target if you care to edit it. You may need to install `sudo apt install libncurses5-dev` to run `make menuconfig`.
   
1. Build the kernel, supplements (modules), and headers. You can parallelize by editing `NCPU=2` in `Makefile`.
   ```bash
   make kernel
   make supplements
   make headers
   ```
   
1. Copy the built kernel, etc. into the right places for JetPack:
   ```bash
   make install
   ```
1. If you already have a rootfs set up, you're ready to make the system image and flash it--see Part 5.

## Part 4: Minimal root filesystem
We'll build up the root filesystem (`rootfs`) up from Ubuntu Base--the minimal Ubuntu distribution. Doing it this way is a bit of a hassle, but it's a fair bit smaller (~1 GB) than the default `rootfs` (~3 GB) -- leading to less resource usage and faster flash times.
Note that I tend to use `apt` rather than `apt-get` -- it's simply friendlier.

1. Prepare the host by installing QEMU (for ARM emulation) and zsync (for downloading Ubuntu Base) 
   ```bash
   sudo apt install qemu-user-static zsync
   ```
1. Download Ubuntu Base 16.04 and extract it. Copy the QEMU binary over so that we can `chroot` into it, `resolv.conf` so that we can access the network after `chroot`ing, and a quick `sed` one-liner to enable all sources for `apt`.
   ```bash
   zsync http://cdimage.ubuntu.com/ubuntu-base/releases/16.04/release/ubuntu-base-16.04.4-base-arm64.tar.gz.zsync
   mkdir rootfs
   sudo tar -xf ubuntu-base-16.04.4-base-arm64.tar.gz -C rootfs
   sudo cp /usr/bin/qemu-aarch64-static rootfs/usr/bin/.
   sudo cp /etc/resolv.conf rootfs/etc/resolv.conf
   sudo sed -i -e "s/^# deb/deb/" rootfs/etc/apt/sources.list
   ```
1. `chroot` into the new rootfs. Note that (1) QEMU isn't perfect and may throw errors about unsupported system calls, though it hasn't been a problem, and (2) Ubuntu 16 uses systemd while 14 does not--it would be ideal to use `systemd-nspawn` rather than `chroot`, but it isn't available.
   ```bash
   sudo chroot rootfs /bin/bash
   ```
   1. Get `apt` and other installed packages up to date. Install a few extras to keep `apt` et al. happy.
      ```bash
      apt update
      apt install -y apt-utils dialog locales
      dpkg-reconfigure locales
      # Choose your locales (e.g. en_US *AND* en_US.UTF-8) and pick the UTF-8 version as the default.
      apt full-upgrade -y
      ```
   1. Install "necessary" packages and pick your favorite editor.
      ```bash
      apt install -y udev dbus net-tools ntp sudo less vim nano
      update-alternatives --config editor
      ```
   1. Install other useful packages.
      ```bash
      apt install -y man-db ethtool hwinfo lshw elinks python htop
      ```
   1. Add a user. Note that you must add a user to the group `video` on the Jetson to access the ISP & NVMM. Maybe replace "user" and "pass" with what you think are reasonable! NVidia's defaults are "ubuntu" and "ubuntu".
      ```bash
      adduser --disabled-password --gecos "Name" user
      echo "user:pass" | chpasswd
      usermod -a -G adm,sudu,video user
      ```
   1. Allow users in group `sudo` to run power commands without a password:
      ```bash
      visudo
      ```
      Add towards the end:
      ```
      # Allow power management without a password
      %sudo ALL=(ALL) NOPASSWD: /sbin/poweroff, /sbin/reboot, /sbin/shutdown, /sbin/rtcwake
      ```
   1. Add systemd overrides for `getty@.service` and `serial-getty@.service` for auto-login on ttys:
      ```bash
      mkdir /etc/systemd/system/getty@.service.d
      editor /etc/systemd/system/getty@.service.d/override.conf
      ```
      Add, replacing `user` with your username:
      ```ini
      [Service]
      ExecStart=
      ExecStart=-/sbin/agetty --autologin user --noclear %I $TERM
      ```
      Or as a one-liner, replacing `user` with your username:
      ```bash
      printf '[Service]\nExecStart=\nExecStart=-/sbin/agetty --autologin user --noclear %%I $TERM\n' > /etc/systemd/system/getty@.service.d/override.conf
      ```
      And similarly for `serial-getty@.service`, but specify the baud rate (115200) and terminal type (vt100). Again, replace `user` with your username:
      ```bash
      mkdir /etc/systemd/system/serial-getty@.service.d
      printf '[Service]\nExecStart=\nExecStart=-/sbin/agetty --autologin user --local-line %%I 115200 vt100\n' > /etc/systemd/system/serial-getty@.service.d/override.conf
      ```
      A few more notes:
         - The dash in `-/sbin/agetty` means that systemd will not treat non-zero exit codes as a failure.
         - The empty entry `ExecStart=` clears any previous values of `ExecStart`.
         - Single quotes means bash won't replace `$TERM` with its current value.
         - `printf` replaces `%%` with `%`
         - I use RealTerm for talking serial and `vt100` is as good as it gets. PuTTY supports `vt102` and `xterm`. Try `ls -R /lib/terminfo` to list all of the terminal types supported by Ubuntu.
   1. Edit `/etc/network/interfaces` to automatically raise `eth0` with DHCP:
      ```bash
      editor /etc/network interfaces
      ```
      Add:
      ```
      auto eth0
      iface eth0 inet dhcp
      iface eth0 inet6 auto
      ```
      Or, as a one-liner:
      ```bash
      printf '\nauto eth0\niface eth0 inet dhcp\niface eth0 inet6 auto\n' >> /etc/network/interfaces
      ```
   1. *Optional:* Add a local/accessible ntp server. I cannot stress how annoying it is to have an incorrect clock in certain scenarios. For example, my institution (UQ) requires a login to access the internet via an https website, however you need the correct time to validate the certificate. Without adding the local ntp server, I would have to manually set the date and time on every power cycle before accessing the network.
      ```bash
      editor /etc/ntp.conf
      ```
      Add (replace the URL with something appropriate for you):
      ```
      pool ntp.uq.edu.au
      ```
      Or as a one-liner:
      ```bash
      printf '\npool ntp.uq.edu.au\n' >> /etc/ntp.conf
      ```
   1. Install graphics stuff. My gut tells me that this is necessary to use the TX1's GPU, but I'm not positive. It's fairly low priority on my TO-DO list for me to test that since I need a GUI anyway. 
      ```bash
      apt install --no-install-recommends xorg lightdm openbox menu python-xdg consolekit
      editor /etc/lightdm/lightdm.conf
      ```
      Set up autologin, since we haven't installed a greeter:
      ```ini
      [Seat:*]
      autologin-user=user
      user-session=openbox
      pam-service=lightdm-autologin
      allow-guest=false
      ```
      Or as a one-liner:
      ```bash
      printf '[Seat:*]\nautologin-user=user\nuser-session=openbox\npam-service=lightdm-autologin\nallow-guest=false\n' > /etc/lightdm/lightdm.conf
      ```
      Note that
         - If you haven't used openbox before, expect to be wowed by a blank gray screen after booting... try right-clicking.
         - NVidia's `apply_binaries.sh` script adds `/etc/lightdm/lightdm.conf.d/50-nvidia.conf` that sets `autologin-user=ubuntu`. You'll have to either delete that file or make your username `ubuntu` too.
         - `--no-install-recommends` cuts down on installing many excess packages
   1. Install a graphical web browser. Dillo is OK and about 5 MB (including fltk). QEMU throws a fit when you install it, but it works in the end.
      ```bash
      apt install dillo
      ```
   1. Install gstreamer and plugins. You might get by with fewer plugins.
      ```bash
      apt install --no-install-recommends gstreamer1.0-tools gstreamer1.0-plugins-base gstreamer1.0-plugins-base-apps gstreamer1.0-plugins-good gstreamer1.0-plugins-ugly
      ```
   1. Exit the `chroot` environment
      ```bash
      exit
      ```
1. Make a backup. I like to do this before letting NVidia's scripts loose.
   ```bash
   sudo tar -cjf rootfs.tbz rootfs
   ```
1. Copy over your kernel and NVidia specific configuration, drivers, etc.
   ```bash
   sudo ./apply_binaries.sh --no-chrome
   ```
   The `--no-chrome` option is added by one of my patches--it stops the script from copying the Chromium browser (230 MB without its dependencies) into the filesystem.

## Part 5: Make and flash the system image.
1. Use `make image` to generate the system image.
1. Boot the TX1 into Force Recovery mode & attach it by USB. I had to futz with my VM's USB settings to get it to enumerate properly--check with `lsusb` to make sure it's recognized.
1. Run `make flash` to flash the system image. Note that this will not regenerate the system image, it will flash whatever image you last generated with `make image`!
