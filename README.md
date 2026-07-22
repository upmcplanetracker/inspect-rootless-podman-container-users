Audit the actual runtime users of your Podman Quadlet containers.

## What it does

This script walks every `.container` unit file in a given directory (default: `~/.config/containers/systemd`), extracts the `Image=` from each, and reports three things per container:

1. **`/etc/passwd`** from inside the image - what accounts exist for reference.
2. **`Config.User`** from the image itself - the launch-time default only. This does **not** reflect an internal privilege drop via `setpriv`, `gosu`, or `su-exec`.
3. **Live user(s)** via `podman top <container> user` against the **running** container - the only check that tells you the truth for images that drop privileges internally after starting as root.

At the end, containers are categorized and summarized:
- Running **only** as root
- Running as root **plus** other users
- Running as **non-root**
- **Stopped** / not running
- **Unknown** / check failed

It also probes the image for known privilege-drop conventions (PUID/PGID for linuxserver.io-style s6-overlay images, or `setpriv`/`gosu`/`su-exec` style drops) and reports what mechanism it found.

## Installation

```bash
# Download to a directory in your PATH
wget -O ~/.local/bin/inspect-container-users   https://raw.githubusercontent.com//YOURREPO/main/inspect-container-users

# Make it executable
chmod +x ~/.local/bin/inspect-container-users
```

> Make sure `~/.local/bin` is in your `$PATH`. If not, add it to your shell profile:
> ```bash
> echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
> source ~/.bashrc
> ```

## Usage

```bash
inspect-container-users [options] [quadlet_dir]
```

### Options

| Flag | Long form | Description |
|------|-----------|-------------|
| `-q` | `--quiet` | Suppress per-container detail output. Shows a progress bar and prints only the summary. |
| `-l` | `--log-to-file` | Write all per-container detail to a timestamped log file in `~/.local/share/inspect-container-users/`. The summary still prints to stdout. |
| `-h` | `--help` | Show help message and exit. |

### Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `quadlet_dir` | `~/.config/containers/systemd` | Directory containing your `.container` Quadlet unit files. |

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `LOG_DIR` | `~/.local/share/inspect-container-users` | Directory where timestamped log files are written when using `--log-to-file`. |

## Examples

### Default — verbose output to stdout
```bash
inspect-container-users
```

### Summary only with progress bar
```bash
inspect-container-users --quiet
```

### Verbose to stdout AND write detail to a timestamped log
```bash
inspect-container-users --log-to-file
```

### Best combo — clean summary to stdout, detail archived to log
```bash
inspect-container-users -q -l
```

### Check a different quadlet directory
```bash
inspect-container-users /etc/containers/systemd
```

### Custom log directory
```bash
LOG_DIR=/var/log/container-audit inspect-container-users -q -l
```

## Sample Output

```
Found 12 unit file(s) in /home/user/.config/containers/systemd

===================================================================
SUMMARY
===================================================================

Running ONLY as root (3):
  - plex (image: lscr.io/linuxserver/plex:latest, live user(s): root, mechanism: PUID/PGID)
  - sonarr (image: lscr.io/linuxserver/sonarr:latest, live user(s): root, mechanism: PUID/PGID)
  - radarr (image: lscr.io/linuxserver/radarr:latest, live user(s): root, mechanism: PUID/PGID)

Running as root + other users (0):
  (none)

Running as non-root (4):
  - caddy (live user(s): caddy)
  - syncthing (live user(s): syncthing)
  - gitea (live user(s): git)
  - vaultwarden (live user(s): vaultwarden)

Not running / stopped (2):
  - jellyfin (image: docker.io/jellyfin/jellyfin:latest)
  - audiobookshelf (image: ghcr.io/advplyr/audiobookshelf:latest)

Unknown / check failed (0):
  (none)

Done.
```

With `--quiet --log-to-file`, you get a progress bar during scanning:

```
Found 12 unit file(s) in /home/user/.config/containers/systemd
Logging detail to: /home/user/.local/share/inspect-container-users/inspect-20260722_143531.log

  [==============================>                    ]  50% (6/12) plex

===================================================================
SUMMARY
===================================================================
...

Detail log: /home/user/.local/share/inspect-container-users/inspect-20260722_143531.log
Done.
```

## How it works

1. **Scans** the quadlet directory for `.container` files.
2. **Extracts** `Image=`, `ContainerName=`, and `User=` from each unit file.
3. **Pulls** `/etc/passwd` from the image to show available accounts.
4. **Inspects** the image's `Config.User` to show the default launch user.
5. **Checks** the running container with `podman top` to see the *actual* process owner(s).
6. **Probes** the image for privilege-drop mechanisms if root is detected at runtime.
7. **Summarizes** everything into actionable categories.

## Requirements

- [Podman](https://podman.io/) installed and configured
- Bash 4.0+
- Images must be pulled locally (the script does not auto-pull)

## License

MIT
