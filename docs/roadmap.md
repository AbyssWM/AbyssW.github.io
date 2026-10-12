# Roadmap

Abyss is in early development (**0.x**). **1.0.0** is the first usable release and the start of the distribution packages. This page lists what is planned; the order is roughly by usefulness for everyday use.

::: warning Work in progress
Nothing on this page is a promise of a date. Details may change.
:::

## Done

- All twelve repositories forked, renamed and building together
- Own dialogs (`abyss-dialog`), no Hyprland packages needed at runtime
- Abyssland runs as its own session
- Abysslang: compiled C++ config with typed options, binds, rules, events, timers, submaps, `abyssctl reload` with last-good fallback
- Native idle rules (`c.idle`) with inhibitor support
- Screensaver in idle: draw your own in C++, clock and audio-visualizer presets included
- Bundled wallpapers drawn by the compositor, no wallpaper app needed
- **AbyssDusk** (blue light filter) and **AbyssPicker** (color picker) are available, see [Tools](/wiki/tools)

## In progress

- **Custom layouts** (`c.addLayout`) for your own tiling algorithms, including a grid layout
- Cursor handling that works for games and Xwayland apps with a single config value
- Everyday testing of the real session

## Planned

### Built-in bar and menus

No external bar needed: a small widget system inside Abyssland lets you build a bar, launcher and power menu in C++ right in your config (workspaces, window title, clock, tray, popups, click and scroll callbacks). Third-party bars keep working through the standard protocols (`ext-workspace`, `wlr-foreign-toplevel`).

### AbyssLock

Lock screen, forked from hyprlock, in the Katana look and configured in compiled C++ like the compositor.

### AbyssPaper

Command line wallpaper tool, forked from hyprpaper: images, GIFs and video with optional audio, plus layers on top such as clock, audio visualizer and now-playing.

### More tools

A polkit agent. The color picker (AbyssPicker) and the blue-light filter (AbyssDusk) already exist, see [Tools](/wiki/tools).

### Documentation

This wiki grows with the project: a generated Abysslang reference and a guide to the differences from Hyprland.

### Packages and 1.0.0

- Release tarballs and tagged versions
- **Arch Linux:** AUR
- **Fedora:** COPR
- **Void Linux:** own xbps repository
- **Gentoo:** own overlay
- Later, maybe: a Nix flake and Debian/Ubuntu packages

## Upstream

Security and bug fixes from hyprwm are picked over selectively. Abyss does not blindly merge new Hyprland releases.
