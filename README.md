# ffmpeg-on-clear-linux

Run [FFmpeg](https://ffmpeg.org/) on [Clear Linux](https://clearlinux.org/) including H.264 and VP9 playback on the GPU.

* [What's included](#whats-included)
* [Requirements](#requirements)
* [Initial preparation](#initial-preparation)
* [Building and installation](#building-and-installation)
* [Multiple libs in x264 and x265](#multiple-libs-in-x264-and-x265)
* [Determining the VA-API driver to use](#determining-the-va-api-driver-to-use)
* [Using the NVIDIA NVDEC-enabled VA-API driver](#using-the-nvidia-nvdec-enabled-va-api-driver)
* [Firefox config file](#firefox-config-file)
* [Firefox settings](#firefox-settings)
* [Chromium installation and configuration](#chromium-installation-and-configuration)
* [Google Chrome installation and run script](#google-chrome-installation-and-run-script)
* [Vivaldi installation and run script](#vivaldi-installation-and-run-script)
* [Brave installation and run script](#brave-installation-and-run-script)
* [Caveat with RPM package installation](#caveat-with-rpm-package-installation)
* [How can I make sure hardware acceleration is working?](#how-can-i-make-sure-hardware-acceleration-is-working)
* [Watch HDR videos](#watch-hdr-videos)
* [See also](#see-also-advert-at-clear-linux-and-wikis-at-arch-linux)

## What's included

This is an automation **how-to** for building FFmpeg and minimum dependencies. My motivation is nothing more than wanting hardware acceleration during video playback. Who doesn't want that? Thank you, @xtknight for the initial [VP9](https://github.com/xtknight/vdpau-va-driver-vp9) acceleration bits. Thank you also, @xuanruiqi for the [VP9-update](https://github.com/xuanruiqi/vdpau-va-driver-vp9) to include additional fixes.

```text
bin        Browser launch scripts to be copied to $HOME/bin/.
build-all  Top-level script for building dependencies and FFmpeg.
desktop    Desktop files for $HOME/.local/share/applications/.
extras     Complementary YouTube player for testing nvdec/nvenc.
localenv   Set your NVIDIA GPU's CUDA compute capability here.
scripts    Contains individual build-install scripts.
```

## Requirements

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

## Initial preparation

This section is for folks using the NVIDIA proprietary driver.

Until CUDA reaches full compatibility with GCC 11.x, install the `c-extras-gcc10` bundle. This applies if `gcc --version` returns 11 or later. A flag will be passed to `nvcc` to use `gcc-10`.

```bash
$ sudo swupd bundle-add c-extras-gcc10
```

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

## Building and installation

The build and installation process for FFmpeg can be done in one step.

```bash
$ sudo bash build-all
```

Or become root and run each script individually. Be sure to run `000-install-dependencies` first if choosing this path. Various scripts exit silently depending on whether `/opt/nvidia/bin/nvidia-settings` or `/usr/local/cuda/cuda.h` exists.

```bash
$ sudo root

cd scripts
./000-install-dependencies
./100-build-nv-codec-headers
./110-build-libvdpau
...
```

The `builddir` (once created) serves as a cache folder. Remove the correspondent `*.tar.gz` file(s) to re-fetch or re-clone from the internet.

I'm hoping that the build process succeeds for you as it does for me. However, I may have a bundle installed that's missing in `000-install-dependencies`. Please reach out if that's the case. The Media SDK is included in the FFmpeg build for Intel hardware, but not yet tested what else is needed for that platform.

Remember to add `/usr/local/bin` to your `PATH` environment variable if not already done, preferably before `/usr/bin`.

## Multiple libs in x264 and x265

The build scripts build each library independently. Below you will find that `x264` supports 8-bits and 10-bits output.

```bash
$ x264 --help | grep "Output bit depth"
Output bit depth: 8/10

$ ffmpeg -hide_banner -h encoder=libx264 | grep "Supported pixel formats"
Supported pixel formats: yuv420p yuvj420p yuv422p yuvj422p yuv444p yuvj444p
nv12 nv16 nv21 yuv420p10le yuv422p10le yuv444p10le nv20le gray gray10le
```

Similarly x265 supports 8-bits, 10-bits and 12-bits output.

```bash
$ x265 --help | grep "Output bit depth"
-D/--output-depth 8|10|12    Output bit depth... Default 8

$ ffmpeg -hide_banner -h encoder=libx265 | grep "Supported pixel formats"
Supported pixel formats: yuv420p yuvj420p yuv422p yuvj422p yuv444p yuvj444p
gbrp yuv420p10le yuv422p10le yuv444p10le gbrp10le yuv420p12le yuv422p12le
yuv444p12le gbrp12le gray gray10le gray12le
```

## Determining the VA-API driver to use

For hardware acceleration to work, the browser may have the driver built-in or you will need a suitable driver, i.e. `ls /usr/lib64/dri/*_drv_video.so`. To be sure, run `vainfo` in a terminal window. For AMD try `LIBVA_DRIVER_NAME=r600 vainfo` or `LIBVA_DRIVER_NAME=radeonsi vainfo`. For Intel the `iHD` driver is newer. So check first `LIBVA_DRIVER_NAME=iHD vainfo` or try `LIBVA_DRIVER_NAME=i965 vainfo`.

Below see captured output for the NVIDIA driver.

```bash
$ LIBVA_DRIVER_NAME=nvidia vainfo

libva info: VA-API version 1.11.0
libva info: User environment variable requested driver 'nvidia'
libva info: Trying to open /usr/lib64/dri/nvidia_drv_video.so
libva info: Found init function __vaDriverInit_1_11
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.11 (libva 2.11.0)
vainfo: Driver version: Splitted-Desktop Systems VDPAU backend for VA-API - 0.7.4
vainfo: Supported profile and entrypoints
      VAProfileMPEG2Simple            : VAEntrypointVLD
      VAProfileMPEG2Main              : VAEntrypointVLD
      VAProfileMPEG4Simple            : VAEntrypointVLD
      VAProfileMPEG4AdvancedSimple    : VAEntrypointVLD
      VAProfileH264ConstrainedBaseline: VAEntrypointVLD
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointVLD
      VAProfileVC1Simple              : VAEntrypointVLD
      VAProfileVC1Main                : VAEntrypointVLD
      VAProfileVC1Advanced            : VAEntrypointVLD
      VAProfileVP9Profile0            : VAEntrypointVLD
```

## Using the NVIDIA NVDEC-enabled VA-API driver

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

## Firefox config file

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

## Firefox settings

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

Verify enabled, necessary for the nvidia-nvdec enabled driver to work.
media.rdd-process.enabled                      true

media.rdd-ffmpeg.enabled                       true
media.rdd-ffvpx.enabled                        false
media.rdd-vpx.enabled                          false
media.av1.enabled                              false

Enable FFMPEG VA-API decoding support for WebRTC on Linux.
media.navigator.mediadatadecoder_vpx_enabled   true
```

## Chromium installation and configuration

[Chromium](https://dev.chromium.org/Home) is an open-source browser project. Some say it's a browser made for developers. The [chromium-latest-linux](https://github.com/scheib/chromium-latest-linux) repository works great for launching Chromium including VP9 media playback. Unfortunately, the browser cannot decode H.264-ACC media.

**Installation**

Change directory to your home directory. The launch script for Chromium will look for the folder here. Run the `update.sh` script initially and periodically to fetch the latest snapshot. The other scripts `update-and-run.sh` and `run-chrome.sh` are not used.

```bash
$ pushd $HOME
$ git clone https://github.com/scheib/chromium-latest-linux.git
$ cd chromium-latest-linux
$ ./update.sh
$ popd
```

**Edit ~/bin/run-chromium-latest**

First copy the custom launch script to your `bin` folder.

```bash
$ mkdir -p ~/bin
$ cp ~/Downloads/ffmpeg-on-clear-linux/bin/run-chromium-latest ~/bin/.
```

Scroll down towards the end of the file. Update the value for `LIBVA_DRIVER_NAME` or leave it `auto`. The driver name is overridden automatically for NVIDIA hardware.

Opening new windows may be larger than the initial window. After a while, that can be annoying. The `--window-size=x,y` option resolves this issue. Optionally adjust the width and height (in pixels) appropriate for your display.

Accelerated 2D canvas is required (default enabled) to decode videos on the GPU. Two more options `--use-gl` and `--enable-features=VaapiVideoDecoder` are needed for hardware acceleration to work when watching a video. Hardware acceleration stopped working in Google Chrome 98. The `--disable-features=UseChromeOSDirectVideoDecoder` option resolves the issue by decoding videos using VDAVideoDecoder.

Hardware accelerated video decoding applies to Google Chrome, Vivaldi, and Brave browsers.

```bash
# Launch browser.
export FONTCONFIG_PATH=/usr/share/defaults/fonts
export LIBVA_DRIVERS_PATH=/usr/lib64/dri
export LIBVA_DRIVER_NAME=auto

if [[ -d /opt/nvidia && -f $LIBVA_DRIVERS_PATH/nvidia_drv_video.so ]]
then
    # add /opt/nvidia/{lib64,lib} to path
    export LD_LIBRARY_PATH="/opt/nvidia/lib64:/opt/nvidia/lib"

    # libva doesn't yet know which driver to load for the nvidia-drm driver
    # this forces libva to load the nvidia backend
    export LIBVA_DRIVER_NAME=nvidia

    # browsers based on chromium support VDPAU acceleration
    export VDPAU_DRIVER=nvidia
fi

[[ $XDG_SESSION_TYPE == wayland ]] && GL=egl || GL=desktop

exec "$EXECCMD" --window-size=1100,900 \
    --disable-features=UseChromeOSDirectVideoDecoder \
    --enable-accelerated-2d-canvas --enable-smooth-scrolling \
    --use-gl=$GL --enable-features=VaapiVideoDecoder \
    --user-data-dir="$DATADIR" $* &> /dev/null &
```

**Running**

On first launch go into `Settings -> Appearance -> Customize fonts` and change the fonts. Metrically compatible with `Times New Roman`, `Arial`, and `Courier New` are `Tinos`, `Arimo`, and `Cousine` respectively. Optionally go into `Settings -> Advanced -> System` and disable "Use hardware acceleration when available". This may be helpful if the GPU is lacking or you prefer the CPU to decode videos.

A desktop file is created the first time it is run and placed in `~/.local/share/applications`. You may run Chromium using the command-line or search for "Chromium" in Application Finder. Launching from the desktop will run the same script.

```bash
$ ~/bin/run-chromium-latest
```

## Google Chrome installation and run script

[Google Chrome](https://www.google.com/chrome/) is a browser built by Google. You will find that the browser is quite fast. For NVIDIA hardware, one nicety is that video playback for VP9 media utilizes the Video Engine. That saves me 15 watts of power consumption versus Chromium and Firefox.

**Installation**

The `RPM` file for Google Chrome can be found at [Google](https://www.google.com/chrome/) and [pkgs.org](https://pkgs.org/download/google-chrome). At the time of writing, I installed version 98.0.4758.80.

**Note:** Installing Google Chrome will add the Google repository so your system will automatically keep Google Chrome up to date. If you don't want Google's repository (which is what we want), do `sudo touch /etc/default/google-chrome` before installing the package. The reason is the package will fail auto-install without the `--nodeps` flag.

The `-U` flag to `rpm` installs the new package, otherwise upgrades the installed package. Periodically, obtain the current stable release and run the `rpm` command as shown.

```bash
$ sudo mkdir -p /etc/default && sudo touch /etc/default/google-chrome

# install-or-update package from Google
$ sudo rpm -Uvh --nodeps \
    ~/Downloads/google-chrome-stable_current_x86_64.rpm 2>/dev/null

# install-or-update package from pkgs.org, change version accordingly
$ sudo rpm -Uvh --nodeps \
    ~/Downloads/google-chrome-stable-97.0.4692.71-1.x86_64.rpm 2>/dev/null
```

**Edit ~/bin/run-chrome-stable**

Copy the launch script and corresponding desktop file. Refer to the notes above for editing the script i.e. `LIBVA_DRIVER_NAME`, et al.

```bash
$ mkdir -p ~/bin && mkdir -p ~/.local/share/applications

$ cp ~/Downloads/ffmpeg-on-clear-linux/bin/run-chrome-stable ~/bin/.
$ cp ~/Downloads/ffmpeg-on-clear-linux/desktop/google-chrome.desktop \
       ~/.local/share/applications/.
```

**Running**

On first launch (just like with Chromium), you may want to go into `Settings -> Appearance -> Customize fonts` and change the fonts. Metrically compatible with `Times New Roman`, `Arial`, and `Courier New` are `Tinos`, `Arimo`, and `Cousine` respectively. Optionally go into `Settings -> Advanced -> System` and disable "Use hardware acceleration when available". Like with Chromium, this may be helpful if the GPU is lacking or you prefer the CPU to decode videos.

Run Chrome using the command-line or search for "Google Chrome" in Application Finder.

```bash
$ ~/bin/run-chrome-stable
```

## Vivaldi installation and run script

[Vivaldi](https://vivaldi.com) is yet another open-source browser. The main highlight is being able to communicate in a much more organized way, while keeping control of your data. That sounds delightful! Similarly to Google Chrome, this also utilizes the Video Engine while watching VP9 media.

**Installation**

The `RPM` file for Vivaldi can be found at [Vivaldi](https://vivaldi.com/download/). At the time of writing, I installed version 5.0.2497.51.

**Note:** Installing Vivaldi will add the Vivaldi repository so your system will automatically keep Vivaldi up to date. If you don't want Vivaldi's repository (which is what we want), do `sudo touch /etc/default/vivaldi` before installing the package. The reason is the package will fail auto-install without the `--nodeps` flag.

The `-U` flag to `rpm` installs the new package, otherwise upgrades the installed package. Periodically, obtain the current stable release and run the `rpm` command as shown.

```bash
$ sudo mkdir -p /etc/default && sudo touch /etc/default/vivaldi

# install-or-update package, change version accordingly
$ sudo rpm -Uvh --nodeps \
    ~/Downloads/vivaldi-stable-5.0.2497.38-1.x86_64.rpm 2>/dev/null
```

**Edit ~/bin/run-vivaldi-stable**

Copy the launch script and corresponding desktop file. See Chromium section above for editing the script i.e. `LIBVA_DRIVER_NAME`, et al.

```bash
$ mkdir -p ~/bin && mkdir -p ~/.local/share/applications

$ cp ~/Downloads/ffmpeg-on-clear-linux/bin/run-vivaldi-stable ~/bin/.
$ cp ~/Downloads/ffmpeg-on-clear-linux/desktop/vivaldi-stable.desktop \
       ~/.local/share/applications/.
```

**Running**

On first launch go into `Settings -> Webpages -> Fonts -> Default Fonts` and change the default fonts. Metrically compatible with `Times New Roman`, `Arial`, and `Courier New` are `Tinos`, `Arimo`, and `Cousine` respectively. Optionally go into `Settings -> Webpages` and uncheck "Use Hardware Acceleration When Available". This may be helpful if the GPU is lacking or you prefer the CPU to decode videos.

Run Vivaldi using the command-line or search for "Vivaldi" in Application Finder.

```bash
$ ~/bin/run-vivaldi-stable
```

## Brave installation and run script

[Brave](https://brave.com) is an open-source browser, reimagined. It claims three times faster than Google Chrome and better privacy than Firefox. Similarly to Google Chrome and Vivaldi, this too utilizes the Video Engine while watching VP9 media.

**Installation**

The `RPM` file for Brave can be found at [sourceforge.net](https://sourceforge.net/projects/brave-browser.mirror/files/). Go to [pkgs.org](https://pkgs.org/download/brave) and scroll to the bottom of the page. It will mention the current release version. At the time of writing, I installed version 1.35.100.

**Note:** Installing Brave will add the Brave repository so your system will automatically keep Brave up to date. If you don't want Brave's repository (which is what we want), do `sudo touch /etc/default/brave-browser` before installing the package. The reason is the package will fail auto-install without the `--nodeps` flag.

The `-U` flag to `rpm` installs the new package, otherwise upgrades the installed package. Periodically, obtain the current stable release and run the `rpm` command as shown.

```bash
$ sudo mkdir -p /etc/default && sudo touch /etc/default/brave-browser

# install-or-update package, change version accordingly
$ sudo rpm -Uvh --nodeps \
    ~/Downloads/brave-browser-1.34.80-1.x86_64.rpm 2>/dev/null
```

**Edit ~/bin/run-brave-stable**

Copy the launch script and corresponding desktop file. See Chromium section above for editing the script i.e. `LIBVA_DRIVER_NAME`, et al.

```bash
$ mkdir -p ~/bin && mkdir -p ~/.local/share/applications

$ cp ~/Downloads/ffmpeg-on-clear-linux/bin/run-brave-stable ~/bin/.
$ cp ~/Downloads/ffmpeg-on-clear-linux/desktop/brave-browser.desktop \
       ~/.local/share/applications/.
```

**Running**

On first launch go into `Settings -> Appearance -> Customize fonts` and change the fonts. Metrically compatible with `Times New Roman`, `Arial`, and `Courier New` are `Tinos`, `Arimo`, and `Cousine` respectively. Optionally go into `Settings -> Additional settings -> System` and disable "Use hardware acceleration when available". Like with other Chromium-based browsers, this may be helpful if the GPU is lacking or you prefer the CPU to decode videos.

Run Brave using the command-line or search for "Brave Web Browser" in Application Finder.

```bash
$ ~/bin/run-brave-stable
```

## Caveat with RPM package installation

It feels hacky in Clear Linux installing a package that was built for another platform such as RedHat. For peace of mind, check for missing library dependencies using the `ldd` utility. Ensure nothing is missing in the output. If true, then install missing packages with `sudo swupd bundle-add PKGNAME`. Run `sudo swupd search LIBNAME` if needed.

Another solution is building from source. This is likely not necessary, although becomes reality if unable to meet library dependencies. Uninstall the browser with `sudo rpm -e NAME`, given below.

```bash
$ ldd ~/chromium-latest-linux/latest/chrome 2>/dev/null | grep "not found$"
$ ldd /opt/brave.com/brave/brave 2>/dev/null | grep "not found$"
$ ldd /opt/google/chrome/chrome 2>/dev/null | grep "not found$"
$ ldd /opt/vivaldi/vivaldi-bin 2>/dev/null | grep "not found$"
```

Hackiness put aside, a benefit of using a package for installation is that the package can be uninstalled easily. Optionally remove your browser data and settings. Though, be sure to export your bookmarks.

```bash
# Brave
$ sudo rpm -e brave-browser 2>/dev/null
$ sudo rm -f /etc/default/brave-browser
$ rm -f ~/.local/share/applications/brave-browser.desktop
$ rm -fr ~/.cache/BraveSoftware/Brave-Browser
$ rm -fr ~/.config/BraveSoftware/Brave-Browser  (optional)
```

```bash
# Google Chrome
$ sudo rpm -e google-chrome-stable 2>/dev/null
$ sudo rm -f /etc/default/google-chrome
$ rm -f ~/.local/share/applications/google-chrome.desktop
$ rm -fr ~/.cache/google-chrome
$ rm -fr ~/.config/google-chrome  (optional)
```

```bash
# Vivaldi
$ sudo rpm -e vivaldi-stable 2>/dev/null
$ sudo rm -f /etc/default/vivaldi
$ rm -f ~/.local/share/applications/vivaldi-stable.desktop
$ rm -fr ~/.cache/vivaldi
$ rm -fr ~/.config/vivaldi  (optional)
```

## How can I make sure hardware acceleration is working?

In Brave, Chromium, Google Chrome, and Vivaldi, check the `chrome://gpu` page. In Firefox, check the `about::support` page. Another way is running a utility suited for your hardware while watching a video.

1. `watch -n 1 /opt/nvidia/bin/nvidia-smi` to check if "GPU-Util" percentage goes up
2. `sudo intel_gpu_top` to check if percentage under the "Video" section goes up
3. `watch -n 1 sudo intel_gpu_frequency` to check if the frequency goes up

## Watch HDR videos

To play HDR videos, see `youtube-play` found in the extras folder.

## See also, advert at Clear Linux and wikis at Arch Linux

* [Annoucement and benchmark results](https://community.clearlinux.org/t/ffmpeg-supporting-h-264-and-vp9-hardware-acceleration-in-firefox/6148)
* [Chromium](https://wiki.archlinux.org/title/Chromium)
* [Firefox](https://wiki.archlinux.org/title/Firefox)
* [Google Chrome](https://wiki.archlinux.org/title/Google_chrome)
* [Hardware video acceleration](https://wiki.archlinux.org/title/Hardware_video_acceleration)
* [Vivaldi](https://wiki.archlinux.org/title/Vivaldi)

