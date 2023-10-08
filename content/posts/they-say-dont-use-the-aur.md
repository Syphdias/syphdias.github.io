---
title: "They Say Don't Use the AUR"
date: 2023-10-08T17:41:04+02:00
draft: false
---

If I had to guess, why people use an Arch-based system, I would guess a big
reason would be the AUR — even though it is not officially supported. It is a
big part of the community and the appeal of Arch. This is a story how the AUR
can break some things and the reason it is not officially supported.

It all started with a normal update. In my case I used the AUR helper [`paru`]
to update all system packages and all AUR packages. Only `obs-studio-tytan652`
failed when trying to compile, but I rarely use OBS Studio (it will get fixed,
eventually).

## Broken Electron-based Applications

When I tried to launch my note taking app [Obsidian], it never started. I
dropped into a shell and tried again.

```console
❯ obsidian
/usr/lib/electron25/electron: error while loading shared libraries: libdav1d.so.6: cannot open shared object file: No such file or directory
```

I had seem behavior like this before after an update. When you update your
kernel and you are still running the old kernel. If the running (old) kernel
needs to load a new module it expects the old libraries but cannot find them as
only the new ones are available.

It is generally also a bad idea to symlink libraries. If you would symlink the
old library to the location of the old one, there is no guarantee that they are
even remotely compatible.

> **Tip 1**: Don't break your system with symlinking shared libraries.

For the new-kernel-old-module problem a reboot usually suffices. However,
electron apps would probably not use any kernel shared libraries. To make sure
anyways, I did reboot.

> **Tip 2**: Reboot after a upgrade if libraries cannot be found.

After the reboot I was still not able to launch Obsidian or any other
Electron-based app.

> **Electron** is a framework for building desktop applications. It uses the
> Chromium browser engine and Node.js to make it easy to build cross-platform
> applications — especially, if you are already familiar with building web apps.


## What is Actually Missing and Where Should it Come From?
Since the issue seems to come from [electon] and not Obsidian itself I looked at
electron directly which yielded the same error.

When looking for library dependencies for the command `electron`, there are two
red herrings. One is, that the package `electron` is only a meta package to
[link to the latest stable version of electron] The second red herring is that
`/usr/bin/electron25` is a shell script to exec `/usr/lib/electron25/electron`.
So to check for the shared library dependencies, we can follow the trail like
this:

```console
❯ which electron
/usr/bin/electron
❯ ls -l /usr/bin/electron
lrwxrwxrwx 1 root root 10 Jun 16 08:50 /usr/bin/electron -> electron25
❯ file /usr/bin/electron25
/usr/bin/electron25: Bourne-Again shell script, ASCII text executable
❯ cat /usr/bin/electron25
#!/usr/bin/bash
[…snip…]
name=electron25
[…snip…]
exec /usr/lib/${name}/electron "${flags[@]}" "$@"
❯ file /usr/lib/electron25/electron
/usr/lib/electron25/electron: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 4.4.0, BuildID[sha1]=adabc98fbf2c1422ad6b6c4de371150f4fe605aa, stripped
```

> **Tip 3**: Find the binary that is actually run.

To find the libraries you can run `ldd` and filtered for the library name.
```console
❯ ldd /usr/lib/electron25/electron | grep libdav1d
	libdav1d.so.7 => /usr/lib/libdav1d.so.7 (0x00007f41ea424000)
❯ ls -l /usr/lib/libdav1d.so.7
lrwxrwxrwx 1 root root 17 Oct  4 17:21 /usr/lib/libdav1d.so.7 -> libdav1d.so.7.0.0
❯ pacman -F /usr/lib/libdav1d.so.7
usr/lib/libdav1d.so.7 is owned by extra/dav1d 1.3.0-1
```

Let's walk through this. The binary wants `libdav1d.so.7` and it can be found in
`/usr/lib/`. The way this resolution from filename to path works is similar to
how the `PATH` variable works. If you want to learn more about where libraries
live, search for `LS_LIBRARY_PATH` and `ld`. For now, it is enough to know that
the library is present at a know location, and points to another (existing)
file.

But actually this library is not the one the error complains about. In the error
above it complained about `libdav1d.so.6` being absent.

I reached out to the Arch Linux Mailing List which was very helpful. With the
help of `lddtree` (from the package `pax-utils`) I could look recursively at the
required libraries. Again, I filtered for the relevant library name but show 5
lines above the matches to see potential parent dependencies.

