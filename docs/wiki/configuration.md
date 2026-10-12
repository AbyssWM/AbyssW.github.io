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
    c.bind(SUPER, "Q", [] { return exec("kitty"); });
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

## Reading options

`abyss::getOption(name)` returns the current value of any option by its string name, the counterpart of `hl.get_config` in Hyprland's Lua API. `abyss::get<T>(name)` reads it typed. Both return `std::optional`, empty when the option does not exist.

```cpp
c.bind(SUPER, "G", [] {
    auto gaps  = get<int>("general:gaps_in");            // std::optional<int>
    auto blur  = get<bool>("decoration:blur:enabled");
    auto layout = get<std::string>("general:layout");
    if (gaps && *gaps < 20)
        set("general:gaps_in", *gaps + 5);
    return ok();
});
```

`T` can be a number (`int`, `float`, `double`, ...), `bool`, `std::string`, `Vec2` or `Gap`. Numbers and `bool` convert into each other, and anything converts to `std::string`. A `Gap` whose four sides are equal reads as a plain number. If the type does not fit (for example `Vec2` on an integer option), the result is empty. Other complex values come back as their config string: a gradient is a `std::string`, while colors are returned as an `int`.

`getOption(name)` returns a `std::variant` of `bool`, `int64_t`, `double`, `std::string`, `Vec2` and `Gap`.

`abyss::optionSetByUser(name)` is true when the config (or `set()`) changed the option and false while it still has its default or does not exist.

::: warning
Typed fields such as `c.general.gaps_in = 5` are only applied after `configure()` returned. Reading an option inside `configure()` therefore still shows the old value. Read options in callbacks, binds, timers and event listeners instead.
:::

## Cursor theme and size

Set the theme and logical size in one place:

```cpp
c.cursor.theme = "Sweet-cursors";
c.cursor.size  = 24;
```

An explicit theme is used for both Abysscursor and XCursor clients. Abyssland synchronizes the XCursor fallback, gsettings, XWayland root cursor and `Xcursor.theme` / `Xcursor.size` X resources, preserving unrelated X resources. It exports `XCURSOR_THEME`, `XCURSOR_SIZE`, `ABYSSCURSOR_THEME` and `ABYSSCURSOR_SIZE` to newly launched children.

In a real desktop session these four variables are also sent asynchronously to the D-Bus activation environment and an already running systemd user manager. This works without systemd; no user manager is started. Nested compositors skip activation export so they do not change their parent's launch environment. `ABYSSLAND_NO_SD_VARS=1` disables activation export.

Defaults `theme = ""` and `size = 0` preserve the startup environment's selection, including separate native and XCursor themes. A size of zero uses the startup size (native if available and enabled, otherwise XCursor); invalid environment sizes fall back to 24. `c.cursor.enable_abysscursor = false` selects XCursor rendering. Changing typed cursor options through a config reload or `abyss::set` refreshes the cursor. `abyssctl setcursor <theme> <size>` applies a temporary change; it does not edit your compiled config.

Applications may retain cursors they already loaded. Theme synchronization does not guarantee that every running app immediately reloads its cursor assets.

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

