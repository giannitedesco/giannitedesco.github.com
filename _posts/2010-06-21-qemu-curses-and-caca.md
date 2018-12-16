---
layout: post
title: Qemu, curses and caca
category: virt
tags: virtualization qemu curses ncurses caca terminals
year: 2010
month: 6
day: 21
published: true
comments: true
summary: Fun with text-mode qemu consoles
---
I have recently discovered, and been very impressed with, the qemu ncurses display driver. Basically it lets you run a guest OS inside of a regular terminal emulator. Currently the curses display works almost perfectly when the BIOS and guest OS has the VGA card programmed in text mode but when switched to graphics mode all you get is a message telling you “Graphic mode 800 x 600″ for example.

Well, realising that there are the excellent aalib and libcaca ASCII-art text rendering engines out there I began thinking about the possibilities.. Firstly it is possible to set the (default) SDL display to use the libcaca video driver. Then I realised libcaca has an ncurses back-end. So the following command will let you run a full graphics mode OS inside your terminal emulator:

```
$ env CACA_DRIVER=ncurses SDL_VIDEODRIVER=caca kvm ...
```

The results look great but the only problem at the moment is mouse integration in that button presses don’t get through. I think I have tracked this down to a bug in the SDL libcaca wrapper so watch this space :)

![Screenshot](/images/winxp-textmode.png)