```console
❯ lddtree /usr/lib/electron25/electron | grep -B5 libdav1d
    libavcodec.so.60 => /usr/lib/libavcodec.so.60
        libswresample.so.4 => /usr/lib/libswresample.so.4
            libsoxr.so.0 => /usr/lib/libsoxr.so.0
                libgomp.so.1 => /usr/lib/libgomp.so.1
        libvpx.so.8 => /usr/lib/libvpx.so.8
        libdav1d.so.6 => None
--
        libva-x11.so.2 => /usr/lib/libva-x11.so.2
            libX11-xcb.so.1 => /usr/lib/libX11-xcb.so.1
            libxcb-dri3.so.0 => /usr/lib/libxcb-dri3.so.0
        libvdpau.so.1 => /usr/lib/libvdpau.so.1
        libOpenCL.so.1 => /usr/lib/libOpenCL.so.1
    libdav1d.so.7 => /usr/lib/libdav1d.so.7
```

This still shows `libdav1d.so.7` at the root level of the dependencies but also
`libdav1d.so.6` as dependency of `libavcodec.so.60`. It cannot resolve the name
to a library location so `None` gets displayed.

So where is this library from?

```console
❯ pacman -F libavcodec.so.60
extra/ffmpeg 2:6.0-12
    usr/lib/libavcodec.so.60
❯ pacman -Qi ffmpeg |grep -e Name -e Prov -e Requi
Name            : ffmpeg-obs
Provides        : ffmpeg=6.0.r12.ga6dc929  libavcodec.so=60-64  libavdevice.so=60-64  libavfilter.so=9-64  libavformat.so=60-64  libavutil.so=58-64  libpostproc.so=57-64  libswresample.so=4-64  libswscale.so=7-64
Required By     : chromaprint  chromium  electron25  ferdium-bin  firefox  gst-libav  krita  obs-studio-tytan652  opencv  peek  qt5-webengine  telegram-desktop  thunderbird  vlc-luajit
```

Hm, do you notice something?

> **Tip 4**: You can use `ldd`, `lddtree` (`pax-utils`) to find shared library
> dependencies that a binary need (is linked against).

> **Tip 5**: You can use `pacman -F` to find the package for file and `pacman
> -Qi` to show information about a package, like actual name or which other
> packages require it.

## The Culprit

The library is provided by `ffmpeg` a widely used library for decoding of all
kinds of media. But the installed version is not the regular version from the
official arch (extra) repositories but [`ffmpeg-obs`], a version from the AUR
which is required by [`obs-studio-tytan652`].

So why is it broken? Every time a library is updated, every program that uses
it, needs to also be rebuilt to link against the latest version. If you are
running only official packages from Arch Linux they take good care that if one
library gets an update every program or library using it (recursively) will be
updated as well to reflect the changed version. This is the reason why a
[partially upgraded system is unsupported]. They cannot guarantee that the
libraries match up.

The AUR on the other hand is independent from the official package repositories.
If something changes in the official repos, the AUR package maintainer need to
notice and react to it. If it is a binary package, they need to rebuild it
themselves. If it is a package built from source it needs to be rebuilt on the
users machine as well as soon as the new libraries are available (can also be
done through a version increment, e.g. increase the suffix to `-2`).

At this point I had two options.
1. Remove `obs-studio-tytan652` and switch back to `extra/ffmpeg` (the
   officially) supported version
2. Rebuilt `ffmpeg-obs` to link against the latest version of `dav1d`.

When I realised what had happened there was already [a new version] of
`ffmpeg-obs`. But I will keep this in mind for future updates.

## Conclusion

If this error had happened in an AUR package I would have known that I probably
had to rebuild it to link to the new dependencies. In this case however, one
package in the dependency chain was replaced by an unofficial one. Which made it
less obvious to track down the issue.

> **Tip 6**: Be weary what packages you replace with packages from the AUR.


[`paru`]: https://github.com/Morganamilo/paru
[Obsidian]: https://obsidian.md/
[link to the latest stable version of electron]: https://gitlab.archlinux.org/archlinux/packaging/packages/electron/-/blob/main/PKGBUILD?ref_type=heads
[`ffmpeg-obs`]: https://aur.archlinux.org/packages/ffmpeg-obs
[`obs-studio-tytan652`]: https://aur.archlinux.org/packages/obs-studio-tytan652
[partially upgraded system is unsupported]: https://wiki.archlinux.org/title/system_maintenance#Partial_upgrades_are_unsupported
[a new version]: https://aur.archlinux.org/cgit/aur.git/commit/?h=ffmpeg-obs&id=8286e7cda14aaa87f9075174fa475907d99ce6bd
