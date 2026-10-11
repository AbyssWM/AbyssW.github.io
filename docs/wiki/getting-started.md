# Getting started

::: warning Work in progress
Abyss is in early development (0.x). There are no release packages yet, so for now you build everything from source. Expect rough edges and changing details.
:::

## What you get

Abyss is not a single program but a stack: the compositor **Abyssland** plus a set of libraries it needs (see [Ecosystem](/wiki/ecosystem)). All of them are built into one private prefix, `/opt/abyss`, and do not touch the system packages of the original Hyprland stack. They can live next to a Hyprland installation.

The development platform is **Void Linux**. The build uses GCC 14 and is distro-neutral in principle, but only Void is tested at the moment.

## Requirements

- A C++ toolchain: GCC 14 (or newer), `cmake`, `ninja`, `meson`, `pkg-config`
- The usual Wayland compositor development packages (wayland, wayland-protocols, libdrm, pixman, cairo, pango, libxkbcommon, ...)
- Additionally, these development libraries (names as on Void Linux, other distributions name them similarly):

| Purpose | Void package |
|---|---|
| Build tools | `meson`, `ninja` |
| Hyprland-derived dependencies | `glaze`, `re2-devel`, `muparser-devel`, `libei-devel`, `pugixml-devel`, `tomlplusplus-devel`, `sdbus-c++-devel` |
| X11 / XWayland | `xcb-util-errors-devel`, `xcb-util-wm-devel` |
| Display and input | `libdisplay-info-devel`, `libseat-devel`, `libinput-devel` |
| Image formats | `librsvg-devel`, `libzip-devel`, `file-devel`, `libjxl-devel`, `libheif-devel` |
| Shaders | `glslang-devel`, `SPIRV-Tools-devel` |
| Misc | `lua55-devel`, `qt6-base-devel`, `readline-devel` |

::: tip
The list above is what the project's own setup script installs. Package names differ between distributions, and the list may shrink as Abyssland drops more legacy parts (it already no longer ships Lua configs or plugins).
:::

## Build from source

Clone the [Abyss repositories](/wiki/ecosystem) next to each other, together with the build tools (`tools/build.sh` and `tools/setup-root.sh` in the project's main checkout).

### 1. One-time root setup

The setup script runs once as root. It installs the build packages, creates the install prefix and writes the session entry for your display manager:

```sh
sudo sh tools/setup-root.sh
```

It does three things:

- installs the packages listed above (Void Linux: via `xbps-install`)
- creates `/opt/abyss` and hands it to your user, so that **all later builds need no root**
- writes `/usr/share/wayland-sessions/abyssland.desktop` (the session **Abyssland** for SDDM and other display managers) and links the portal files into `/usr/share/xdg-desktop-portal` and `/usr/share/dbus-1/services`

On another distribution, do the same by hand: install the dependencies, create a writable `/opt/abyss`, and add a Wayland session that runs `dbus-run-session /opt/abyss/bin/start-abyssland`.

### 2. Build everything

```sh
sh tools/build.sh
```

The script builds and installs into `/opt/abyss`, in dependency order:

1. Abyssutils
2. Abyss-scanner
3. Abyssland-protocols
4. Abyssparse
5. Abysswire
6. Abyssgraphics
7. Abyssal
8. Abysscursor
9. Abyssland
10. Xdg-desktop-portal-abyss

You can also build single repositories (`sh tools/build.sh Abyssal`), compile a different config (`ABYSS_CONFIG=/path/to/config.hpp sh tools/build.sh Abyssland`), or keep going after errors to collect all of them (`KEEP=1 sh tools/build.sh`). Logs land in `build-logs/`.

### 3. Start a session

Pick **Abyssland** in your display manager (the session runs `dbus-run-session /opt/abyss/bin/start-abyssland`). To try it first without leaving your current desktop, run the compositor nested inside another Wayland session:

```sh
/opt/abyss/bin/Abyssland
```

## Your first config

Abyssland is configured by a C++ file, compiled into the compositor. The default lives in the source tree as `config.def.hpp` (like dwm's `config.def.h`). Your own config is `~/.config/abyss/config.hpp`. Learn how it works on the [Configuration](/wiki/configuration) page.

After every change:

```sh
abyssctl reload
```

## Packages

Distribution packages are **coming with 1.0.0**:

| Distribution | Plan |
|---|---|
| Arch Linux | AUR (`-git` and release packages) |
| Fedora | COPR |
| Void Linux | own xbps repository |
| Gentoo | own overlay |

Until then, building from source is the only way.
