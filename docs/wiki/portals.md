# Desktop portals

Abyss ships its own portal, `xdg-desktop-portal-abyss`. It handles screen sharing (monitors and single windows, with restore tokens per window), screenshots, global shortcuts and input capture. Without any change, this is what runs.

## Using another portal

Everything in Abyss can be swapped out. Abyssland offers all the standard protocols other portals need (`zwlr_screencopy`, `ext_image_copy_capture`, `zwlr_foreign_toplevel`, `ext_foreign_toplevel_list`, `xdg_output`, `linux_dmabuf`), so no fork is needed.

::: warning Your responsibility
Abyss only tests its own portal. If you switch to another one, problems with that setup are yours to solve, not a bug in Abyss.
:::

### xdg-desktop-portal-wlr

1. Install it (Void: `xdg-desktop-portal-wlr`).
2. Copy one of the example files from [`contrib/portals`](https://github.com/AbyssWM/Xdg-desktop-portal-abyss/tree/main/contrib/portals) to `~/.config/xdg-desktop-portal/abyssland-portals.conf`. A file in your home wins over the system one.

Screen sharing and screenshots through wlr, the rest stays on Abyss (recommended):

```ini
[preferred]
default=abyss;gtk
org.freedesktop.impl.portal.ScreenCast=wlr
org.freedesktop.impl.portal.Screenshot=wlr
```

Only wlr. It has no global shortcuts and no input capture, so switch them off explicitly:

```ini
[preferred]
default=wlr;gtk
org.freedesktop.impl.portal.GlobalShortcuts=none
org.freedesktop.impl.portal.InputCapture=none
```

3. Restart the portals so they pick up the change: `pkill xdg-desktop-portal; pkill xdg-desktop-portal-abyss`. D-Bus starts them again on the next request.
4. If the wlr portal refuses to start because of the desktop name, add it to the list in your config. Abyssland accepts a colon separated `XDG_CURRENT_DESKTOP`:

```cpp
c.env("XDG_CURRENT_DESKTOP", "Abyssland:wlroots");
```

What you lose with wlr: per-window restore tokens, the Abyss share picker (wlr uses its own `chooser_cmd`), global shortcuts and input capture.

### Other portals

The same works for `gtk`, `kde` or `gnome` backends: name them in `portals.conf`. Check that the backend supports Wayland compositors other than its own desktop.
