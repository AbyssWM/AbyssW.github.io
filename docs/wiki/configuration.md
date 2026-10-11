# Configuration (Abysslang)

**Abysslang** is how Abyssland is configured: your configuration is a C++ file that is compiled into the compositor, in the spirit of dwm's `config.h`. There is no config file to parse at runtime and no scripting layer.

::: warning Work in progress
Abysslang is under active development. The API below is taken from the current headers, but names and details may still change before 1.0.0.
:::

## Concept

- Your config is `~/.config/abyss/config.hpp`. It defines one function, `configure(Config&)`, which runs on every config load (at start and after `abyssctl reload`).
- Everything registered inside it (binds, rules, listeners, timers, animations) is cleared before the next load, so nothing is registered twice.
- Because it is plain C++, you can use variables, loops, functions and your own lambdas as keybind actions.
- **Split files:** big configs can be divided into several headers and pulled in with `#include "binds.hpp"`.
- **Reload:** `abyssctl reload` rebuilds the compositor with your config, installs it and restarts Abyssland. Windows are closed by the restart.
- **Errors are notifications:** if the config does not compile, the first compiler error is shown as a notification and your session keeps running with the old build.
- **Last-good fallback:** the previous working binary is saved as `Abyssland.last-good`. If a new build dies right after starting, the start wrapper falls back to it.
- **Typed options:** options are fields such as `c.general.gaps_in`. Misspell one and the compile fails – no silent ignored typos.
- Designated initializers (for example `{.cls = "x"}`) must list their fields in declaration order, a C++ rule.

A minimal config:

```cpp
#include <abyss/config.hpp>

using namespace abyss;

void configure(Config& c) {
    c.general.gaps_in  = 5;
    c.general.gaps_out = 20;
    c.bind(SUPER, "Q", spawn("kitty"));
}
```

## Options

Options are typed fields grouped like the compositor's sections (`general`, `decoration`, `input`, `animations`, `misc`, `dwindle`, `master`, ...). Colors, gradients and gaps have their own types:

```cpp
c.general.gaps_in     = 5;
c.general.gaps_out    = 20;
c.general.border_size = 2;
c.general.col.active_border   = gradient({rgba(0x33ccff, 0xee), rgba(0x00ff99, 0xee)}, 45);
c.general.col.inactive_border = rgba(0x595959, 0xaa);
c.general.layout = "dwindle";

c.decoration.rounding = 10;
c.decoration.shadow.enabled = true;
c.decoration.blur.enabled   = true;
c.decoration.blur.passes    = 1;

c.input.kb_layout    = "us";
c.input.follow_mouse = 1;
```

Value helpers: `rgb(0xRRGGBB)`, `rgba(0xRRGGBB, alpha)` (alpha as `0..255` or `0.0..1.0`), `argb(0xAARRGGBB)`, `gradient({colors...}, angleDeg)` and `Gap` (CSS style: 1, 2 or 4 values). A type mismatch is a compile error as well.

An option can also be set by its string name with the same checks: `c.set("general:gaps_in", 5)`.

Per-device settings use `c.device(name, {...})`, for example `c.device("logitech-mx-master", {.sensitivity = -0.5});`.

## Bundled wallpapers

Abyssland includes seven backgrounds, named `wall0.png` through `wall6.png`.
The four newest variants show mist and water, a katana with tentacles, a stone
wall with a gold crack, and a minimal katana scene. They use artwork generated
with Google Flow for Abyss.

```cpp
c.misc.disable_abyssland_logo = false;
c.misc.disable_splash_rendering = true;
c.misc.force_default_wallpaper = -1; // choose randomly at compositor startup
// c.misc.force_default_wallpaper = 4; // always use wall4.png
```

The background stays fixed during the session; there is no timed slideshow.
An external wallpaper client can cover it. New bundled files belong in
`assets/install/`, numbered consecutively without gaps. CMake detects added
assets on the next build and includes them in the installation.

## Programs, environment and autostart

```cpp
const std::string terminal = "kitty";

c.env("XCURSOR_SIZE", "24");          // for the compositor and everything it starts
c.execOnce("waybar");                 // only on the first load of the session
c.exec("some-command");               // on every config load
c.onStart([] { exec("quickshell"); }); // once per compositor process
c.onShutdown([] { /* ... */ });
c.execShutdown("cleanup-command");
```

`onStart` is not run again by an in-process reload (`abyssctl reload --no-build`), which makes it the right place for programs that must start only once.

## Binds and actions

`c.bind(mods, key, action, opts)` registers a keybind. Modifiers are `NONE`, `SHIFT`, `CTRL`, `ALT`, `SUPER` (combine with `|`). Keys are xkb key names, `code:N`, `mouse:N`, `mouse_down`/`mouse_up` and so on. A combined string works too: `c.bind("SUPER+SHIFT+Q", killActive())`.

