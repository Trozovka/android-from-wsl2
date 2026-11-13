# Android from WSL2

**Controlling an Android phone from WSL2 on Windows: USB, Wi-Fi, and the problems in between**

By **John Winsome Munar (Trozovka)**, Master Mariner
[trozovka.github.io](https://trozovka.github.io) · [GitHub](https://github.com/Trozovka)

---

I do most of my work in Ubuntu on WSL2 (Windows Subsystem for Linux). I also test Android
apps I build, set up phones for use at sea, and help my family with theirs. So I wanted
the phone's screen on my laptop, and `adb` (Android Debug Bridge) working from Linux,
both with a cable and over Wi-Fi.

It worked for months. Then one week it broke in three different ways at once. This page
explains how the setup works, what broke, how I found the cause, and how I fixed it, so the
next person (or the next me) can do it in minutes.

## Contents

1. [The parts](#1-the-parts)
2. [Building scrcpy](#2-building-scrcpy)
3. [The USB path](#3-the-usb-path)
4. [The Wi-Fi path](#4-the-wi-fi-path)
5. [Problem 1: USB attach fails after an update](#5-problem-1-usb-attach-fails-after-an-update)
6. [Problem 2: the phone shows the wrong IP address](#6-problem-2-the-phone-shows-the-wrong-ip-address)
7. [Problem 3: adb freezes on start](#7-problem-3-adb-freezes-on-start)
8. [Safety notes](#8-safety-notes)
9. [Quick checklist](#9-quick-checklist)
10. [What I learned](#10-what-i-learned)

---

## 1. The parts

```
 Phone (USB debugging or Wireless debugging ON)
   │                         │
   │ USB cable               │ Wi-Fi (same network)
   ▼                         │
 Windows                     │
   usbipd-win  ── shares the USB device ──┐
                                          ▼
 WSL2 (Ubuntu)                 usbip client + kernel vhci driver
   adb server  ◄──── USB or TCP ─────────┘
   scrcpy      ── mirrors the screen, sends taps and keys
```

- **adb** talks to the phone. It runs a small background "server" on the laptop, and every
  `adb` command (and scrcpy) talks to that server through a local port.
- **scrcpy** shows the phone's screen in a window and lets you use mouse and keyboard on it.
- **WSL2 has no direct USB access.** A Windows tool, **usbipd-win**, shares a USB device over
  a virtual network into WSL2, where the Linux `usbip` client attaches it.
- **Wireless debugging** (Android 11 and later) lets adb connect over Wi-Fi after a one-time
  pairing with a 6-digit code. No cable, no usbipd.

## 2. Building scrcpy

The scrcpy package in Ubuntu's own repository was too old for a phone on a recent Android
version: it starts, then fails because Android's internals changed. The snap version needed a
chain of extra packages just to open a window.

What worked: **build scrcpy from source** at a release that still uses SDL2 (newer releases
need SDL3, which my Ubuntu version doesn't ship), and use the **prebuilt server file** that
scrcpy publishes with each release. That way you don't need the whole Android SDK to build
it. Install into your home folder (`meson --prefix=$HOME/.local`, then `ninja install`), so
no system files change.

Two scrcpy options I always use: `--screen-off-timeout` (so the phone doesn't lock while I
work) and `-w` (keep awake while charging).

## 3. The USB path

On Windows (once, as administrator, after each Windows restart):

```
usbipd list                 # find the phone's BUSID
usbipd bind --busid <BUSID>
```

Then, each time you plug in:

```
usbipd attach --wsl --busid <BUSID>
```

In WSL the phone then appears in `adb devices`. The Linux side needs the `usbip` client
tools (`linux-tools-generic` on Ubuntu) and the kernel's `vhci-hcd` driver, which the WSL2
kernel already includes.

## 4. The Wi-Fi path

1. Phone and laptop on **the same Wi-Fi network**.
2. Phone: **Settings → Developer options → Wireless debugging → On**.
3. The screen shows an **IP address and port** for connecting. Tap **Pair device with
   pairing code** to get a second IP:port and a 6-digit code.
4. On the laptop, once per laptop:

   ```
   adb pair 192.0.2.25:41234 123456      # pairing IP:port and code
   ```

5. Then, every time:

   ```
   adb connect 192.0.2.25:37001          # the connect IP:port
   scrcpy
   ```

The ports change every time Wireless debugging is switched on, so a small script that
remembers the last working address and asks for the new one when needed saves a lot of
typing. (The addresses above are examples from the range reserved for documentation.)

---

## 5. Problem 1: USB attach fails after an update

**Symptom.** The Windows side found the phone and tried to attach it, but nothing appeared
in WSL. My script said Windows didn't attach the device. Running `usbip` by hand printed:

```
WARNING: usbip not found for kernel 6.x.x-microsoft-standard-WSL2
  You may need to install the following packages for this specific kernel: ...
```

**Cause.** Ubuntu's `usbip` command is a small wrapper that looks for a version matching the
running kernel. The WSL2 kernel is Microsoft's, and Ubuntu has no tools package for it, so
the usual trick is to point `usbip` at the generic tools with `update-alternatives`. That
link pointed at a specific version folder, for example `linux-tools/6.8.0-139-generic/usbip`.
A routine `apt upgrade` installed a newer version and **removed the old folder**. The link
now pointed at nothing, so the system fell back to the wrapper, which complained.

**Fix.** Replace the fixed link with a tiny script that always uses whatever version is
installed:

```
sudo rm -f /usr/local/bin/usbip
printf '#!/bin/sh\nexec "$(ls -d /usr/lib/linux-tools/*/usbip | sort -V | tail -1)" "$@"\n' \
  | sudo tee /usr/local/bin/usbip >/dev/null
sudo chmod +x /usr/local/bin/usbip
usbip version
```

The `rm` matters: `tee` follows the broken link and fails with "No such file or directory"
if you skip it. After this, future upgrades can't break it again.

## 6. Problem 2: the phone shows the wrong IP address

**Symptom.** The Wireless debugging screen showed an address like `10.111.222.1`, while the
laptop was on `192.168.x.x`. Pairing timed out. It looked as if the phone and laptop were on
different networks, but they were on the same Wi-Fi.

**Cause.** The phone runs a **firewall app that works as a local VPN** (on Android, firewall
apps that need no root do this). Android's Wireless debugging screen showed the **VPN
tunnel's address**, not the Wi-Fi address. adb still listens on the Wi-Fi interface; only the
displayed address is misleading.

**Fix.** Use the phone's real Wi-Fi address (**Settings → Wi-Fi → the gear icon next to your
network → IP address**) with the **ports** from the Wireless debugging screen. To check
before pairing, from Linux:

```
timeout 4 bash -c '</dev/tcp/192.0.2.25/37001' && echo open || echo closed
```

Also check the firewall app doesn't route local network traffic through its tunnel. Mine has
an option to leave private (LAN) addresses outside the VPN, and that must be on.

## 7. Problem 3: adb freezes on start

**Symptom.** Every `adb` command hung forever, even `adb devices`. Restarting didn't help.
The newer adb from the Android SDK hung the same way. There was no adb process left
afterwards and nothing obviously holding the port.

**How I found it.** Run the server in the foreground with full tracing, on a different port:

```
ADB_TRACE=all timeout 6 adb -P 5099 nodaemon server
```

The trace reached `Event loop starting`. So the server itself was fine. And a client
connected to it without trouble:

```
setsid adb -P 5099 nodaemon server &      # server in the background
adb -P 5099 devices                        # works
```

What hangs is the **normal start**, where `adb` forks a background server and waits for it
to report back. In my setup (WSL2 with the "mirrored" networking mode, after a WSL kernel
update) that handshake never completed. I didn't find the deepest reason, and I didn't need
to: the server works when started directly.

**Fix.** Start the server yourself and point every tool at it. A small file that each phone
script loads:

```
export ANDROID_ADB_SERVER_PORT=5099      # adb and scrcpy both read this
export ADB_MDNS=0                        # no network discovery, fewer surprises
adb_up() {
  if ! ss -ltn | grep -q '127.0.0.1:5099 '; then
    setsid adb -P 5099 nodaemon server >/dev/null 2>&1 < /dev/null &
    sleep 2
  fi
}
adb_up
```

**A trap I fell into while debugging:** `pkill -f "adb nodaemon"` also matched the shell that
was running my command (its command line contained the same words) and killed it. Use
`pgrep -a adb` first, then kill by process number.

## 8. Safety notes

- **Turn USB debugging and Wireless debugging off when you're done.** Some banking apps
  refuse to open while they're on, and an open debug port is a door.
- Pairing codes expire in about a minute and only work on the same network. That's good.
- Only pair laptops you own. Remove old ones under Wireless debugging → Paired devices.
- Don't run adb commands you don't understand on a phone with your personal data on it.
  When someone helps you, agree first on what they will and won't touch.

## 9. Quick checklist

| Check | Command or place |
|---|---|
| Phone seen by Windows? | `usbipd list` (on Windows) |
| Bound once after reboot? | `usbipd bind --busid <BUSID>` (admin) |
| `usbip` works in WSL? | `usbip version` |
| adb server running? | `ss -ltn \| grep 5099` |
| Phone seen by adb? | `adb devices -l` |
| Same Wi-Fi? | compare laptop `ip addr` with phone's Wi-Fi IP (not the VPN one) |
| Port open? | `timeout 4 bash -c '</dev/tcp/<ip>/<port>'` |

## 10. What I learned

- **Three faults at once look like one big fault.** Separate them: USB, network, adb. Test
  each with the smallest possible command.
- **Fixed version paths break on upgrades.** Point at "the newest installed", not at a number.
- **The screen can lie.** A displayed address isn't always the address that works.
- **When the normal way hangs, run the piece by hand.** `nodaemon` plus tracing showed the
  server was healthy and narrowed the fault to the start-up handshake.
- **Write it down.** Next time this takes five minutes instead of an afternoon.

---

**About the author.** John Winsome Munar (Trozovka) is a Master Mariner who builds practical
software for seafarers. More at [trozovka.github.io](https://trozovka.github.io).

Text © 2026 John Winsome Munar. Shared under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/): you may share and adapt it with credit.
