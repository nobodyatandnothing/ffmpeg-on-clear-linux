## nv-requirements-and-preparation

This guide is for folks using the NVIDIA proprietary driver.

* [Requirements](#requirements)
* [CUDA preparation](#cuda-preparation)
* [Enable Service File](#enable-service)
* [Using the NVIDIA NVDEC-enabled VA-API driver](#nvdec-driver)

### <a id="requirements">Requirements

Although testing was done using a NVIDIA GPU, the Intel(R) Media SDK is included during the build process. For NVIDIA graphics, this requires the proprietary driver to be installed under `/opt/nvidia`. Optionally install CUDA for extra hardware acceleration capabilities. See installation guides [NVIDIA Drivers](https://docs.01.org/clearlinux/latest/tutorials/nvidia.html) and [NVIDIA CUDA Toolkit](https://docs.01.org/clearlinux/latest/tutorials/nvidia-cuda.html) at Clear Linux.

Set your GPU's [compute capability](https://en.wikipedia.org/wiki/CUDA) in `localenv`. The file resides at the top-level and is ignored by Git. For example, the GeForce GTX 1660 model supports max `7.5` compute capability. Omit this step is using a non-NVIDIA GPU.

```text
cudaarch="compute_75"  # Turing
cudacode="sm_75"
```

Optionally enable `ForceCompositionPipeline` for a better desktop experience, especially when moving-resizing a terminal window while playing a video. This can be done at the device level by adding-or-editing a file `/etc/X11/xorg.conf.d/nvidia-device.conf`. Replace `MODEL_STRING` with your actual GPU model (i.e. GTX 1660). Finally reboot for the change to take effect.

```text
Section "Device"
    Identifier    "Device0"
    Driver        "nvidia"
    VendorName    "NVIDIA Corporation"
    BoardName     "GeForce MODEL_STRING"
    Option        "ForceCompositionPipeline" "On"
    Option        "ForceFullCompositionPipeline" "On"
EndSection
```

### <a id="cuda-preparation">CUDA preparation

Until CUDA reaches full compatibility with GCC 11.x, install the `c-extras-gcc10` bundle. This applies if `gcc --version` returns 11 or later. A flag will be passed to `nvcc` to use `gcc-10`.

```bash
$ sudo swupd bundle-add c-extras-gcc10
```

### <a id="enable-service">Enable service file

The `swupd` tool is not yet mindful of the NVIDIA proprietary installation. Create a systemd service unit to overwrite the Clear Linux OS provided libGL files. The service accommodates the NVIDIA 64-bit libs residing in `/opt/nvidia/lib64` or `/opt/nvidia/lib`.

Running `swupd bundle-add devpkg-libva` or `devpkg-mediasdk` or `devpkg-mesa` restores the Clear Linux OS provided libGL files which breaks the NVIDIA installation. The service removes libGL files that shouldn't be there i.e. `libEGL.so*`, `libGLESv1_CM.so*`, `libGLESv2.so*`, and `libGL.so*`.

```bash
$ sudo tee /etc/systemd/system/fix-nvidia-libGL-trigger.service >/dev/null <<'EOF'
[Unit]
Description=Fixes libGL symlinks for the NVIDIA proprietary driver
BindsTo=update-triggers.target

[Service]
Type=oneshot
ExecStart=/usr/bin/sh -c '[ -f /opt/nvidia/lib64/libGL.so.1 ] && lib=lib64 || lib=lib; /usr/bin/ln -sfv /opt/nvidia/$lib/libGL.so.1 /usr/lib/libGL.so.1'
ExecStart=/usr/bin/sh -c '/usr/bin/rm -fv /usr/lib64/libEGL.so* /usr/lib32/libEGL.so*'
ExecStart=/usr/bin/sh -c '/usr/bin/rm -fv /usr/lib64/libGLESv1_CM.so* /usr/lib32/libGLESv1_CM.so*'
ExecStart=/usr/bin/sh -c '/usr/bin/rm -fv /usr/lib64/libGLESv2.so* /usr/lib32/libGLESv2.so*'
ExecStart=/usr/bin/sh -c '/usr/bin/rm -fv /usr/lib64/libGL.so* /usr/lib32/libGL.so*'
EOF
```

Reload the systemd manager configuration to pickup the new serivce.

```bash
$ sudo systemctl daemon-reload
```

Add the service as a dependency to the Clear Linux OS updates trigger causing the service to run after every `swupd bundle-add` and `swupd update`.

```bash
$ sudo systemctl add-wants update-triggers.target fix-nvidia-libGL-trigger.service
```

Run the service manually and subsequently get the status about the service.

```bash
$ sudo systemctl start fix-nvidia-libGL-trigger.service

$ systemctl status fix-nvidia-libGL-trigger.service
$ journalctl -xeu fix-nvidia-libGL-trigger.service
```

### <a id="nvdec-driver">Using the NVIDIA NVDEC-enabled VA-API driver

The NVDEC-enabled VA-API driver (see the official [Github](https://github.com/elFarto/nvidia-vaapi-driver)) is compiled automatically if your NVIDIA display driver is greater than 470.57, but it is not enabled by default as it is still experimental. The implementation is specifically designed to be used by Firefox. To use it, change `LIBVA_DRIVERS_PATH` mentioned in the Firefox configuration to point to `/usr/local/lib/dri` first.

```bash
LIBVA_DRIVERS_PATH=/usr/local/lib/dri:/usr/lib64/dri
```

Note that this requires enabling modeset for the `nvidia-drm` module, or else it won't load. Reboot for the change to take effect.

```bash
$ sudo mkdir -p /etc/modprobe.d

$ sudo tee /etc/modprobe.d/enable-nvidia-modeset.conf >/dev/null <<'EOF'
options nvidia-drm modeset=1
EOF
```

Below see capture output using the NVDEC-enabled driver.

```bash
$ LIBVA_DRIVERS_PATH=/usr/local/lib/dri LIBVA_DRIVER_NAME=nvidia vainfo

libva info: VA-API version 1.11.0
libva info: User environment variable requested driver 'nvidia'
libva info: Trying to open /usr/local/lib/dri/nvidia_drv_video.so
libva info: Found init function __vaDriverInit_1_0
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.11 (libva 2.11.0)
vainfo: Driver version: VA-API NVDEC driver
vainfo: Supported profile and entrypoints
      VAProfileMPEG2Simple            : VAEntrypointVLD
      VAProfileMPEG2Main              : VAEntrypointVLD
      VAProfileVC1Simple              : VAEntrypointVLD
      VAProfileVC1Main                : VAEntrypointVLD
      VAProfileVC1Advanced            : VAEntrypointVLD
      <unknown profile>               : VAEntrypointVLD
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointVLD
      VAProfileH264ConstrainedBaseline: VAEntrypointVLD
      VAProfileHEVCMain               : VAEntrypointVLD
      VAProfileVP8Version0_3          : VAEntrypointVLD
      VAProfileVP9Profile0            : VAEntrypointVLD
```

