# Hardware h264 encoding using vaapi

### Original Project Description
This plugin is designed to enable hardware-accelerated transcoding profiles using VAAPI (Video Acceleration API) on Linux systems. It's important to note that this is an experimental test project , and significant tinkering will likely be necessary to get it working properly with your specific hardware configuration.

### Enhanced Features
Building upon the original foundation, this modified version now includes:

1. RC Mode Support : Added support for Rate Control modes, allowing more flexibility in managing video quality and file size during transcoding.
2. Lossless Original Video Conversion : Implemented a feature that enables lossless conversion of the original video files, preserving the highest possible quality while still benefiting from hardware acceleration when applicable.

For more information on vaapi and hardware acceleration:

- https://jellyfin.org/docs/general/administration/hardware-acceleration.html#enabling-hardware-acceleration
- https://wiki.archlinux.org/index.php/Hardware_video_acceleration#Comparison_tables


# Building a compatible docker image

Official docker images do not ship with required libraries for hardware transcode.
You can build your own image with the following Dockerfile:

```Dockerfile
ARG VERSION=v6.1.0
FROM chocobozzz/peertube:${VERSION}-bookworm


# install dependencies for vaapi
RUN 	   apt update \
	&& apt install -y --no-install-recommends wget apt-transport-https \
	&& echo "deb http://deb.debian.org/debian/ bookworm main contrib non-free non-free-firmware" > /etc/apt/sources.list \
  && echo "deb http://deb.debian.org/debian/ bookworm-updates main contrib non-free non-free-firmware" >> /etc/apt/sources.list \
  && echo "deb http://deb.debian.org/debian/ bookworm-backports main contrib non-free non-free-firmware" >> /etc/apt/sources.list \
  && echo "deb https://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware" >> /etc/apt/sources.list \
	&& apt update \
	&& apt install -y --no-install-recommends libmfx1 libmfx-tools libva-dev libmfx-dev intel-media-va-driver-non-free vainfo \
	&& rm /var/lib/apt/lists/* -fR \
  && sed -i 's/(scaleFilterValue/(scaleFilterValue \&\& !builderResult.result.copy/' ./packages/ffmpeg/dist/shared/presets.js \
  && sed -i '/find \/data ! -user peertube -exec  chown peertube:peertube {} \\;/a \    if [ -e '/dev/dri/renderD128' ]; then\n        chmod 777 /dev/dri/renderD128\n    fi' /usr/local/bin/entrypoint.sh
```

If you are using an older Intel CPU (generation 7 and older), replace `intel-media-va-driver-non-free` by `i965-va-driver-shaders`.


# Running the docker image

In order to access the GPU inside docker, the `docker-compose.yml` should be adapted as follow.
Note that you must find the id of the `render` group on your machine.
You can use `grep $(ls -l /dev/dri/renderD128 | awk '{print($4)}') /etc/group | cut -d':' -f3`  to find the id.


```yaml
version: "3"

services:
  peertube:
    # replace image key with
    build:
      context: .
      args:
        VERSION: v6.1.0
    # usual peertube configuration
    # ...

    # add these keys
    group_add:
      - <replace with the id of the render group>
    devices:
      # VAAPI Devices
      - /dev/dri:/dev/dri
```
