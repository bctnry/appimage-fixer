# appimage-fixer

Install an AppImage system-wide by **unpacking** it instead of running it from a
FUSE mount, then repair the permissions of the bundled `chrome-sandbox` binary
and emit a matching **uninstall script** for every installation.

This is mainly useful for Chromium/Electron apps (VS Code, Obsidian, Joplin,
etc.) that fail with:

> The SUID sandbox helper binary was found, but is not configured correctly.
> Rather than run without sandboxing I'm aborting now.

The Chromium sandbox requires `chrome-sandbox` to be owned by `root` and to have
the setuid bit set (`4755`). An AppImage that you double-click runs from a
read-only FUSE mount, so that permission can never be applied and the app either
refuses to start or needs an unsafe `--no-sandbox`. Unpacking to a normal
directory makes the fix possible.

## Requirements

- A POSIX-ish Linux with `bash` 4+ and the usual coreutils (`find`, `stat`,
  `mktemp`, `cp`, `chown`, `chmod`).
- `root` (`sudo`) for installing an app, because `chrome-sandbox` must be owned
  by `root` and setuid. Installing the tool itself does not need root (see
  below).
- No FUSE needed: the AppImage is unpacked with its built-in
  `--appimage-extract` mode.

## Installing this tool

`appimage-fixer` is a single self-contained script with no build step. Put it on
your `PATH` so you can call it from anywhere as `sudo appimage-fixer ...`.
Installing it does **not** need root; only running it does.

From a checkout of this directory:

```sh
sudo install -m 755 appimage-fixer /usr/local/bin/appimage-fixer
```

Or, if you only have a download URL (replace the URL with where you got the
file), fetch it straight into place:

```sh
sudo curl -fsSL <url-to-appimage-fixer> -o /usr/local/bin/appimage-fixer
sudo chmod 755 /usr/local/bin/appimage-fixer
```

Verify it is reachable:

```sh
command -v appimage-fixer && appimage-fixer --help
```

To remove the tool itself (this is unrelated to the per-app uninstallers it
generates):

```sh
sudo rm -f /usr/local/bin/appimage-fixer
```

## Usage

```sh
sudo appimage-fixer [options] /path/to/SomeApp-1.2.3.AppImage
```

With no options it unpacks to `/opt/SomeApp-1.2.3`, creates the launcher
`/usr/local/bin/SomeApp-1.2.3`, and writes the uninstaller
`/usr/local/bin/uninstall-SomeApp-1.2.3`.

Common invocations:

```sh
# Default location, name derived from the file name
sudo appimage-fixer ~/Downloads/Obsidian-1.5.3.AppImage       # -> /opt/Obsidian-1.5.3

# Stable location that does not include the version number
sudo appimage-fixer --name obsidian ~/Downloads/Obsidian-1.5.3.AppImage   # -> /opt/obsidian

# Exact destination directory
sudo appimage-fixer --dest /opt/obsidian ~/Downloads/Obsidian-1.5.3.AppImage

# Also add a desktop entry
sudo appimage-fixer --desktop ~/Downloads/Obsidian-1.5.3.AppImage

# Reinstall over an existing copy
sudo appimage-fixer --force ~/Downloads/Obsidian-1.6.0.AppImage
```

### Options

| Option | Description |
| --- | --- |
| `-n`, `--name NAME` | Application name used for the directory, launcher and uninstaller. |
| `-d`, `--dest DIR` | Exact directory to unpack into (default `/opt/<name>`). |
| `-b`, `--bin-dir DIR` | Where the launcher symlink goes (default `/usr/local/bin`). |
| `-u`, `--uninstall-dir DIR` | Where the per-install uninstaller is written (default `/usr/local/bin`). |
| `--desktop` / `--desktop-dir DIR` | Also install a `.desktop` entry (default `/usr/share/applications`). |
| `--no-symlink` | Skip the launcher symlink. |
| `--no-uninstaller` | Skip generating an uninstall script. |
| `-f`, `--force` | Replace an existing installation. |
| `-k`, `--keep-temp` | Keep the temporary extraction directory (useful for debugging). |
| `--dry-run` | Print what would happen without changing anything (no root needed). |
| `-h`, `--help` | Show help. |

## What it does

1. **Unpacks** the AppImage with `<file> --appimage-extract` inside a temporary
   directory created with `mktemp -d` (removed on exit unless `--keep-temp`).
   The `squashfs-root` it produces is moved to the install directory.
2. **Sets ownership/permissions**: the tree becomes `root:root`, directories are
   made traversable and files world-readable (some AppImages ship the squashfs
   root as `0700`, which `cp -a` would preserve and which makes the app
   unrunnable for normal users), `AppRun` is made executable, and every
   `chrome-sandbox` found is fixed with `chown root:root` + `chmod 4755` (setuid
   root). It warns on a `nosuid` mount and verifies the resulting mode where
   possible.
3. **Generates an uninstall script** for that specific installation at
   `<uninstall-dir>/uninstall-<name>`. It records the exact install directory,
   launcher and desktop file, so it removes precisely what was installed, then
   deletes itself. It is stored outside the install directory and re-executes
   from a temporary copy, so removing the installation (and itself) is safe.

## Uninstalling

Each installation gets its own uninstaller, named after it:

```sh
sudo /usr/local/bin/uninstall-obsidian
```

## Troubleshooting

**`zsh: permission denied: myapp`** right after installing.

The launcher symlink is fine; the app directory (or something inside it) is not
readable or executable by your user. This happens when the AppImage ships its
squashfs root with a restrictive mode such as `0700`, which the installer used
to preserve while unpacking. Check with:

```sh
ls -ld /opt/myapp
```

If it shows `drwx------`, either reinstall with a version that normalizes
permissions, or repair it in place:

```sh
sudo chmod 755 /opt/myapp
sudo find /opt/myapp -type d -exec chmod a+rx {} +
sudo find /opt/myapp -type f -exec chmod a+rX {} +
```

Then refresh your shell's command cache (`hash -r` in zsh/bash) and try again.

## Notes

- The install directory is owned by `root`. To update the app, re-run the
  installer with `--force`; do not edit files in place as a normal user.
- `--name` is restricted to `[A-Za-z0-9._+-]` so it can never escape the target
  directory or break the generated scripts.
- The script refuses to install into `/` and refuses to overwrite an existing
  installation (or launcher/uninstaller) unless `--force` is given.
