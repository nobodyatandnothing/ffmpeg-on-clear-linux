## firefox-config-file-and-settings

* [Firefox config file](#config)
* [Review Firefox settings](#settings)
* [See also](#wikis)

### <a id="config">Firefox config file

The following is my Firefox config. Update the value for `LIBVA_DRIVER_NAME` or leave it `auto`. Subsequently, the driver name is overridden automatically for NVIDIA hardware.

```bash
$ cat ~/.config/firefox.conf

export FONTCONFIG_PATH=/usr/share/defaults/fonts
export LD_LIBRARY_PATH=/usr/local/lib

export LIBVA_DRIVERS_PATH=/usr/lib64/dri:/usr/local/lib/dri
export LIBVA_DRIVER_NAME=auto

if [[ -d /opt/nvidia ]]
then
    # vdpau-va-driver-v9 (default VA-API driver)
    nv_va_path1=/usr/lib64/dri/nvidia_drv_video.so

    # nvidia-vaapi-driver (decodes using the NVDEC engine on the GPU)
    # change LIBVA_DRIVERS_PATH above to /usr/local/lib/dri:/usr/lib64/dri
    nv_va_path2=/usr/local/lib/dri/nvidia_drv_video.so

    if [[ -f $nv_va_path1 || -f $nv_va_path2 ]]
    then
        # add /opt/nvidia/{lib64,lib} to path
        export LD_LIBRARY_PATH="/opt/nvidia/lib64:/opt/nvidia/lib:$LD_LIBRARY_PATH"

        # libva doesn't yet know which driver to load for the nvidia-drm driver
        # this forces libva to load the nvidia backend
        export LIBVA_DRIVER_NAME=nvidia
    fi
fi

if [[ $XDG_SESSION_TYPE == wayland ]]
then
    export MOZ_ENABLE_WAYLAND=1
else
    export MOZ_DISABLE_WAYLAND=1
    export MOZ_X11_EGL=1
fi

export MOZ_ACCELERATED=1
export MOZ_DISABLE_RDD_SANDBOX=1
export MOZ_USE_XINPUT2=1
export MOZ_WEBRENDER=1
```

### <a id="settings">Review Firefox settings

Below are the minimum settings applied via `about:config` to enable hardware acceleration. The `media.rdd-ffmpeg.enable` flag must be enabled for h264ify to work along with VP9. Basically, this allows you to choose to play videos via the h264ify extension or VP9 media by disabling h264ify and enjoy beyond 1080P playback.

```text
gfx.canvas.azure.accelerated                   true
gfx.webrender.all                              true
gfx.webrender.enabled                          true

Enable software render if you want to render on the CPU instead of GPU.
This is helpful for NVIDIA graphics if you prefer the desktop to remain
fluid while watching a video, which the GPU is handling along with
desktop composition. For Intel graphics, leave this setting false
since webrender on the GPU is needed to decode videos in hardware.
gfx.webrender.software                         false

Do not add xrender if missing or set to false or click on the trash icon.
This is a legacy setting that shouldn't be used as it disables WebRender.
gfx.xrender.enabled                            false

Ensure false so to be on a supported code path for using WebRender.
layers.acceleration.force-enabled              false

media.ffmpeg.dmabuf-textures.enabled           true
media.ffmpeg.vaapi-drm-display.enabled         true
media.ffmpeg.vaapi.enabled                     true
media.ffvpx.enabled                            false

Enable to help get decoding to work for NVIDIA 470 driver series.
widget.dmabuf.force-enabled                    true

Verify enabled, necessary for the NVIDIA-NVDEC enabled driver to work.
media.rdd-process.enabled                      true

media.rdd-ffmpeg.enabled                       true
media.rdd-ffvpx.enabled                        false
media.rdd-vpx.enabled                          false

Enable for Intel graphics. Enable also for NVIDIA 3000+ series graphics
using proprietary driver (v510+) and NVIDIA-NVDEC enabled VA-API driver
(v0.0.5+).
media.av1.enabled                              false

Enable FFMPEG VA-API decoding support for WebRTC on Linux.
media.navigator.mediadatadecoder_vpx_enabled   true
```

### <a id="wikis">See also, wikis at Arch Linux

* [Firefox](https://wiki.archlinux.org/title/Firefox)
* [Hardware video acceleration](https://wiki.archlinux.org/title/Hardware_video_acceleration)