c.env("GDK_BACKEND", "wayland,x11,*"); // compositor and everything it starts
c.onStart([] { exec("quickshell"); }); // once per compositor process
c.onShutdown([] { /* ... */ });
```

`exec(command)` starts the command immediately through `/bin/sh -c`, with shell expansion, pipes and redirects. Put it inside a callback for autostart or a keybind. `exec` returns a result, not an `Action`, so `c.bind(..., exec("kitty"))` does not compile. The callback defers execution until the key is pressed.

`onStart` is not run again by an in-process reload (`abyssctl reload --no-build`), which makes it the autostart entry point. A rebuilt compositor is a new process and runs it again. Existing compatibility helpers remain available: `c.exec(command)` runs on every config load, `c.execOnce(command)` starts once per compositor process, and `c.execShutdown(command)` runs on shutdown. Prefer `c.onStart` with `exec` for autostart.

## Binds and actions

`c.bind(mods, key, action, opts)` registers a keybind. Modifiers are `NONE`, `SHIFT`, `CTRL`, `ALT`, `SUPER` (combine with `|`). Keys are xkb key names, `code:N`, `mouse:N`, `mouse_down`/`mouse_up` and so on. A combined string works too: `c.bind("SUPER+SHIFT+Q", killActive())`.

```cpp
c.bind(SUPER, "Q", [terminal] { return exec(terminal); });
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
c.bind(NONE, "XF86AudioRaiseVolume", [] { return exec("wpctl set-volume -l 1 @DEFAULT_AUDIO_SINK@ 5%+"); },
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
| Session | `exit()` |
| Windows | `killActive()`, `forceKillActive()`, `closeWindow()`, `killWindow()`, `signal()`, `toggleFloating()`, `setFloating()`, `setTiled()`, `pseudo()`, `pin()`, `fullscreen()`, `fullscreenState()`, `centerWindow()`, `setProp()`, `tag()`, `clearTags()`, `signalWindow()`, `toggleSwallow()` |
| Focus and movement | `moveFocus(Dir)`, `moveWindow(Dir)`, `moveWindowToMonitor()`, `swapWindow(Dir)`, `swapWith()`, `swapNext()`, `cycleNext()`, `focusWindow()`, `focusUrgentOrLast()`, `focusCurrentOrLast()`, `alterZOrder()` |
| Size and position | `resizeActive(dx, dy)`, `moveActive(dx, dy)`, `resizeWindowPixel()`, `moveWindowPixel()`, `mouseMove()`, `mouseResize()` |
| Groups | `toggleGroup()`, `changeGroupActive()`, `moveGroupWindow()`, `moveIntoGroup()`, `moveIntoOrCreateGroup()`, `moveOutOfGroup()`, `setGroupActive()`, `lockGroups()`, `lockActiveGroup()`, `denyWindowFromGroup()` |
| Workspaces and monitors | `workspace()`, `moveToWorkspace()`, `toggleSpecial()`, `moveToSpecial()`, `renameWorkspace()`, `changeWorkspaceId()`, `focusMonitor()`, `focusWorkspaceOnCurrentMonitor()`, `moveCurrentWorkspaceToMonitor()`, `moveWorkspaceToMonitor()`, `swapActiveWorkspaces()`, `dpms()` |
| Layout | `setLayout()`, `layoutMsg()`, `toggleSplit()`, `swapSplit()` |
| Misc | `submap()`, `pass()`, `sendShortcut()`, `sendKeyState()`, `global()`, `event()`, `moveCursor()`, `moveCursorToCorner()`, `forceRendererReload()`, `forceIdle()`, `releaseInputCapture()`, `noop()`, `dispatch(name, arg)` |
| Helpers | `notify()`, `notifications()`, `dismissNotifications()`, `print()`, `log()`, `clearCrashedLockscreen()`, `execScheduledPropRefreshImmediately()`, `ok()`, `error(msg)` |

`exec(command)` is immediate and returns `SDispatchResult`. Dispatcher actions such as `killActive()` are deferred; utilities such as `notify`, `ok` and `error` run immediately. `spawn` and `spawnRaw` are not public config functions.

`dispatch(name, arg)` creates an action for any dispatcher by name, see [abyssctl](/wiki/abyssctl).

### Window selectors

Many window actions take an optional `selector` as their last parameter: `moveWindow`, `moveWindowOrGroup`, `swapWindow`, `swapNext`, `centerWindow`, `toggleGroup`, `changeGroupActive`, `moveIntoGroup`, `moveIntoOrCreateGroup`, `moveOutOfGroup` and `moveCursorToCorner` (new), next to the ones that already had one (`closeWindow`, `killWindow`, `signalWindow`, `toggleFloating`, `tag`, `clearTags`, `setGroupActive`, `fullscreen`, `moveToWorkspace`, ...).

```cpp
c.bind(SUPER, "K", moveWindow(Dir::Left, false, "class:^(kitty)$"));   // silent = false, then the selector
c.bind(SUPER, "J", centerWindow("title:^(Notes)$"));
c.bind(SUPER, "M", moveIntoGroup(Dir::Right, "tag:work"));
c.bind(SUPER, "F", fullscreen(0, "set", /*layoutAware=*/false, "initialclass:^(mpv)$"));
```

Without a selector (or with `""`) these stay plain dispatchers and act on the focused window. With one, the window is resolved when the action runs. A selector that matches no window is an error, never a silent fallback to the focused window.

| Selector | Matches |
|---|---|
| `class:<regex>` | the window class |
| `initialclass:<regex>` | the class at map time |
| `title:<regex>` | the window title |
| `initialtitle:<regex>` | the title at map time |
| `tag:<name>` | windows with this tag |
| `address:0x...` | exactly one window, as printed by `abyssctl` or `byAddress(w)` |
| `pid:<n>` | windows of this process |
| `floating` / `tiled` | all floating or all tiled windows |

The regex has to match the whole string, so use `class:^(kitty)$` or `class:.*kitty.*`, not a bare `class:kitty` for a partial match. A string without one of these prefixes matches nothing.

String literals convert to `bool` before they become a `std::string`. `moveWindow(Dir::Left, "class:x")` would therefore have silently meant `silent = true` with no selector. These overloads are deleted and are a compile error now: `moveWindow(Dir, const char*)`, `moveWindowToMonitor(std::string, const char*)`, `swapNext(const char*)`, `cycleNext(const char*)` and `cycleNext(bool, const char*)`, `moveToWorkspace(int / std::string, const char*)`, `changeGroupActive(const char*)` and `fullscreen(int, const char*, const char*)`. Pass the `bool` explicitly, as in the example above.

New actions in this area:

| Action | Effect |
|---|---|
| `clearTags(selector = "")` | removes every tag, including the dynamic ones from rules |
| `signalWindow(sig, selector = "")` | sends a signal to the process of the window |
| `setGroupActive(index, selector = "")` | makes the index-th window of the group active (1-based, `0` = last) |
| `changeWorkspaceId(from, to)` | gives workspace `from` the id `to`, fails if `to` exists |
| `noop()` | does nothing, a bind that only swallows the key |
| `fullscreen(mode, state, layoutAware, selector = "")` | `mode` 0 = fullscreen, 1 = maximize; `state` is `"toggle"`, `"set"` or `"unset"`; with `layoutAware = false` the layout is not told and the window just covers the monitor |

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

Rules return a `RuleHandle`, see [Handles](#handles).

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

Available events: `Start`, `Shutdown`, `WindowOpen`, `WindowOpenEarly`, `WindowClose`, `WindowDestroy`, `WindowKill`, `WindowActive`, `WindowUrgent`, `WindowTitle`, `WindowClass`, `WindowPin`, `WindowFullscreen`, `WindowFloating`, `WindowUpdateRules`, `WindowMoveToWorkspace`, `LayerOpened`, `LayerClosed`, `MonitorAdded`, `MonitorRemoved`, `MonitorFocused`, `MonitorLayoutChanged`, `WorkspaceActive`, `WorkspaceSpecialActive`, `WorkspaceCreated`, `WorkspaceRemoved`, `WorkspaceMoveToMonitor`, `ConfigReloaded`, `PropsRefreshed`, `Submap`, `ScreenshareState`, `KeyboardKey`.

`WindowDestroy` fires while the window object is being destroyed and hands you a `PHLWINDOWREF` that may already be expired, so check `w.lock()` before using it; use `WindowClose` when you need a still valid window. `WindowKill` fires just before click-to-kill (the `kill` dispatcher, then a click on a window) sends SIGKILL to the window's process.

```cpp
c.on(Event::WindowDestroy, [](PHLWINDOWREF w) {
    if (auto win = w.lock())
        notify(windowTitle(win));
});
c.on(Event::WindowKill, [](PHLWINDOW w) { notify("killing " + windowTitle(w)); });
```

`c.on` returns an `EventHandle`; `cancel()` removes the listener early and `isActive()` tells whether it still is registered (false after `cancel()` or once a reload dropped it).

## Notifications

`notify(text, ms = 5000, color = std::nullopt, icon = "info")` shows a notification. The second form takes options and is the one to use for the font size:

```cpp
auto n = notify("building...", NotifyOpts{.ms = 10000, .icon = "hint", .fontSize = 16.f});
```

`NotifyOpts` has `ms` (default 5000), `color` (optional, default is the icon's color), `icon` (`info`, `warning`, `error`, `hint`, `ok`, `confused` or `none`) and `fontSize` (default 13).

Both return a `NotificationHandle`. It is copyable and every call does nothing once the notification is gone, so a stale handle is safe to keep.

| Member | Effect |
|---|---|
| `alive()` | false once timed out or dismissed |
| `text()` / `setText(text)` | read or replace the text |
| `setColor(color)`, `setIcon(icon)`, `setFontSize(size)` | restyle it |
| `setTimeout(ms)` | restarts the countdown with a new duration |
| `pause()` / `resume()` | stop or restart the countdown, calls must be balanced |
| `dismiss()` | removes it |

```cpp
auto progress = std::make_shared<NotificationHandle>();
c.bind(SUPER, "P", [progress] {
    if (progress->alive())
        progress->setText("still running");
    else
        *progress = notify("started", NotifyOpts{.ms = 60'000});
    return ok();
});
```

`notifications()` lists every notification that is shown, oldest first. `dismissNotifications(n)` removes the oldest `n`, with no argument or `-1` it removes all.

## Logging and small helpers

- `abyss::print(fmt, args...)` and `abyss::log(LogLevel, text)` write `[abyss] text` into the compositor log (`abyssctl rollinglog`), like `print()` in the Lua API. `print` uses `std::format` syntax and the `Info` level; `log` takes `Trace`, `Debug`, `Info`, `Warn` or `Error`. Nothing is written while `debug:disable_logs` is on, which is the default, so set `c.debug.disable_logs = false` while you debug.
- `clearCrashedLockscreen()` unlocks the session when the lock screen client crashed and the session stays locked behind a red screen. It refuses while a working lock screen holds the lock. It runs immediately and is deliberately not a dispatcher, so other local processes cannot call it.
- `execScheduledPropRefreshImmediately()` applies window rule and property changes from `set()` now instead of on the next loop iteration. It returns `false` if nothing was pending.

```cpp
c.debug.disable_logs = false;
c.bind(SUPER, "L", [] { print("active gaps: {}", get<int>("general:gaps_in").value_or(-1)); return ok(); });
```

## Reading state

Everything you could read in Hyprland's Lua API is available as plain functions (`abyss/query.hpp`). Missing objects come back as null handles, empty lists or `std::nullopt`; every accessor accepts a null handle.

```cpp
c.bind(SUPER, "I", [] {
    for (auto& w : windows({.workspace = activeWorkspace(monitorAtCursor()), .floating = true}))
        notify(windowTitle(w));
});
```

| Area | Functions |
|---|---|
| Lookups | `windows(WindowFilter)`, `urgentWindow()`, `lastWindow()`, `workspaceWindows(ws)`, `workspaceBySelector("name:web")`, `activeWorkspace(monitor)`, `activeSpecialWorkspace()`, `lastWorkspace()`, `monitorBySelector("DP-1")`, `monitorAt(x, y)`, `monitorAtCursor()`, `layers(LayerFilter)`, `cursorPos()`, `currentSubmap()`, `isKeyDown("Shift_L")`, `version()` |
| Window | `windowMapped`, `windowHidden`, `windowVisible`, `windowAt`, `windowSize`, `windowWorkspace`, `windowMonitor`, `windowInitialTitle`, `windowXwayland`, `windowPinned`, `windowFullscreenModes`, `windowGroup`, `windowTags`, `windowSwallowing`, `windowInhibitingIdle`, `windowXdgTag`, `windowContentType`, `windowStableId`, `windowActive`, `windowLayoutInfo`, … |
| Monitor | `monitorId`, `monitorDescription`, `monitorSize`, `monitorRefreshRate`, `monitorPosition`, `monitorScale`, `monitorTransform`, `monitorActiveWorkspace`, `monitorDpms`, `monitorVrrActive`, `monitorAvailableModes`, `monitorReserved`, `monitorSetWorkspace(m, "3")`, `monitorSetSpecialWorkspace(m, "special:term")`, … |
| Workspace | `workspaceMonitor`, `workspaceVisible`, `workspaceSpecial`, `workspaceHasUrgent`, `workspaceHasFullscreen`, `workspaceEmpty`, `workspaceTiledLayout`, `workspaceLastWindow`, `workspaceGroups`, … |
| Layer | `layerNamespace`, `layerGeometry`, `layerPid`, `layerMonitor`, `layerLayer`, `layerInteractivity`, … |
| Group | `groupSize`, `groupCurrent`, `groupCurrentIndex` (1-based), `groupMembers`, `groupLocked`, `groupAdd(g, w)`, `groupRemove(g, w)` |

The full list with comments is in `include/abyss/query.hpp`.

## Timers

```cpp
auto t = c.timer(60'000, [] { notify("one minute passed"); }, /*repeat=*/true);
t.setEnabled(false);   // disarm
t.cancel();            // remove
```

The full signature is `c.timer(ms, fn, repeat = false, enabled = true)`. With `enabled = false` the timer is created disarmed and starts when you call `setEnabled(true)` or `setTimeout(ms)`. See [Handles](#handles).

## Handles

Binds, rules, timers and event listeners return a small handle you can keep (in a variable, a lambda capture or a global) to switch the thing later without a reload. A reload drops every bind, rule, timer and listener, which invalidates all handles: query them again from the new `configure()`.

| Handle | From | Methods |
|---|---|---|
| `BindHandle` | `c.bind` | `setEnabled(bool)`, `isEnabled()`, `remove()` |
| `RuleHandle` | `c.windowRule`, `c.layerRule`, `c.workspaceRule` | `setEnabled(bool)`, `isEnabled()` |
| `TimerHandle` | `c.timer` | `setEnabled(bool)`, `isEnabled()`, `setTimeout(int ms)`, `cancel()` |
| `EventHandle` | `c.on` | `isActive()`, `cancel()` |

### Binds

A disabled bind stays registered but does not trigger. `remove()` unregisters exactly this bind (other binds on the same key stay) and the handle is dead afterwards. `isEnabled()` is false for a removed or reload-dropped bind.

```cpp
auto b = c.bind(SUPER, "G", [] { return exec("steam"); });
b.setEnabled(false);
bool on = b.isEnabled();   // false
b.remove();
```

### Rules

`c.windowRule`, `c.layerRule` and `c.workspaceRule` return a `RuleHandle`. A disabled rule stays registered but is ignored. Switching it applies the rules to the existing windows, layers or workspaces right away.

- `windowRule(match, effects, name = "", enabled = true)` and `layerRule(match, effects, name = "", enabled = true)`: a call with a name that was already used in this config load updates that rule (match properties and effects are added, `enabled` is set again). Without a name every call adds a new rule. `enabled = false` registers or switches the rule disabled.
- `workspaceRule(workspace, rule)`: set `rule.enabled = false` (`WorkspaceRule::enabled`) to register it disabled. An enabled rule for the same workspace string is merged into instead of adding a new one, so a disabled rule is only created when none is enabled yet; otherwise switch it with the handle.

```cpp
auto r = c.windowRule({.cls = "^(kitty)$"}, {.opacity = "0.8"}, "kitty-opacity", /*enabled=*/false);

c.bind(SUPER, "O", [r] { r.setEnabled(!r.isEnabled()); });
```

`isEnabled()` is false when the rule is gone (after a reload).

### Timers

`TimerHandle::isEnabled()` means armed: a oneshot timer is not enabled any more after it fired, and a cancelled or reload-dropped timer never is. `setEnabled(true)` re-arms with the original interval, `setTimeout(ms)` sets a new interval (at least 1 ms) and arms the timer to fire after it; for a repeating timer the new interval is kept.

```cpp
auto t = c.timer(5000, [] { notify("still here"); }, /*repeat=*/true, /*enabled=*/false);
t.setTimeout(30'000);      // armed now, fires every 30 s
```

### Events

`EventHandle::isActive()` is false after `cancel()` or once a reload dropped the listener, also on copies of the handle.

```cpp
auto h = c.on(Event::WindowOpen, [](PHLWINDOW w) { notify(windowTitle(w)); });
// later
if (h.isActive())
    h.cancel();
```

## Idle

`c.idle` runs a callback after a period without input (keyboard, pointer, touch, tablet) and, optionally, another one on the first input afterwards:

```cpp
c.idle(300, [] { exec("abysslock"); });                    // lock after 5 minutes
c.idle(600, [] { exec("abyssctl dispatch dpms off"); },
            [] { exec("abyssctl dispatch dpms on"); });    // screen off, back on with the first input
```

- Nothing is registered by default: no built-in lock, suspend or helper.
- Active idle inhibitors (video players, browsers in fullscreen, idle-inhibit window rules) block the rule. When they go away, the whole timeout counts again.
- `onResume` only runs after `onIdle` ran. It may be left out.
- `seconds` must be finite, at least `0.001` and at most about `4294966`.
- A reload drops and re-creates the rules, they never stack up.
- Callbacks run on the compositor thread: keep them short and use `exec()` for anything slow. Exceptions are caught and logged.
- The `forceIdle` action bypasses inhibitors, as it does for the `ext-idle-notify` protocol.

## Screensaver

The screensaver is part of idle. Pass `screensaver(...)` instead of the two callbacks:

```cpp
c.idle(300, screensaver(saver::clock()));                                   // big clock with date
c.idle(300, screensaver(saver::cava({.bars = 64, .color = 0x80b0d8})));    // audio bars
```

Everything else you draw yourself. The callback runs once per monitor and frame while the screensaver is shown:

```cpp
c.idle(300, screensaver([](Canvas& g, const SaverFrame& f) {
    g.clear(0x050816);
    g.text(f.width / 2, f.height / 2, "刀", {.size = 200, .color = 0xc8a25a, .align = Align::Center});
    for (size_t i = 0; i < f.audio.size(); ++i)                               // only with .audio = true
        g.rect(i * f.width / f.audio.size(), f.height, 8, -f.audio[i] * 200, 0x80b0d8);
}, {.fps = 30, .audio = true, .bars = 64}));
```

- `Canvas`: `clear`, `rect`, `roundRect`, `line`, `circle`, `text`, `textSize`, `image` (PNG) and `cairo()` for raw cairo. Colors are `0xRRGGBB` with an optional alpha, coordinates are logical pixels.
- `SaverFrame`: `width`, `height`, `scale`, `monitor`, `time` (seconds since it started), `now`, `audio` (bars from 0 to 1).
- Options: `fps` (0 draws once, at most 144), `audio` (starts `cava` while the screensaver runs), `bars`, `monitors` (only these outputs, the others stay black).
- `saver::clock()` and `saver::cava()` are plain functions in `abyss/saver.hpp` built from the same API, so they double as examples.
- The first key or click only wakes up and never reaches a window, a bind or the lock screen.
- The saver is drawn above everything, including the lock screen. The cursor is hidden while it runs.
- `saver::clock()` jumps to a new spot once a minute against burn-in (`.wander = false` keeps it centered).
- Without `cava` installed, or without audio, the bars stay quiet. The compositor keeps running if `cava` dies.
- `abyssctl dispatch screensaver` shows it right away, for testing.

Lock or screen off are separate rules. Lock at the same time as the saver and the lock screen is what you see after waking it up:

```cpp
c.idle(300, screensaver(saver::clock()));
c.idle(300, [] { exec("abysslock"); });
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

Touchpad gestures run a built-in action or your own code:

```cpp
c.gesture(3, "horizontal", "workspace");                     // built-in action
c.gesture(3, "down", [] { exec("grim"); });                  // runs once when the fingers are lifted
c.gesture(4, "swipe", GestureCallbacks{
    .start  = [](const GestureEvent& e) { /* direction recognised */ },
    .update = [](const GestureEvent& e) { /* e.delta, e.scale, e.rotation */ },
    .finish = [](const GestureEvent& e) { /* e.cancelled */ },
});
```

- Built-in actions: `workspace`, `move`, `resize`, `special`, `close`, `float`, `fullscreen`, `cursor_zoom`, `scroll_move`, `unset`.
- Directions: `horizontal`, `vertical`, `left`, `right`, `up`, `down`, `swipe`, `pinch`, `pinchin`, `pinchout`.
- `fingers` must be between 2 and 9. Options (`mods`, `scale`, `disableInhibit`) work for all three forms.

Per-device settings, including tags for keyboards and pointers:

```cpp
c.device("my-touchpad", {.naturalScroll = true, .tags = "touchpad"});
```

`c.permission(binary, type, mode)` sets rules such as `c.permission("/usr/bin/grim", "screencopy", "allow")` (types: `screencopy`, `cursorpos`, `keyboard`, `input-capture`; modes: `allow`, `ask`, `deny`).

::: info Touch and tablets
All touchscreen, touchpad and tablet options are available (`c.input.touchpad`, `c.input.touchdevice`, `c.input.tablet`). They have not been tested on real touch or tablet hardware yet.
:::

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
