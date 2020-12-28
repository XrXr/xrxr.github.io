---
title: "Hotkeys For Programmable Keyboards"
date: 2020-12-24T15:15:12-05:00
---

I have a [K-Type][ktype-link], which is a programmable keyboard that lets me
configure custom hotkeys. The configurations live with and on the keyboard,
which is neat. Among other things, this lets my configuration work across
different operating systems.

My current configuration has several navigation hotkeys that activate with the
`CAPSLOCK` key. `CAPSLOCK+{j,k,l,;}` send arrow keys, in a way similar to Vim's
default normal mode bindings. `CAPSLOCK+{i,n}` gives page up and down, while
`CAPSLOCK+{h,quote}` sends `HOME` and `END`. This hotkey cluster lets me scroll
and move cursors without having to take my right hand away from the home row
(index finger on `j`).

In addition to these, I have `CAPSLOCK+SPACE` sending `CONTROL` and
`CAPSLOCK`+`S` sending `SHIFT`. One of my favorite use of these is pressing
`CAPSLOCK+SPACE+{i,n}` to send `CONTROL+{PAGEUP,PAGEDOWN}`, which lets me
switch between tabs in Firefox and Chrome without moving from home row. These
also allow me to select text in input boxes.

The K-Type has a bunch of RGB LEDs, which I prefer to be off most of the time.
When I first tried to program it in 2017, the provided user-friendly tools
didn't give a way to make the board boot with LEDs off. I found myself turning
off the lights every time I booted up my computer.  Eventually, I bit the
bullet and dug into the firmware's code and [figured out][lights-off-code] a
way to have the lights off by default. Thankfully the firmware is all open
source, the board is hard to brick, and I knew C well enough.

I found my current `CAPSLOCK` setup on a 60% custom keyboard I built with the
[Instant60][instant60]. It's too small to have dedicated keys for some
navigation keys which pushed me to come up with a solution. Unfortunately,
my 60% board broke. It started to frequently drop key presses for a column of
keys after a few months of use. I don't have the means to repair the PCB, so
I'm back to using the K-Type.

When trying to add hotkeys to my old K-Type configuration, I found that it
doesn't build any more. It was kind of a pain to coerce the updated build
system to pick up my custom configuration, but I got there. My
[configuration][config] now lives in a single repository instead of being
spread across two, which is nice.

[ktype-link]: https://kono.store/products/k-type-mechanical-keyboard
[lights-off-code]: https://github.com/XrXr/kiibohd-controller/blob/8240bc27bc5bf834a9228c972e10c6d1337546ec/Scan/Devices/ISSILed/led_scan.c#L828
[instant60]: https://cannonkeys.com/products/preorder-instant60-pcb
[config]: https://github.com/XrXr/kiibohd-controller/commit/ee286b741281dbf5a937a750483bfc3399f4725d
