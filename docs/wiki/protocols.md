# Choosing Wayland protocols

By default Abyssland offers every protocol it implements. Nothing needs to be configured, and that is the supported setup.

::: warning You own the consequences
You can switch protocols off, but if you do, anything that breaks because of it is a user issue, not a platform issue. A client that needs a protocol which is not offered either falls back to something worse or refuses to start. Abyss does not test or support reduced protocol sets.
:::

## Configuring

Names are the Wayland interface names, for example `zwlr_screencopy_manager_v1`. Everything below goes into `configure()` in your `config.hpp`.

```cpp
// switch one off
c.protocol("zwlr_screencopy_manager_v1", false);

// switch several off
c.protocols.disable({"abyssland_toplevel_export_manager_v1", "ext_image_copy_capture_manager_v1"});

// keep only the listed ones, everything else that is not core goes off
c.protocols.enableOnly({"zwlr_layer_shell_v1", "zwlr_screencopy_manager_v1"});

// offer at most version 2 of a protocol
c.protocol("zwlr_screencopy_manager_v1", {.maxVersion = 2});
```

`c.protocols.enable({...})` switches names on again. Explicit `disable`, `enable` and `protocol` calls that come after `enableOnly` still apply.

### Typical setups

Only wlr screencopy (grim, wf-recorder, xdg-desktop-portal-wlr), see [Desktop portals](/wiki/portals):

```cpp
c.protocols.disable({"abyssland_toplevel_export_manager_v1", "ext_image_copy_capture_manager_v1",
                     "ext_output_image_capture_source_manager_v1",
                     "ext_foreign_toplevel_image_capture_source_manager_v1"});
```

Only Abyss' own screen capture (xdg-desktop-portal-abyss):

```cpp
c.protocols.disable({"zwlr_screencopy_manager_v1"});
```

ext-image-copy-capture only: disable `zwlr_screencopy_manager_v1` and `abyssland_toplevel_export_manager_v1`.

## Rules of the game

- **Core protocols cannot be switched off.** `wl_seat`, `wl_compositor`, `wl_subcompositor`, `wl_shm`, `wl_data_device_manager`, `wl_output`, `xdg_wm_base` and `zwp_linux_dmabuf_v1` are required, nothing could start without them. Trying to disable one is a config error.
- **Unknown names are a config error**, so a typo never goes unnoticed. The error points you to `abyssctl protocols`.
- **Changes need a restart.** Protocols are created when the compositor starts, before the first client can connect. A reload without restart (`abyssctl reload --no-build`) cannot add or remove globals safely: if the config asks for something different than what is running, a notification says it takes effect after a restart and nothing changes until then.
- **Conditional protocols** (drm lease, syncobj, commit timing, color management, `wl_drm`, ...) stay absent when the hardware or the config does not support them, regardless of this setting.

::: warning NVIDIA
Keep `wp_linux_drm_syncobj_manager_v1` (explicit sync) switched on. Turning it off on NVIDIA causes flickering and tearing artefacts. See [NVIDIA](/wiki/nvidia).
:::

## Checking the state

```sh
abyssctl protocols
abyssctl -j protocols
```

Each protocol is listed with its state: `on`, `off`, or `not-created` (enabled, but this session has no such global because it depends on the hardware or config). Core protocols are marked `core`, a version limit is shown as `maxVersion=N`. The JSON output has the fields `name`, `core`, `state`, `maxVersion`, `version` (built-in version) and `note`.

## All protocols

### Core (cannot be disabled)

| Protocol | Notes |
|---|---|
| `wl_seat` | input |
| `wl_data_device_manager` | clipboard / drag and drop |
| `wl_compositor` | surfaces |
| `wl_subcompositor` | subsurfaces |
| `wl_shm` | shared memory buffers |
| `wl_output` | monitors |
| `xdg_wm_base` | application windows |
| `zwp_linux_dmabuf_v1` | GPU buffers (Vulkan / GL clients); only offered when the renderer supports it |

### Abyssland's own

| Protocol | Notes |
|---|---|
| `abyssland_lock_notifier_v1` | lock screen notification for helpers |
| `abyssland_focus_grab_manager_v1` | popup focus grabs (launchers, menus) |
| `abyssland_toplevel_mapping_manager_v1` | maps toplevel handles to windows (portal) |
| `abyssland_global_shortcuts_manager_v1` | global shortcuts (xdg-desktop-portal-abyss) |
| `abyssland_ctm_control_manager_v1` | colour transform matrix (night light helpers) |
| `abyssland_input_capture_manager_v1` | input capture (portal, remote desktop) |
| `abyssland_surface_manager_v1` | Abyssland surface extensions (client side effects / rules) |
| `abyssland_toplevel_export_manager_v1` | own screencopy for windows (xdg-desktop-portal-abyss needs it) |

### wlr

