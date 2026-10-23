webscreensaver
==============

A screen hack which uses WebKit and works with XScreensaver or XFCE.

**NOTE:** XScreensaver does not work on Wayland. On Wayland, run webscreensaver standalone with `-fullscreen` or `-kiosk` (see [Wayland](#wayland-standalone)).

**GNOME Users:** There have been reports of this not working with Gnome.

Dependencies
------------

 * Python 3
 * xscreensaver or xfce4-screensaver
 * GTK3
 * WebKit2 gtk

On Debian/Ubuntu, try this command:

```
    sudo apt install python3 python3-gi gir1.2-webkit2-4.1 gir1.2-gtk-3.0
```

Install (XScreensaver)
----------------------

Copy `webscreensaver` into `/usr/libexec/xscreensaver` (or `/usr/lib/screensaver` on some systems) and then edit `~/.xscreensaver`:

```
    programs:
                  webscreensaver                  \n\
```

If you wish to set the url:

```
    -url <url_to_the_page_you_want>
```

Otherwise it will choose a random one.

You can persist cookies by specifying a cookie file:

```
    -cookie-file <path_to_cookie_file>
```

*NOTE:* All parameters should be all on the same line as `webscreensaver`

Install (XFCE)
--------------

 1. Copy `webscreensaver` into `/usr/lib/xscreensaver`.
 2. Copy `webscreensaver.desktop` into `/usr/share/applications/screensavers`.

Wayland (standalone)
--------------------

Without XScreensaver, webscreensaver can cover the screen itself:

 * `-fullscreen`: screensaver mode. Covers every monitor, hides the cursor, and exits on any key, click, touch, scroll or mouse movement.
 * `-kiosk`: the page is interactive and stays up until the process is killed. It is a normal fullscreen window, so other windows (e.g. a terminal) can still be opened on top of it.
 * `-hide-cursor`: with `-kiosk`, hide the mouse pointer over the page, for touchscreens or display-only kiosks. The screensaver always hides it.
 * `-all-monitors`: show the page on every monitor. By default only the first monitor shows it and the rest are black, which is easier on a Raspberry Pi.

It uses [gtk-layer-shell](https://github.com/wmww/gtk-layer-shell) to sit above panels and other windows on wlroots compositors (labwc, Wayfire, sway) and KDE. Without it, it falls back to a regular fullscreen window.

```
    sudo apt install python3 python3-gi gir1.2-webkit2-4.1 gir1.2-gtk-3.0 gir1.2-gtklayershell-0.1 swayidle
```

Use something like `swayidle` to start it when idle. On Raspberry Pi OS (labwc), add to `~/.config/labwc/autostart`:

```
    swayidle timeout 300 'webscreensaver -fullscreen -cycle' &
```

webscreensaver exits by itself on input, so no `resume` command is needed. (Avoid `resume 'pkill -f webscreensaver'`: it also matches swayidle's own command line and kills it.)

### Locking

webscreensaver doesn't lock the screen itself, but it can run a locker before exiting. Then the desktop is never shown:

```
    webscreensaver -fullscreen -lock-command 'swaylock -f'
```

The lock command must return once the screen is locked (`swaylock -f` daemonizes after locking). `-lock-after SECS` only locks if the screensaver has been running at least that long. The lock also runs when webscreensaver is stopped with SIGTERM.

### Kiosk

For a dedicated display, start it at login, e.g. `~/.config/autostart/webscreensaver.desktop`:

```
    [Desktop Entry]
    Type=Application
    Name=webscreensaver kiosk
    Exec=/usr/bin/lwrespawn /home/pi/.local/bin/webscreensaver -kiosk -url https://example.com
```

`lwrespawn` (Raspberry Pi OS) restarts it if it exits. A page that fails to load, e.g. before the network is up at boot, is retried every 30 seconds.

`-debug` prints the page's JavaScript console and any failed requests.

On devices with under 1GB of RAM (or with `-low-memory`), WebKit is set to use as little memory as it can: no page or disk cache, and memory-pressure limits so it frees memory before the system starts swapping.

### Running WebKit from a container

If the system's WebKit is broken or too old, `webscreensaver-container` runs webscreensaver against a Debian root filesystem using bubblewrap. The compositor, idle daemon and `-lock-command` stay on the host.

For example, Raspbian trixie's WebKit (2.48.0) crashes on a Pi Zero/1 (armv6). Debian's armel port runs on those boards and has a current, security-supported WebKit:

```
    sudo apt install mmdebstrap arch-test bubblewrap
    # Debian's archive keyring isn't packaged by Raspbian; get debian-archive-keyring from deb.debian.org
    sudo mmdebstrap --arch=armel --variant=apt --keyring=/path/to/debian-archive-keyring.gpg \
        --include=python3-gi,gir1.2-gtk-3.0,gir1.2-webkit2-4.1,gir1.2-gtklayershell-0.1,ca-certificates,fonts-dejavu-core,bubblewrap \
        trixie /opt/debian-armel \
        "deb http://deb.debian.org/debian trixie main" \
        "deb http://security.debian.org/debian-security trixie-security main"

    webscreensaver-container -kiosk -url https://example.com
```

The script must sit next to `webscreensaver-container`, or be named with `WEBSCREENSAVER_SCRIPT`. The root defaults to `/opt/debian-armel` (`WEBSCREENSAVER_ROOT`). Update it with `sudo chroot /opt/debian-armel sh -c 'apt-get update && apt-get -y upgrade'`.

On a Pi Zero it is slow: a busy dashboard can take several minutes to first appear, so it suits `-kiosk` better than an idle screensaver.

### Raspberry Pi setup scripts

[`contrib/raspberry-pi`](contrib/raspberry-pi) has scripts that do all of the above on Raspberry Pi OS (labwc). Run them as the desktop user from a clone of this repo:

 * `setup-screensaver [--timeout SECS] [--lock] [args]`: idle screensaver via swayidle (Pi 4/5).
 * `setup-kiosk [--slim] [args]`: dedicated kiosk started at login, with the desktop and panel turned off. On a Pi Zero/1 it builds and uses the Debian container. `--slim` also turns off services a kiosk doesn't need (Bluetooth, audio, automounting).
 * `kiosk-mode on|off|status`: switch a kiosk back to the normal desktop for troubleshooting, and back again.
 * `build-container`: build or rebuild the Debian armel container.

For example, a Pi Zero W showing a dashboard:

```
    contrib/raspberry-pi/setup-kiosk --slim -hide-cursor -url https://dashboard.example.com
```

Sites List
----------

The list of sites chosen can be customised via a configuration file (TOML).

The format of the file is series of sections (one per site) as follows:

```
    [thisisasite]
    url = "http://abcd.efg"
    inject_css = "body { color: black }"
    remove_tags = ["header", "hullo"]
    remove_ids = ["hellohello", "heyhey"]
```

The `inject_css`, `remove_tags` and `remove_ids` keys are used to apply
customisations to the page after it has loaded. The value can be either
a single string or a list of strings.

 * `inject_css`  - the CSS string will be injected into the page.
 * `remove_tags` - all tags matching the specified tag name will be removed.
 * `remove_ids`  - all tags with the specified IDs will be removed.

Cycling
-------

If you set your screensaver settings to "cycle", you should use
`-cycle` which will pick the next screensaver in the list each time
WebScreensaver is run. (Cycling is implemented in XScreensaver by stopping
the current screensaver program and starting another one -- even in the
case of a single screensaver being selected.)

3rd Party Packages
------------------

You can install webscreensaver easily using these packages (unsupported):

 * Arch Linux (AUR) - `webscreensaver-git` - https://aur.archlinux.org/packages/webscreensaver-git