```cpp
c.bind(SUPER, "Q", spawn(terminal));
c.bind(SUPER, "C", killActive());
c.bind(SUPER, "V", toggleFloating());
c.bind(SUPER, "left", moveFocus(Dir::Left));

// workspaces 1..10 (key "0" is workspace 10)
for (int i = 1; i <= 10; ++i) {
    const std::string key = std::to_string(i % 10);
    c.bind(SUPER, key, workspace(i));
    c.bind(SUPER | SHIFT, key, moveToWorkspace(i));
}

// special workspace (scratchpad)
c.bind(SUPER, "S", toggleSpecial("magic"));

// mouse binds
c.bind(SUPER, "mouse:272", mouseMove(), {.mouse = true});
c.bind(SUPER, "mouse:273", mouseResize(), {.mouse = true});

// media keys, also on the lock screen, repeating while held
c.bind(NONE, "XF86AudioRaiseVolume", spawn("wpctl set-volume -l 1 @DEFAULT_AUDIO_SINK@ 5%+"),
       {.locked = true, .repeat = true});

// any C++ works as an action
c.bind(SUPER, "N", [] { notify("hello from C++"); return ok(); });
```

Bind options (`BindOpts`): `locked`, `release`, `repeat`, `longPress`, `mouse`, `nonConsuming`, `autoConsuming`, `transparent`, `ignoreMods`, `dontInhibit`, `click`, `drag`, `submapUniversal`, `allowInputCapture`, `description`, `devices`.

`bind` returns a `BindHandle` (`setEnabled(bool)`); `unbind` and `unbindAll` remove binds.

### Available actions

Actions are values. Pass them to `bind()` or run them from your own code with `run(action)`.

| Area | Actions |
|---|---|
| Programs | `spawn(cmd)`, `spawnRaw(cmd)`, `exit()` |
| Windows | `killActive()`, `forceKillActive()`, `closeWindow()`, `killWindow()`, `signal()`, `toggleFloating()`, `setFloating()`, `setTiled()`, `pseudo()`, `pin()`, `fullscreen()`, `fullscreenState()`, `centerWindow()`, `setProp()`, `tag()`, `toggleSwallow()` |
| Focus and movement | `moveFocus(Dir)`, `moveWindow(Dir)`, `moveWindowToMonitor()`, `swapWindow(Dir)`, `swapWith()`, `swapNext()`, `cycleNext()`, `focusWindow()`, `focusUrgentOrLast()`, `focusCurrentOrLast()`, `alterZOrder()` |
| Size and position | `resizeActive(dx, dy)`, `moveActive(dx, dy)`, `resizeWindowPixel()`, `moveWindowPixel()`, `mouseMove()`, `mouseResize()` |
| Groups | `toggleGroup()`, `changeGroupActive()`, `moveGroupWindow()`, `moveIntoGroup()`, `moveIntoOrCreateGroup()`, `moveOutOfGroup()`, `lockGroups()`, `lockActiveGroup()`, `denyWindowFromGroup()` |
| Workspaces and monitors | `workspace()`, `moveToWorkspace()`, `toggleSpecial()`, `moveToSpecial()`, `renameWorkspace()`, `focusMonitor()`, `focusWorkspaceOnCurrentMonitor()`, `moveCurrentWorkspaceToMonitor()`, `moveWorkspaceToMonitor()`, `swapActiveWorkspaces()`, `dpms()` |
| Layout | `setLayout()`, `layoutMsg()`, `toggleSplit()`, `swapSplit()` |
| Misc | `submap()`, `pass()`, `sendShortcut()`, `sendKeyState()`, `global()`, `event()`, `moveCursor()`, `moveCursorToCorner()`, `forceRendererReload()`, `forceIdle()`, `releaseInputCapture()`, `dispatch(name, arg)` |
| Helpers | `notify(text, ms, color, icon)`, `ok()`, `error(msg)` |

`dispatch(name, arg)` runs any dispatcher by name, see [abyssctl](/wiki/abyssctl).

## Window and layer rules

Rules pair a match with effects. Unset fields do not take part in matching. Regexes are used for strings.

```cpp
// ignore maximize requests from apps
c.windowRule({}, {.suppressEvent = "maximize"}, "suppress-maximize");

// always float pavucontrol
c.windowRule({.cls = "^(org.pulseaudio.pavucontrol)$"},
             {.floating = true, .center = true, .size = "800 600"},
             "pavucontrol-float");
```

`WindowMatch` fields include `cls`, `title`, `initialClass`, `initialTitle`, `floating`, `tag`, `xwayland`, `fullscreen`, `pinned`, `focus`, `group`, `modal`, `workspace`, `content`. `WindowEffects` include `floating`, `tile`, `fullscreen`, `maximize`, `center`, `pin`, `size`, `move`, `minSize`, `maxSize`, `monitor`, `workspace`, `group`, `opacity`, `borderColor`, `borderSize`, `rounding`, `animation`, `tag`, `noFocus`, `suppressEvent` and more. For anything not covered there is an `extra` list of `{name, value}` pairs.

