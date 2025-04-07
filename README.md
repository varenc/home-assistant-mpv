# mpv integration for Home Assistant

[![hacs_badge](https://img.shields.io/badge/HACS-Custom-41BDF5.svg?style=for-the-badge)](https://github.com/hacs/integration)

An Home Assistant Media Player integration for the [mpv][mpv] media player, using mpv's [JSON IPC][mpv-ipc] API.

## Setup

### Installation

The integration can be installed by adding it as a custom repository to [HACS][hacs]. In Home Assistant, navigate to
HACS > Integrations > Custom repositories (in the top-right menu). Under Repository enter `oxan/home-assistant-mpv`,
and under Category select Integration. The integration should now appear in HACS.

### Configuration

Start mpv with the `input-ipc-server` option set to the socket location:
```sh
mpv --input-ipc-server=/path/to/mpv-socket
```

Configure the integration in the Home Assistant `configuration.yaml` file:
```yaml
media_player:
  - platform: mpv
    name: "MPV Player"
    server:
      path: /path/to/mpv-socket
```

Restart Home Assistant and enjoy!

#### Remote mpv

It is also possible to connect to a remote mpv instance over the network. For security reasons, it's strongly recommended to use SSH tunneling to create an encrypted connection rather than exposing the socket directly over the network. Directly exposing the mpv control socket is a security risk, as it would allow anybody with access to the port to execute arbitrary commands via mpv's `run` command.

First, ensure that `socat` is installed, and create a script that:
1. Uses socat to bridge the mpv Unix socket to a localhost-only port on your machine
2. Creates an SSH reverse tunnel to securely forward that port to your Home Assistant server

The script must have the `.run` extension and be executable (run `chmod +x secure-mpv-tunnel.run`):
```sh
#!/bin/bash
# Replace these values with your actual settings
MPV_SOCKET="/path/to/mpv-socket"
LOCAL_PORT="2352"
HA_HOST="homeassistant.local"  # Your Home Assistant machine
HA_USER="user"                 # SSH user on your Home Assistant machine

# Exit if port already in use (script likely already running)
if nc -z 127.0.0.1 ${LOCAL_PORT} 2>/dev/null; then
  echo "Port ${LOCAL_PORT} already in use, assuming secure-mpv-tunnel script is already running"
  exit 0
fi

# Create localhost-only TCP listener that connects to mpv socket
socat TCP-LISTEN:${LOCAL_PORT},bind=127.0.0.1,reuseaddr,fork UNIX-CONNECT:${MPV_SOCKET} &

# Create secure SSH tunnel
ssh -N -R ${LOCAL_PORT}:localhost:${LOCAL_PORT} ${HA_USER}@${HA_HOST}
```

> **Note:** This method requires SSH access to your Home Assistant instance. If using the popular [Home Assistant SSH addon](https://github.com/hassio-addons/addon-ssh), you'll need to enable `allow_remote_port_forwarding: true` in its configuration for the reverse tunnel to work.

Start mpv with the `--script` option to run the script on startup:
```sh
mpv --input-ipc-server=/path/to/mpv-socket --script=/path/to/secure-mpv-tunnel.run
```

Finally, configure the integration to connect to the local end of the tunnel:
```yaml
media_player:
  - platform: mpv
    name: "MPV Player"
    server:
      host: localhost
      port: 2352
```

This approach ensures all communication between Home Assistant and mpv is encrypted through SSH, preventing unauthorized access to your mpv instance.

#### Other useful mpv options

You can additionally use the `--idle` mpv option to have it remain alive if no media is playing.

#### Playback using local paths

When starting playback through Home Assistant, by default it will stream all media through its own HTTP server. If mpv
and Home Assistant can access the media files using the same filesystem path, you can disable this and play media
directly from the filesystem. This reduces resource usage and allows mpv to find external subtitle files.

```yaml
media_player:
  - platform: mpv
    server:
      path: /path/to/mpv-socket
    proxy_media: false
```

[hacs]: https://hacs.xyz/
[mpv]: https://mpv.io/
[mpv-ipc]: https://mpv.io/manual/stable/#json-ipc
