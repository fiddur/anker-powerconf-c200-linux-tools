# anker-powerconf-c200-linux

Linux CLI tools for the Anker PowerConf C200 webcam.

This project reverse engineers the vendor camera controls used by the official Anker app and exposes them as a small command-line utility for Linux.

## Status

Confirmed on an Anker PowerConf C200 with USB VID:PID `291a:3369`.

The current binary is:

```sh
./build/anker-powerconf-c200-linux-tools
```

## Features

- Read and set the vendor FOV presets on Linux
- Access additional vendor-only controls that are not exposed as normal V4L2 controls
- Access standard V4L2/UVC controls through the same CLI

Currently exposed controls:

- `fov`
- `hdr`
- `horizontal_flip` (only applies while streaming, see notes below)
- `vertical_screen` (not implemented on all firmware, see notes below)
- `anti_flicker` (asymmetric read-back, see notes below)
- `brightness`
- `contrast`
- `saturation`
- `hue`
- `white_balance_automatic`
- `gamma`
- `power_line_frequency`
- `white_balance_temperature`
- `sharpness`
- `auto_exposure`
- `exposure_time_absolute`
- `pan_absolute`
- `tilt_absolute`
- `focus_absolute`
- `focus_automatic_continuous`
- `zoom_absolute`

## Reverse Engineering Notes

The C200 uses a UVC Extension Unit for at least some vendor-specific camera controls.

Confirmed FOV control path:

- Unit ID: `0x06`
- Selector: `0x10`
- Linux `GET_INFO`: `0x03`
- Linux `GET_LEN`: `60`

FOV payload layout:

- bytes `0..1`: `0x0100` little-endian when setting
- bytes `2..3`: FOV value as `uint16_le`
- byte `4`: `0x00`

Known FOV presets:

- `65`: narrow
- `78`: medium
- `95`: wide

Other confirmed vendor selectors currently wired into the CLI:

- `0x0a`: `vertical_screen`
- `0x11`: `horizontal_flip`
- `0x12`: `anti_flicker`
- `0x13`: `hdr`

### Firmware observations (291a:3369, firmware as shipped 2026-09)

Measured on a single unit, so some of this may be firmware specific.

- `fov` and `zoom_absolute` are two interfaces onto the same control.
  Selecting a preset moves `zoom_absolute` to `100` (95 deg), `175` (78 deg)
  or `295` (65 deg). Changing `zoom_absolute` afterwards widens or narrows the
  image while `fov` keeps reporting the last preset written, so the `fov`
  read-back goes stale. `zoom_absolute` scales by area rather than linearly,
  so the linear crop is `sqrt(zoom / 100)`; that is why `295` lands on 65 deg.
- `anti_flicker` is asymmetric. It takes `0`, `1` and `2` on write but reads
  back `0`, `50` and `60` respectively. Anything above `2` clamps to `60`.
- `vertical_screen` (selector `0x0a`) is not implemented on this firmware.
  `GET_LEN` fails for that selector and the extension unit's `bmControls`
  bitmap (`0x00 0xf8 0x27`) does not advertise it.
- `hdr` works, but its read-back does not. Writes take effect: toggling it
  while streaming moves mean frame luminance by roughly 20 levels out of 255
  and visibly lifts shadow detail, and the change reverts when it is switched
  back. `GET_CUR` nevertheless always returns `1`, both idle and while
  streaming, so the CLI cannot report the real state.
- `horizontal_flip` only takes effect while the camera is streaming. Toggled
  against a running stream it mirrors the image correctly, confirmed by
  comparing captured frames against their own mirror. Writes made while the
  camera is idle are silently dropped: the register reads back `0` and the
  setting does not carry into the next stream. `hdr`, by contrast, can be set
  either way.
- Selectors `0x0d`, `0x0f` and `0x16` are live (`GET_LEN` 60, `GET_INFO` 3)
  and currently read `1`, but their meaning is unknown.
- The extension unit GUID as stored in the descriptor is
  `41769ea2-04de-e347-8b2b-f4341aff003b`, which is the UVC H.264 XU GUID with
  its first three fields byte-reversed. Anker does the same on the C310.

## Releases

Prebuilt binaries are published on the GitHub Releases page:

- [GitHub Releases](https://github.com/erans/anker-powerconf-c200-linux-tools/releases)

Download the release tarball for your architecture if you do not want to build from source.

## Build

```sh
make
```

Run tests:

```sh
make test
```

## Usage

List controls:

```sh
./build/anker-powerconf-c200-linux-tools list-controls
```

Read a control:

```sh
./build/anker-powerconf-c200-linux-tools get fov
./build/anker-powerconf-c200-linux-tools get hdr
./build/anker-powerconf-c200-linux-tools get brightness
```

Set a control:

```sh
./build/anker-powerconf-c200-linux-tools set fov narrow
./build/anker-powerconf-c200-linux-tools set fov wide
./build/anker-powerconf-c200-linux-tools set hdr on
./build/anker-powerconf-c200-linux-tools set horizontal_flip off
./build/anker-powerconf-c200-linux-tools set brightness 60
```

Use a different device node:

```sh
./build/anker-powerconf-c200-linux-tools --device /dev/video0 get fov
```

Compatibility aliases are also available:

```sh
./build/anker-powerconf-c200-linux-tools get-fov
./build/anker-powerconf-c200-linux-tools set-fov narrow
```

## Notes

- The standard `zoom_absolute` control is separate from the vendor FOV preset control.
- On this machine, `/dev/video0` is the node that accepts the camera extension-unit queries.
- The vendor macOS code uses a 64-byte staging buffer for FOV, but Linux reports the real control length as `60`, and the Linux ioctl path must use the reported length.
