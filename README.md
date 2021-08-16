# myLFS

## 0. Introduction
I started my experience with LFS since 2012, failed at first, then I tried again, this time I have built up the complete LFS successfully, but I got bored when the ugly terminal finally showed up. Now, I want to re-build Linux From Scratch, inspired by my mentor Mr. Liu.

## 1. VirtualBox Debian GNU/Linux

I built my first LFS on a physical laptop rather than VirtualBox, but this time, I want to save time for the quick and better understanding of LFS, so now VirtualBox is the choice.

We all know something about VirtualBox and Linux, but it is strongly recommanded to dive into the [VirtualBox Snapshots](https://docs.oracle.com/en/virtualization/virtualbox/6.0/user/snapshots.html). Take a snapshot on each building step. VirtualBox network config is another topic that won't be covered here but is also very much worth to take an effort to study.

As of Linux, I prefer Debian GNU/Linux being the host system for LFS. 

In my case, I use macOS as my host system, iTerm is a good tool for the whole task.

So, the pipeline seems like:

- Start a headless VM
- ssh to the VM via iTerm, start a Tmux session
- Follow the LFS recipes
- Take a snapshot on each building step
- Save the VM if needed

> 1. I didn't notice that I only give the VM 1 CPU until the building process was half passed, to save your time, give it as many as possible! 
> 2. I didn't use Tmux until Chapter 8.44. Automake-1.16.2, we should introduce this awesome tool from the very beginning!
> 3. Do the following to make sure your VM works perfectly.
> ```bash
>       sudo apt install -y dkms linux-headers-$(uname -r)-amd64
> ```

## 2. LFS

All commands here are run as user **root** if there is no special explanation.

### 2.1 Preparing the Host System
[Host System Requirements](http://www.linuxfromscratch.org/lfs/view/stable/chapter02/hostreqs.html)

As in Debian, do the following should solve all the problems.

```bash
apt install -y build-essential bison gawk m4 texinfo
rm -fv /bin/sh
ln -s /usr/bin/bash /bin/sh
```

Now, rsync the `version-check.sh` script to your virtualbox host, then execute it as root user.

### 2.2 All about the partition

- Creating a New Partition

    It is so frustated to deal with the partition on VirtualBox, to finish the building ASAP, I renew a Debian VM with an empty partition left.

    ```bash
    cfdisk /dev/sdb
    mkfs.ext4 /dev/sdb1  # the partition is /dev/sdb1 in my case
    ```

- Setting The $LFS Variable
    ```bash
    export LFS=/mnt/lfs
    ```
- Mounting the New Partition
    ```bash
    mkdir -pv $LFS
    mount -v -t ext4 /dev/sdb1 $LFS
    ```
- Creating a limited directory layout in LFS filesystem
    ```bash
    mkdir -pv $LFS/{bin,etc,lib,lib64,sbin,usr,var,tools}
    ```

### 2.3 Packages and Patches

In China, use `https://mirrors.ustc.edu.cn/lfs/lfs-packages/lfs-packages-10.1.tar` to speed up the downloading of the packages and patches.

```bash
wget -c "https://mirrors.ustc.edu.cn/lfs/lfs-packages/lfs-packages-10.1.tar" -P /tmp
tar xf /tmp/lfs-packages-10.1.tar -C /tmp
mv /tmp/10.1/* $LFS/sources/
```

### 2.4 Final Preparations

- Adding the LFS User
    ```bash
    groupadd lfs
    useradd -s /bin/bash -g lfs -m -k /dev/null lfs
    
    chown -v -R lfs:lfs $LFS
    ```
- Setting Up the Environment

    **RUN AS USER lfs AT THIS STAGE**

    ```bash
    cat > ~/.bash_profile << "EOF"
    exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
    EOF

    cat > ~/.bashrc << "EOF"
    set +h
    umask 022
    LFS=/mnt/lfs
    LC_ALL=POSIX
    LFS_TGT=$(uname -m)-lfs-linux-gnu
    PATH=/usr/bin
    if [ ! -L /bin ]; then PATH=/bin:$PATH; fi
    PATH=$LFS/tools/bin:$PATH
    export LFS LC_ALL LFS_TGT PATH
    EOF
    ```

### 2.4.5 READ CAREFULLY!!!

Part III. Building the LFS Cross Toolchain and Temporary Tools

- Introduction
- Toolchain Technical Notes
- General Compilation Instructions

    The build instructions assume that the [Host System Requirements](http://www.linuxfromscratch.org/lfs/view/stable/chapter02/hostreqs.html), including symbolic links, have been set properly:

    - bash is the shell in use.
    - sh is a symbolic link to bash.
    - /usr/bin/awk is a symbolic link to gawk.
    - /usr/bin/yacc is a symbolic link to bison or a small script that executes bison.
    
    To re-emphasize the build process:

    1. Place all the sources and patches in a directory that will be accessible from the chroot environment such as /mnt/lfs/sources/.
    1. Change to the sources directory.
    1. For each package:
        - Using the tar program, extract the package to be built. In Chapter 5 and Chapter 6, ensure you are the lfs user when extracting the package.
        - Change to the directory created when the package was extracted.
        - Follow the book's instructions for building the package.
        - Change back to the sources directory.
        - Delete the extracted source directory unless instructed otherwise. 

**However, run `ch5-build.sh` as user lfs under the path `/mnt/lfs/sources` should be working like a charm from now on.**

### 2.5 Compiling a Cross-Toolchain

- 5.2. Binutils-2.35 - Pass 1
- 5.3. GCC-10.2.0 - Pass 1
- 5.4. Linux-5.8.3 API Headers
- 5.5. Glibc-2.32
- Caution: sanity check
    ```bash
    echo 'int main(){}' > dummy.c
    $LFS_TGT-gcc dummy.c
    readelf -l a.out | grep '/ld-linux'
    ```
- 5.6. Libstdc++ from GCC-10.2.0, Pass 1

### 2.6 Cross Compiling Temporary Tools

- 6.2. M4-1.4.18(Failed at first, but passed after I regenerate the limits.h file)
    ```bash
    cat gcc/limitx.h gcc/glimits.h gcc/limity.h > \
    `dirname $($LFS_TGT-gcc -print-libgcc-file-name)`/install-tools/include/limits.h
    # the last step of chapter 5.3 
    ```
- 6.3. Ncurses-6.2
- 6.4. Bash-5.0
- 6.5. Coreutils-8.32
- 6.6. Diffutils-3.7
- 6.7. File-5.39
- 6.8. Findutils-4.7.0
- 6.9. Gawk-5.1.0
- 6.10. Grep-3.4
- 6.11. Gzip-1.10
- 6.12. Make-4.3
- 6.13. Patch-2.7.6
- 6.14. Sed-4.8
- 6.15. Tar-1.32
- 6.16. Xz-5.2.5
- 6.17. Binutils-2.35 - Pass 2
- 6.18. GCC-10.2.0 - Pass 2

### 2.7 Entering Chroot and Building Additional Temporary Tools

- 7.2. Changing Ownership
- 7.3. Preparing Virtual Kernel File Systems
- 7.3.1. Creating Initial Device Nodes
- 7.3.2. Mounting and Populating /dev
- 7.3.3. Mounting Virtual Kernel File Systems
- 7.4. Entering the Chroot Environment
- 7.5. Creating Directories
- 7.6. Creating Essential Files and Symlinks
- 7.7. Libstdc++ from GCC-10.2.0, Pass 2(failed at first, configure: error: no acceptable C compiler found in $PATH. Going back to the 6.18, make a soft link of gcc to $LFS/usr/bin/cc, problem solved.)
- 7.8. Gettext-0.21
- 7.9. Bison-3.7.1
- 7.10. Perl-5.32.0
- 7.11. Python-3.8.5 (when I come back the next morning, the ssh session lost, ssh to host system and follow the 7.4, re-chroot)
- 7.12. Texinfo-6.7
- 7.13. Util-linux-2.36
- 7.14. Cleaning up and Saving the Temporary System

### 2.8 Building the LFS System

Before continue, make sure you are in the chroot environment by following the instructions of Chapter 7.3 and 7.4.

- 8.3. Man-pages-5.08
- 8.4. Tcl-8.6.10
- 8.5. Expect-5.45.4
- 8.6. DejaGNU-1.6.2
- 8.7. Iana-Etc-20200821
- 8.8. Glibc-2.32
- 8.9. Zlib-1.2.11
- 8.10. Bzip2-1.0.8
- 8.11. Xz-5.2.5
- 8.12. Zstd-1.4.5
- 8.13. File-5.39
- 8.14. Readline-8.0
- 8.15. M4-1.4.18
- 8.16. Bc-3.1.5
- 8.17. Flex-2.6.4
- 8.18. Binutils-2.35
- 8.19. GMP-6.2.0
- 8.20. MPFR-4.1.0
- 8.21. MPC-1.1.0
- 8.22. Attr-2.4.48
- 8.23. Acl-2.2.53
- 8.24. Libcap-2.42
- 8.25. Shadow-4.8.1
- 8.26. GCC-10.2.0
- 8.27. Pkg-config-0.29.2
- 8.28. Ncurses-6.2
- 8.29. Sed-4.8
- 8.30. Psmisc-23.3
- 8.31. Gettext-0.21
- 8.32. Bison-3.7.1
- 8.33. Grep-3.4
- 8.34. Bash-5.0
- 8.35. Libtool-2.4.6
- 8.36. GDBM-1.18.1
- 8.37. Gperf-3.1
- 8.38. Expat-2.2.9
- 8.39. Inetutils-1.9.4
- 8.40. Perl-5.32.0
- 8.41. XML::Parser-2.46
- 8.42. Intltool-0.51.0
- 8.43. Autoconf-2.69
- 8.44. Automake-1.16.2 (since this step, I use tmux for the building, I should introduce this awesome tool from the very beginning)
- 8.45. Kmod-27
- 8.46. Libelf from Elfutils-0.180
- 8.47. Libffi-3.3
- 8.48. OpenSSL-1.1.1g
- 8.49. Python-3.8.5
- 8.50. Ninja-1.10.0
- 8.51. Meson-0.55.0
- 8.52. Coreutils-8.32
- 8.53. Check-0.15.2
- 8.54. Diffutils-3.7
- 8.55. Gawk-5.1.0
- 8.56. Findutils-4.7.0
- 8.57. Groff-1.22.4
- 8.58. GRUB-2.04
- 8.59. Less-551
- 8.60. Gzip-1.10
- 8.61. IPRoute2-5.8.0
- 8.62. Kbd-2.3.0
- 8.63. Libpipeline-1.5.3
- 8.64. Make-4.3
- 8.65. Patch-2.7.6
- 8.66. Man-DB-2.9.3
- 8.67. Tar-1.32
- 8.68. Texinfo-6.7
- 8.69. Vim-8.2.1361
- 8.70. Systemd-246
- 8.71. D-Bus-1.12.20
- 8.72. Procps-ng-3.3.16
- 8.73. Util-linux-2.36
- 8.74. E2fsprogs-1.45.6
- 8.75. About Debugging Symbols
- 8.76. Stripping Again
- 8.77. Cleaning Up

### 2.9 System Configuration

- 9.2. General Network Configuration
- 9.3. Overview of Device and Module Handling
- 9.4. Managing Devices
- 9.5. Configuring the system clock
- 9.6. Configuring the Linux Console
- 9.7. Configuring the System Locale
- 9.8. Creating the /etc/inputrc File
- 9.9. Creating the /etc/shells File
- 9.10. Systemd Usage and Configuration

### 2.10 Making the LFS System Bootable

- 10.2. Creating the /etc/fstab File
- 10.3. Linux-5.8.3
- 10.4. Using GRUB to Set Up the Boot Process
 
### 2.11 The End

- 11.1. The End
- 11.3. Rebooting the System
    - install OpenSSH is strongly recommanded
    - or make sure your network config is functional and wget or curl is installed

For me, after enter info the brand new LFS environment, the next thing is to compile BLFS packages, but I can't ssh into the new environment any more because I didn't install OpenSSH. So I rolled back to the previous snapshot, rsync the packages, make OpenSSH installed and configured, then reboot. There's another way to make sure you can use the network inside your LFS: give the VM two or more network cards at the very beginning. Normally, three cards for three modes: NAT, HostOnly, Bridge. This topic will be covered later if I got time to review the whole building procedure.

Now, let's embark on the BLFS adventrue.

## Step 3. BLFS

### II. Post LFS Configuration and Extra Software

- Configuring for Adding Users
- About System Users and Groups
- The Bash Shell Startup Files
- The /etc/vimrc and ~/.vimrc Files
- Customizing your Logon with /etc/issue

### Common and useful tools

- tmux
- openssh
- rsync
- git

### VI. X + Window and Display Managers

### Setting up the Xorg Build Environment

- util-macros-1.19.2
- xorgproto-2020.1
- libXau-1.0.9
- libXdmcp-1.1.3
- xcb-proto-1.14
- libxcb-1.14
- libpng-1.6.37
- Which-2.21 and Alternatives
- FreeType-2.10.2
- HarfBuzz-2.7.1
- FreeType-2.10.2 Pass 2
- Fontconfig-2.13.1
- Xorg Libraries: in China, use http://mirrors.ustc.edu.cn/Xorg/pub/individual/lib to speed up the downloading
- xcb-util-0.4.0
- xcb-util-image-0.4.0
- xcb-util-keysyms-0.4.0
- xcb-util-renderutil-0.3.9
- xcb-util-wm-0.4.1
- xcb-util-cursor-0.1.3
- libdrm-2.4.102
- MarkupSafe-1.1.1
- Mako-1.1.3
- libuv-1.38.1
- cURL-7.71.1
- LZO-2.10
- Nettle-3.6
- libarchive-3.4.3
- libxml2-2.9.10
- nghttp2-1.41.0
- CMake-3.18.1
- LLVM-10.0.1
    - fatal error solved! Give the VM more memory!

To be continued...

- Mesa-20.1.5