Layer surfaces (bars, launchers) use `c.layerRule(LayerMatch{.nspace = "..."}, LayerEffects{...})`.

## Monitors and workspaces

```cpp
c.monitor("DP-1", "2560x1440@144", "0x0", 1.0);
c.monitor("", "preferred", "auto", 1.0);   // every other monitor: automatic
c.disableMonitor("HDMI-A-1");

c.workspaceRule("3", {.layout = "dwindle"});
```

`monitor(name, mode, position, scale, opts)` takes a mode (`"preferred"`, `"highres"`, `"highrr"`, `"WxH"`, `"WxH@Hz"`), a position (`"auto"`, `"XxY"`, ...) and a scale (number or `"auto"`). `MonitorOpts` cover `mirror`, `transform`, `bitdepth`, `cm`, `vrr`, `reserved`, HDR and luminance settings. `WorkspaceRule` covers `monitor`, `isDefault`, `persistent`, `gapsIn`, `gapsOut`, `borderSize`, `layout`, `animation` and more.

## Animations and curves

```cpp
c.animations.enabled = true;
c.bezier("easeOutQuint", 0.23, 1, 0.32, 1);
c.spring("easy", 1, 878.5, 59.29);          // a spring instead of a bezier
c.animation("global", true, 10, "default");
c.animation("windowsIn", true, 4.1, "easeOutQuint", "popin 87%");
c.animation("workspaces", true, 1.94, "almostLinear", "fade");
```

`animation(leaf, enabled, speed, curve, style)` addresses a leaf such as `global`, `windows`, `windowsIn`, `fade`, `layers` or `workspaces`; the curve is a bezier or spring name.

## Events

Subscribe with `c.on(Event::..., callback)`. A callback may take fewer arguments than the event provides, down to none.

```cpp
c.on(Event::WindowOpen, [](PHLWINDOW w) { /* a window was mapped */ });
c.on(Event::WorkspaceActive, [] { /* ... */ });
```

Available events: `Start`, `Shutdown`, `WindowOpen`, `WindowOpenEarly`, `WindowClose`, `WindowActive`, `WindowUrgent`, `WindowTitle`, `WindowClass`, `WindowPin`, `WindowFullscreen`, `WindowFloating`, `WindowUpdateRules`, `WindowMoveToWorkspace`, `LayerOpened`, `LayerClosed`, `MonitorAdded`, `MonitorRemoved`, `MonitorFocused`, `MonitorLayoutChanged`, `WorkspaceActive`, `WorkspaceSpecialActive`, `WorkspaceCreated`, `WorkspaceRemoved`, `WorkspaceMoveToMonitor`, `ConfigReloaded`, `PropsRefreshed`, `Submap`, `ScreenshareState`, `KeyboardKey`.

`c.on` returns an `EventHandle`; `cancel()` removes the listener early.

## Timers

```cpp
auto t = c.timer(60'000, [] { notify("one minute passed"); }, /*repeat=*/true);
t.setEnabled(false);   // disarm
t.cancel();            // remove
```

## Submaps

A submap is a separate set of binds that is active until you leave it:

```cpp
c.bind(SUPER, "X", submap("resize"));
c.submap("resize", [](Config& c) {
    c.bind(NONE, "right", resizeActive(20, 0), {.repeat = true});
    c.bind(NONE, "left",  resizeActive(-20, 0), {.repeat = true});
    c.bind(NONE, "escape", submap("reset"));
});
```

## Gestures and permissions

`c.gesture(fingers, direction, action, opts)` registers touchpad gestures. `c.permission(binary, type, mode)` sets rules such as `c.permission("/usr/bin/grim", "screencopy", "allow")` (types: `screencopy`, `cursorpos`, `keyboard`, `input-capture`; modes: `allow`, `ask`, `deny`).

## Custom layouts

::: warning Work in progress
Custom layouts are being implemented right now. The API below is what the current headers declare.
:::

A simple layout is a function that places boxes:

```cpp
c.addLayout("grid", [](LayoutCtx& l) {
    const auto cells = l.grid(l.count());
    for (size_t i = 0; i < l.count(); ++i)
        l.place(i, cells[i]);
});
c.general.layout = "grid";
```

For full control, derive from `abyss::TiledLayout` and register it with `c.addLayout<MyAlgo>("name")`. Select a layout globally with `c.general.layout`, per workspace with a workspace rule, or at runtime with `abyssctl dispatch setlayout grid`. The full-power base exposes compositor internals that are not a stable API.

## Splitting your config

```cpp
// config.hpp
#include <abyss/config.hpp>
#include "binds.hpp"
#include "rules.hpp"

using namespace abyss;

void configure(Config& c) {
    c.general.gaps_in = 5;
    setupBinds(c);
    setupRules(c);
}
```

Each extra file can hold ordinary C++ functions taking `Config&`.
