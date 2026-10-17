# Installation

Install Abyss through the package source for your distribution. The repositories
cover the compositor, its libraries, desktop portal and companion tools. Your
package manager installs dependencies and owns the installed files.

::: warning Early development
The first experimental package publication is being prepared. Public packaging
recipes are available for all 17 projects; a recipe repository does not mean
that its binary package source is already online. Real-session tests are still
pending. Check the status below before following installation commands.
:::

## Package sources

| Distribution | Package source | Status |
|---|---|---|
| Fedora | [COPR: bb8therobot1717/Abyss](https://copr.fedorainfracloud.org/coprs/bb8therobot1717/Abyss/) · [Recipes](https://github.com/AbyssWM/fedora-builds) | Project created for Fedora 44 and 45 on x86_64; packages require successful builds before installation |
| Ubuntu | Launchpad PPA · [Recipes](https://github.com/AbyssWM/ubuntu-builds) | Recipes prepared; PPA publication pending |
| Debian | APT repository · [Recipes](https://github.com/AbyssWM/debian-builds) | Recipes prepared; signed APT source pending |
| Void Linux | XBPS repository · [Recipes](https://github.com/AbyssWM/void-builds) | All 17 primary packages built on x86_64/glibc; public binary source pending |
| Gentoo | [Overlay](https://github.com/AbyssWM/gentoo-builds) | All 17 ebuilds available; complete native verification in progress |
| Arch Linux | AUR · [PKGBUILDs](https://github.com/AbyssWM/arch-builds) | All 17 recipes available; AUR publication and native verification pending |
| Linux Mint | Matching Ubuntu package source | Depends on the Mint version's Ubuntu base and separate compatibility tests |
| LMDE | Matching Debian package source | Depends on its Debian base and separate compatibility tests |

## Fedora

The COPR project is intended to provide all 17 primary packages and seven
library development packages in one repository. Build targets are
`fedora-44-x86_64` and `fedora-45-x86_64`. Check the project's build results for
your version: a configured target is not proof of a successful build.

Once the packages have successfully built and been published:

```sh
sudo dnf copr enable bb8therobot1717/Abyss
sudo dnf install abyssland
```

This installs the compositor and its package dependencies. Companion tools can
be selected separately:

| Package | Purpose |
|---|---|
| `xdg-desktop-portal-abyss` | Desktop portal and share picker |
| `abyss-guiutils` | Dialogs, launcher and welcome/update screens |
| `abysslock` | Lock screen |
| `abysspaper` | Wallpaper application |
| `abyssdusk` | Color temperature control |
| `abysspicker` | Color picker |
| `abysspolkit` | Polkit authentication agent |

For example, after publication:

```sh
sudo dnf install abysspaper
```

Installing a tool does not by itself configure it or start its user service.
See [Tools](/wiki/tools) and the individual source repositories for current
behavior. Package availability does not imply every feature has been tested.

## Ubuntu, Debian and Mint

The Ubuntu and Debian packaging repositories each contain definitions for all
17 projects. Their APT sources are not published yet; no repository-add command
is available at this stage. Ubuntu series must be selected and their dependency
versions verified before PPA uploads.

Ubuntu-based Mint will need packages for its matching Ubuntu base; LMDE needs
packages for its matching Debian base. A PPA is not automatically enabled on
Mint. Installation instructions will be added with the corresponding published
source and tested release combinations.

## Void Linux

Use the [Void package repository](https://github.com/AbyssWM/void-builds) for
current build results and package recipes. All 17 primary packages have built
successfully with xbps-src on x86_64/glibc, including seven development
subpackages. Installation and desktop tests remain open.

A signed, hosted XBPS source still needs to be published. Its URL and
`xbps-install` instructions will be added here when available. The recipes can
already be built using Void's package build system as documented in that repo.

## Gentoo

The [Gentoo overlay](https://github.com/AbyssWM/gentoo-builds) contains all 17
ebuilds with pinned source archives, manifests and dependencies. Gentoo builds
the packages from source through Portage. Complete native validation is still
in progress; follow the overlay README for its current status and setup.

## Arch Linux

The [Arch package repository](https://github.com/AbyssWM/arch-builds) contains
PKGBUILDs and `.SRCINFO` for all 17 projects. Native clean-chroot builds and AUR
publication remain pending. Until an AUR entry is actually published, use the
Git repository for recipe review and package-build instructions.

## Start the session

After installing the compositor package, select **Abyssland** in your display
manager. Packages install the compositor and session files under the normal
system prefix, `/usr`; the old development prefix `/opt/abyss` is not the
package installation path.

The initial package builds use the public default C++ configuration. The
[configuration reference](/wiki/configuration) describes the API and the
intended user config at `~/.config/abyss/config.hpp`. Rebuilding that config from
a package installation still requires a matching source/build environment and
development dependencies; the complete packaged reload workflow is not yet
validated.

Real-session and hardware testing follows the COPR publication. There is no
separate setup script in the package installation flow.