| Protocol | Notes |
|---|---|
| `zwlr_gamma_control_manager_v1` | night light (gammastep, wlsunset) |
| `zwlr_foreign_toplevel_manager_v1` | taskbars (waybar wlr/taskbar, ...) |
| `zwlr_output_power_manager_v1` | DPMS from clients (wlopm, idle helpers) |
| `zwlr_virtual_pointer_manager_v1` | virtual pointer (wlrctl, remote desktop) |
| `zwlr_output_manager_v1` | monitor configuration (kanshi, wlr-randr, nwg-displays) |
| `zwlr_layer_shell_v1` | bars, launchers, wallpapers, notification daemons: most shells stop working without it |
| `zwlr_data_control_manager_v1` | clipboard managers (wl-clipboard --watch, cliphist) |
| `zwlr_screencopy_manager_v1` | grim, wf-recorder, xdg-desktop-portal-wlr |

### ext

| Protocol | Notes |
|---|---|
| `ext_foreign_toplevel_list_v1` | toplevel list (needed by ext-image-copy-capture window capture) |
| `ext_idle_notifier_v1` | idle detection of clients (hypridle, swayidle) |
| `ext_session_lock_manager_v1` | lock screens (hyprlock, swaylock, abysslock): without it nothing can lock the session |
| `ext_workspace_manager_v1` | workspace info for bars |
| `ext_data_control_manager_v1` | clipboard managers (ext variant) |
| `ext_background_effect_manager_v1` | blur for client surfaces |
| `ext_output_image_capture_source_manager_v1` | monitor capture sources (ext-image-copy-capture) |
| `ext_foreign_toplevel_image_capture_source_manager_v1` | window capture sources (ext-image-copy-capture) |
| `ext_image_copy_capture_manager_v1` | ext screencopy (portals, OBS, recent screenshot tools) |

### wp

| Protocol | Notes |
|---|---|
| `wp_viewporter` | surface scaling / cropping (video players, browsers) |
| `wp_tearing_control_manager_v1` | tearing for games (allow_tearing) |
| `wp_fractional_scale_manager_v1` | fractional scaling: clients render blurry without it |
| `wp_cursor_shape_manager_v1` | cursor shapes |
| `wp_alpha_modifier_v1` | surface opacity |
| `wp_presentation` | presentation timing (video players, games) |
| `wp_single_pixel_buffer_manager_v1` | solid colour buffers |
| `wp_security_context_manager_v1` | sandboxed clients (Flatpak) |
| `wp_content_type_manager_v1` | content type hints (game / video) |
| `wp_pointer_warp_v1` | clients warping the pointer |
| `wp_fifo_manager_v1` | frame pacing |
| `wp_commit_timing_manager_v1` | commit timing (render:commit_timing_enabled) |
| `wp_color_manager_v1` | colour management / HDR (render:cm_enabled) |
| `wp_drm_lease_device_v1` | VR headset leases; only offered with a DRM backend |
| `wp_linux_drm_syncobj_manager_v1` | explicit sync. NVIDIA: keep it ON, turning it off causes flickering |

### xdg

| Protocol | Notes |
|---|---|
| `xdg_activation_v1` | focus requests / startup notification |
| `xdg_wm_dialog_v1` | dialog hints |
| `xdg_toplevel_tag_manager_v1` | window tags |
| `xdg_system_bell_v1` | system bell |
| `zxdg_output_manager_v1` | output geometry for clients: many toolkits and bars need it |
| `zxdg_decoration_manager_v1` | server side decorations |
| `zxdg_exporter_v2` | window parenting across clients |
| `zxdg_importer_v2` | window parenting across clients |

### zwp and misc

| Protocol | Notes |
|---|---|
| `zwp_idle_inhibit_manager_v1` | video players keeping the screen on |
| `zwp_relative_pointer_manager_v1` | games, 3D applications |
| `zwp_pointer_gestures_v1` | touchpad gestures in clients |
| `zwp_keyboard_shortcuts_inhibit_manager_v1` | VMs / remote desktops grabbing shortcuts |
| `zwp_text_input_manager_v1` | input methods (legacy) |
| `zwp_text_input_manager_v3` | input methods |
| `zwp_pointer_constraints_v1` | games (pointer lock / confine) |
| `zwp_input_method_manager_v2` | input methods (fcitx5, ibus) |
| `zwp_virtual_keyboard_manager_v1` | virtual keyboards (wtype, on-screen keyboards) |
| `zwp_tablet_manager_v2` | drawing tablets |
| `zwp_primary_selection_device_manager_v1` | middle-click paste |
| `org_kde_kwin_server_decoration_manager` | KDE style server side decorations |
| `xwayland_shell_v1` | Xwayland: X11 windows cannot be mapped without it |
| `vicinae_hotkey_manager_v1` | vicinae launcher hotkeys |
| `wl_drm` | legacy Mesa buffer sharing; only offered together with dmabuf |
