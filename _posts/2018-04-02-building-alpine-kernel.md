---
title: Building a modified kernel for Alpine Linux
layout: post
category: blog
---


I've recently helped had a friend upgrade his Alpine Linux installation to v3.7, and he complained that his grsec kernel was 
still relatively old, stuck at version 4.9.32.
This probably has a background story in the recent of grsecurity making their patches available only to paying customers. You 
can read about that at https://en.wikipedia.org/wiki/Grsecurity and judge for yourself.
I found [copperhead's linux-hardened repository](https://github.com/copperhead/linux-hardened/), that seems to maintain a 
free version of these patches for more recent kernel version.

While i'm myself not that interested in running a hardened kernel, i took this opportunity to write a small tutorial on how 
to build and install modified versions Alpine Linux packages.

First you need to setup your build environment within Alpine:

    apk add alpine-sdk

abuild cannot run abuild as root. It is recommended to  a user called 'abuild'.

Switch to your build user, and get the aports repository, where we find the kernel build recipe:

    git clone https://github.com/alpinelinux/aports
    cd aports/main/linux-hardened

To incorporate the [linux-hardened patch](https://github.com/copperhead/linux-hardened/releases/download/4.14.32.a/linux-hardened-4.14.32.a.patch),
open the APKBUILD file and make some changes:

- Change pkgver to 4.14.32 (the base version of the current stable patch)
- Look for the lines after inside the source="" assignment and remove the old grsec patch
- Add there the download link to the new patch: https://github.com/copperhead/linux-hardened/releases/download/4.14.32.a/linux-hardened-4.14.32.a.patch 
- I found that i also had to remove zfs-fix.patch. This is only about a license check, not concerning actual functionality.

When you're done, automatically re-generate the checksums at the bottom of APKBUILD with:

    abuild checksum

Since this is quite a version jump, the Linux buildsystems will ask a lot of questions about new config options.
I decided to instead going through all the new options that came in between from 4.9 to 4.14, i'd rather copy over the 
linux-vanilla 4.14 config and just deal with the new options from the linux-hardened patch (i haven't checked if the normal 
linux-hardened build sets any non-grsec-related options differently): 

    cp ../linux-vanilla/config-vanilla.x86_64 config-hardened.x86_64
    cp ../linux-vanilla/config-virt.x86_64 config-virthardened.x86_64
    apk checksum

Normally we'd start a full build with 'abuild -r', but since we will interactively modify the kernel config, we will 
explicitly run the single steps of abuild to save the modified config:

    abuild deps # Install build dependencies through the pseudo-package .makedepends-linux-hardened, that you can remove later
    abuild unpack # Extract sources to src/
    abuild prepare # Apply patches
    abuild build # The actual build. Will ask about new kernel options

Note that make config will run two times, once for linux-hardened, and after that kernel is built,
agin for the linux-virthardened flavor. 
Save your modified config from the build directories, so you can commit it to your aports branch later:

    cp src/build-hardened/.config config-hardened.x86_64
    cp src/build-virthardened/.config config-virthardened.x86_64
    apk checksum

Now you can finish assembling the actual package:

    abuild package # "Install" the kernel into pkg/
    abuild rootpkg # Assemble the package, will appear in ~/packages


Before you can sign and install the package, you need to generate a signing keypair:

    abuild-keygen

You need to distribute the public key in some way and place it in /etc/apk/key/ on machines where you want to install the new 
package!

Now we can sign the local repository and refresh the index:

    abuild index

Your package is ready in $HOME/packages/main .
You can add this path to /etc/apk/repositories to install it locally.

