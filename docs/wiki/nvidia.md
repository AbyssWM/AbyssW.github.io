# NVIDIA

NVIDIA support is a priority for Abyss. Abyssland keeps all of Hyprland's NVIDIA handling unchanged: driver detection, explicit sync, the atomic commit workarounds, multi-GPU blitting and the cursor fallbacks.

::: warning Not yet tested on real hardware
The code paths are identical to upstream Hyprland 0.56.2, but Abyss itself has not run on an NVIDIA card yet. Reports are welcome.
:::

## Supported cards

| Generation | Driver | Notes |
|---|---|---|
| RTX 50 (Blackwell) | 595, **open kernel modules** | NVIDIA ships Blackwell support only in the open modules. Some distributions (Void, for example) only package the proprietary ones, so you may have to install the open modules yourself. |
| RTX 40, 30, 20, GTX 16 | 595 (proprietary or open modules) | |
| GTX 10 and older | older driver branches | Not targeted. |

Use the proprietary NVIDIA driver, not nouveau.

## Setup

1. Install the driver. On Void: `nvidia` (with `nvidia-dkms`, which pulls in `nvidia-libs` with EGL-Wayland and GBM), plus `nvidia-libs-32bit` for Steam and `nvidia-vaapi-driver` for video decoding.
2. Make sure kernel modesetting is on. After a reboot, `cat /sys/module/nvidia_drm/parameters/modeset` must print `Y`. If it does not, add `nvidia-drm.modeset=1` to the kernel command line.
3. Optional environment for applications, in your config:

```cpp
c.env("LIBVA_DRIVER_NAME", "nvidia");          // hardware video decoding (nvidia-vaapi-driver)
c.env("NVD_BACKEND", "direct");
c.env("__GLX_VENDOR_LIBRARY_NAME", "nvidia");  // only on systems with an NVIDIA card driving the session
```

## Options that matter on NVIDIA

All of them default to automatic detection and normally need no change.

| Option | Default | Meaning |
|---|---|---|
| `cursor:no_hardware_cursors` | `2` (auto) | Software cursor on NVIDIA with multiple GPUs or VRR. Set `0` to force hardware cursors. |
| `cursor:use_cpu_buffer` | `2` (auto) | CPU buffer for the cursor on NVIDIA. |
| `opengl:nvidia_anti_flicker` | `true` | Waits for rendering to finish when explicit sync is not used. |
| `render:direct_scanout` | `0` | Leave off if you see flicker in fullscreen games. |

Explicit sync (`linux-drm-syncobj-v1`) is enabled automatically when the driver supports it. Do not switch it off via [protocol selection](/wiki/protocols): on NVIDIA that causes flickering.

## Two GPUs (for example AMD for the monitors, NVIDIA for CUDA or games)

Pin Abyssland to the card that drives the monitors, so the NVIDIA card is only used for compute or PRIME offload:

```cpp
#include <filesystem>
// a udev rule creates /dev/dri/amd-gpu, see below
if (std::filesystem::exists("/dev/dri/amd-gpu"))
    c.env("ABYSSAL_DRM_DEVICES", "/dev/dri/amd-gpu");
```

```
# /etc/udev/rules.d/61-amd-gpu-symlink.rules
SUBSYSTEM=="drm", KERNEL=="card*", ATTRS{vendor}=="0x1002", SYMLINK+="dri/amd-gpu"
```

The same, typed, with `c.gpu` (no udev rule needed; devices that do not exist on a machine are skipped, so the config also works elsewhere):

```cpp
// render only on the AMD card, the NVIDIA card stays free for CUDA
c.gpu.devices = {"/dev/dri/by-path/pci-0000:0b:00.0-card"};
```

`c.gpu.noModifiers`, `noExplicitMgpu`, `noAtomic` and `eglNoModifiers` set the other variables of the table below. They are read at start; a reload only warns. See [Patching and replacing](/wiki/patching#gpu-options-c-gpu).

`ABYSSAL_DRM_DEVICES` takes a colon-separated list; the first device is the primary one. Paths are resolved, so `/dev/dri/by-path/...` links work too. Start games on the NVIDIA card with PRIME offload (`prime-run %command%` in Steam).

## Environment variables (Hyprland → Abyss)

| Hyprland / aquamarine | Abyss |
|---|---|
| `AQ_DRM_DEVICES` | `ABYSSAL_DRM_DEVICES` |
| `AQ_NO_ATOMIC` | `ABYSSAL_NO_ATOMIC` |
| `AQ_NO_MODIFIERS` | `ABYSSAL_NO_MODIFIERS` |
| `AQ_MGPU_NO_EXPLICIT` | `ABYSSAL_MGPU_NO_EXPLICIT` |
| `AQ_NO_KMS_REQUIREMENT` | `ABYSSAL_NO_KMS_REQUIREMENT` |
| `AQ_TRACE` | `ABYSSAL_TRACE` |
| `HYPRLAND_EGL_NO_MODIFIERS` | `ABYSSLAND_EGL_NO_MODIFIERS` |

Variables that belong to the NVIDIA driver or to applications (`LIBVA_DRIVER_NAME`, `__GLX_VENDOR_LIBRARY_NAME`, `GBM_BACKEND`, `__GL_*`) keep their names. Set them with `c.env(...)`; the `ABYSSAL_*` ones also have a typed form, `c.gpu`. The config runs before the GPU backend starts.
