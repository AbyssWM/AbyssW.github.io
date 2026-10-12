# Tools

Small companion tools forked from hyprwm projects. Both install to `/opt/abyss` with the rest of Abyss (`sh tools/build.sh AbyssDusk`, `sh tools/build.sh AbyssPicker`).

## AbyssDusk

Blue light filter, forked from [hyprsunset](https://github.com/hyprwm/hyprsunset). It sets a color transform matrix through the Abyss protocol `abyssland_ctm_control_manager_v1`, so it only works on Abyssland.

```sh
abyssdusk -t 4000        # fixed temperature in K
abyssdusk -g 90          # gamma in percent
abyssdusk -i             # identity (no color change)
abyssdusk                # use the profiles from the config
```

Config file: `~/.config/abyss/abyssdusk.conf` (Abyssparse format):

```
max-gamma = 100

profile {
    time = 7:30
    identity = true
}

profile {
    time = 20:30
    temperature = 3500
    gamma = 0.8
}
```

### IPC

A running `abyssdusk` listens on a socket at `$XDG_RUNTIME_DIR/abyss/$ABYSSLAND_INSTANCE_SIGNATURE/.abyssdusk.sock`. It accepts the commands `temperature`, `gamma` and `identity`, so scripts can change the filter at runtime (temperature range 1000 to 20000 K, gamma 0 to `max-gamma` percent).

## AbyssPicker

Color picker, forked from [hyprpicker](https://github.com/hyprwm/hyprpicker). Launch it, click a pixel, done. It freezes the displays while you pick. Options: `abysspicker --help`.

::: warning Needs screencopy
AbyssPicker needs the Wayland protocols `zwlr_screencopy_manager_v1` and `zwlr_layer_shell_v1`. If you disable screencopy in your config with `c.protocol("zwlr_screencopy_manager_v1", false)` (or `c.protocols.enableOnly` without it), the picker stops working. That is intentional, see [Wayland protocols](/wiki/protocols).
:::
