# Multimedia and CD/DVD Setup

This guide covers the multimedia setup playbook at `linux/playbooks/multimedia-setup.yml`.

It is focused on CD ripping on Debian-based Linux hosts, with optional (disabled-by-default) notes for DVD support.

## Compatibility

This setup is intended for the same distributions listed in the root README:

- Ubuntu 22.04 LTS / 24.04 LTS
- Debian 11 (Bullseye) / 12 (Bookworm)
- Linux Mint (Debian/Ubuntu-based editions)

The playbook uses `apt` packages and is not intended for non-Debian-based systems.

## Hardware requirement

CD ripping or DVD access only works if the target machine has access to an optical drive:

- Internal SATA CD/DVD drive
- USB external CD/DVD drive

If no optical drive is present, the playbook still installs packages, but ripping/playback tasks are not usable until a drive is attached.

## What the playbook installs

The playbook installs:

- `abcde`, `cdparanoia`, `cdrdao` (CLI ripping tools)
- `ffmpeg`, `flac`, `lame`, `vorbis-tools`, `id3v2` (encoding/metadata tools)
- `udisks2` and enables the service for media device handling

It also adds the Ansible user to `cdrom` and `audio` groups.

## Run it

From the repository root:

```bash
ansible-playbook -i linux/inventory/hosts-local.ini linux/playbooks/multimedia-setup.yml \
  --ask-become-pass
```

After group membership changes, log out and back in on the target host.

## Ripping CDs (headless)

> **SSH in first.** All commands below run on the server, not your local machine.
> ```bash
> ssh user@<SERVER_IP>
> ```

Insert the CD into the drive first, then run:

```bash
abcde -o flac -x
```

Flag breakdown:
- `-o flac` — output format. FLAC is the recommended default: it compresses audio without any quality loss (lossless), so your rip is bit-for-bit identical to the CD. Replace with `mp3`, `ogg`, or `opus` if you prefer a smaller lossy format
- `-x` — eject the disc automatically when ripping is complete

`abcde` will look up the disc in MusicBrainz, fetch track metadata, rip each track with `cdparanoia`, encode it, and tag the files automatically.

### Why FLAC?

FLAC is the best format for archiving CDs because:

- **Lossless** — no audio data is discarded, unlike MP3 or AAC
- **Compressed** — typically 50–60% the size of a raw WAV file
- **Free and open** — no patents or licensing fees
- **Widely compatible** — supported natively by Jellyfin, Navidrome, Plex, VLC, foobar2000, Rhythmbox, Strawberry, and most other open source media players
- **Future-proof** — you can always convert FLAC to MP3/AAC later; you cannot recover quality lost by ripping to MP3 directly

### Finding your CD rips

Files land in `~/Music/` by default, organised as:

```
~/Music/<Artist>/<Album>/<track_number>_<track_title>.flac
```

List what's been ripped:

```bash
ls ~/Music/
```

Or find all FLAC files recursively:

```bash
find ~/Music/ -type f -name "*.flac"
```

## Ripping DVDs (headless)

> **SSH in first.** All commands below run on the server, not your local machine.
> ```bash
> ssh user@<SERVER_IP>
> ```

DVD support requires first enabling the optional commented tasks in the playbook (`libdvd-pkg` and `dpkg-reconfigure libdvd-pkg`), then installing a ripping tool.

Install `dvdbackup` for a straight disc-to-folder copy:

```bash
sudo apt install dvdbackup
```

Insert the DVD and identify the drive (usually `/dev/sr0`):

```bash
ls /dev/sr*
```

Copy the full DVD structure to a folder:

```bash
dvdbackup -M -i /dev/sr0 -o ~/dvd-rips/
```

`-M` mirrors the full disc (VIDEO_TS and all extras). The output folder will be named after the disc title.

To transcode to a file instead of a raw copy, install HandBrakeCLI:

```bash
sudo apt install handbrake-cli
```

Encode the main title to MP4:

```bash
HandBrakeCLI -i /dev/sr0 -o ~/dvd-rips/output.mp4 --preset="Fast 1080p30"
```

### Output format options

Unlike CD ripping, the two tools have different approaches:

- **`dvdbackup`** — no format choice. Always outputs the raw `VIDEO_TS/` folder structure with `.VOB` files. Think of it as a 1:1 copy of the disc.
- **`HandBrakeCLI`** — full format control. Change the container and codec via the `-o` filename and `--encoder` flag:

  | Format | Example |
  |--------|---------|
  | MP4 (H.264) | `-o output.mp4 --encoder x264` |
  | MKV (H.264) | `-o output.mkv --encoder x264` |
  | MKV (H.265) | `-o output.mkv --encoder x265` (smaller file, slower encode) |

  H.265/MKV gives the best size-to-quality ratio for archiving. MP4/H.264 is the most compatible for playback on TVs and devices.

### Finding your DVD rips

**`dvdbackup`** — output is in the folder you specified with `-o`, named after the disc:

```
~/dvd-rips/<DISC_TITLE>/VIDEO_TS/
```

List ripped discs:

```bash
ls ~/dvd-rips/
```

Find the actual video files:

```bash
find ~/dvd-rips/ -name "*.VOB"
```

**`HandBrakeCLI`** — the file is exactly where you pointed `-o`:

```bash
ls ~/dvd-rips/
du -sh ~/dvd-rips/*
```

## Optional DVD support (currently commented out)

The playbook includes commented tasks for:

- `libdvd-pkg` installation
- `dpkg-reconfigure libdvd-pkg`

These are kept disabled by default and can be uncommented if needed.

Notes:

- DVD decryption support may depend on repository/component availability on your distro.
- In some environments, `dpkg-reconfigure libdvd-pkg` may build/download `libdvdcss2` and can take additional time.
