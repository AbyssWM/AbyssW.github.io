# abyssctl reference

`abyssctl` talks to a running Abyssland: it queries state, sends commands and rebuilds the compiled config.

```
abyssctl [flags] <command> [args...|--help]
```

## Flags

| Flag | Meaning |
|---|---|
| `-j` | Output in JSON |
| `-r` | Refresh state after issuing the command (for example to update variables) |
| `--batch` | Execute a batch of commands separated by `;` |
| `-i`, `--instance` | Use a specific instance, either its signature or its index in `abyssctl instances` (0, 1, ...) |
| `-q`, `--quiet` | Disable output |
| `-h`, `--help` | Print help; `<command> --help` shows arguments that did not fit the overview |

## Config and session

| Command | Description |
|---|---|
| `reload [--no-build \| --dry-run]` | Rebuild the compiled config, install it and restart Abyssland |
| `restart` | Restart Abyssland (needs `start-abyssland`) |
| `configerrors` | List current config errors |
| `seterror <color> <message...>` | Set the abyssctl error string; resets on the next config load |
| `rollinglog [-f]` | Print the tail of the log; `-f` / `--follow` keeps following |
| `version` | Abyssland version, build flags, commit and branch |
| `status` | Internal status information |
| `systeminfo` | System info |
| `splash` | The current splash text |

### `reload`

Abyssland's config is C++ compiled into the binary. `abyssctl reload`:

1. builds the target (`cmake --build`, with progress)
2. on a compile error: shows the first error as a notification, keeps the session running and exits with code 1
3. otherwise saves the installed binary as `Abyssland.last-good`, runs `cmake --install` and asks Abyssland to restart via `start-abyssland` (if the new binary dies right after starting, it falls back to `last-good`)

`--no-build` only re-applies the already compiled config in place. `--dry-run` builds but neither installs nor restarts.

Environment: `ABYSS_SOURCE_DIR` and `ABYSS_BUILD_DIR` override the compiled-in directories, `ABYSS_PREFIX` overrides the install prefix.

## Querying state

| Command | Description |
|---|---|
| `activewindow` | The active window and its properties |
| `activeworkspace` | The active workspace and its properties |
| `clients` | All windows with their properties |
| `workspaces` | All workspaces with their properties |
| `workspacerules` | All workspace rules |
| `monitors [all]` | Active outputs; `all` includes inactive ones |
| `layers` | All surface layers |
| `layouts` | All available layouts |
| `devices` | Connected keyboards and mice |
| `binds` | All registered binds |
| `animations` | Configured animations and beziers |
| `decorations <window_regex>` | Decorations of matching windows |
| `cursorpos` | Cursor position in global layout coordinates |
| `globalshortcuts` | All global shortcuts |
| `instances` | All running Abyssland instances |
| `getoption <option>` | Value of a config option |

## Acting

| Command | Description |
|---|---|
| `dispatch <dispatcher> [args]` | Call a keybind dispatcher with arguments (list below) |
| `kill` | Kill mode: click a window to kill its app, leave with Escape |
| `notify <icon> <time_ms> <color> <message...>` | Show a notification. Icons: 0 warning, 1 info, 2 hint, 3 error, 4 confused, 5 ok, -1 or 6 none. Color `0` is the default for the icon |
| `dismissnotify [amount]` | Dismiss all or up to `amount` notifications |
| `setcursor <theme> <size>` | Temporarily set cursor theme and size; synchronize the XCursor fallback, gsettings, XWayland root cursor and X resources, child environment, and activation environment in a real session |
| `setprop <regex> <property> <value> [lock]` | Set a window property; with `lock`, dynamic window rules cannot override it |
| `getprop <regex> <property>` | Get a window property |
| `switchxkblayout <device> <next\|prev\|id>` | Set the keyboard layout index of a device |
| `output create <wayland\|x11\|headless\|auto>` | Create a virtual output |
| `output remove <name>` | Remove a virtual output (names from `abyssctl monitors`) |

## Dispatchers

Used as `abyssctl dispatch <name> [args]`:

`exec`, `execr`, `pass`, `sendshortcut`, `sendkeystate`, `killactive`, `closewindow`, `workspace`, `movetoworkspace`, `movetoworkspacesilent`, `togglefloating`, `setfloating`, `settiled`, `fullscreen`, `fakefullscreen`, `fullscreenstate`, `dpms`, `pin`, `movefocus`, `movewindow`, `swapwindow`, `centerwindow`, `resizeactive`, `moveactive`, `resizewindowpixel`, `movewindowpixel`, `cyclenext`, `swapnext`, `tagwindow`, `focuswindow`, `focusmonitor`, `splitratio`, `toggleopaque`, `movecursortocorner`, `movecursor`, `renameworkspace`, `exit`, `forcerendererreload`, `movecurrentworkspacetomonitor`, `focusworkspaceoncurrentmonitor`, `moveworkspacetomonitor`, `swapactiveworkspaces`, `alterzorder`, `togglespecialworkspace`, `focusurgentorlast`, `togglegroup`, `changegroupactive`, `focuscurrentorlast`, `lockgroups`, `lockactivegroup`, `moveintogroup`, `moveoutofgroup`, `movewindoworgroup`, `movegroupwindow`, `denywindowfromgroup`, `global`, `submap`, `event`.

Layout dispatchers `setlayout` and `layoutmsg` are described in the [configuration](/wiki/configuration#custom-layouts) page.

## Companion tools

::: warning Work in progress
`abyssctl abysspaper` and `abyssctl abysssunset` are reserved for the planned wallpaper tool and blue-light filter, which do not exist yet (see the [Roadmap](/roadmap)).
:::

- `abysspaper wallpaper <mon>,<path>[,<fit_mode>]` – change a wallpaper dynamically
- `abysssunset temperature <temp>` / `identity` / `gamma <gamma>` – blue-light and gamma filter
