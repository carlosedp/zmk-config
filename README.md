# ZMK Firmware configuration for my keyboards

This repo contains the ZMK firmware configuration for my keyboards, including the Anywhy Flake and the Eyelash Sofle. It includes custom keymaps, build configurations, and shield overlays for these keyboards.

## Anywhy Flake

![Flake Picture](./docs/flake.jpg)

This is the ZMK firmware for the [Anywhy Flake](https://github.com/anywhy-io/flake) keyboard with my custom keymap.

![Anywhy Flake Keymap](keymap-drawer/anywhy_flake.svg)

The original keymap is available at <https://github.com/anywhy-io/flake-zmk-module> and the Flake keyboard project is at <https://github.com/anywhy-io/flake>.

## Eyelash Sofle

![Eyelash Sofle Picture](./docs/sofle.png)

This is the ZMK firmware for the Eyelash Sofle keyboard with my custom keymap. The build configuration includes both the central dongle and standalone versions of the keyboard.

![Eyelash Sofle Keymap](keymap-drawer/eyelash_sofle.svg)

## Building and Flashing

All changes pushed to the repository trigger a build in GitHub Actions. You can download the latest firmware artifacts from the [Actions tab](https://github.com/carlosedp/zmk-sofle/actions/workflows/build.yml).

Download the generated `.zip` file and extract the three `.bin` files. One is the settings reset firmware that can be used to reset the keyboard settings if needed (same for both sides). The other two files are the left and right keyboard firmwares.

Press reset switch on the back of the keyboard to reset.

Double clicking it will enter the Bootloader state. A drive will be shown in your computer where you can copy the corresponding `.bin` or `.uf2` file to flash the keyboard. Do this for both sides.

If changing just the keymap or settings, flashing only one side (usually the left) is sufficient.

## Building Locally

### Quick Start with Build Script

The easiest way to build the firmware locally is using the provided build script:

```sh
# Clone the repository
git clone https://github.com/carlosedp/zmk-config.git
cd zmk-config

# Build all firmware (auto-initializes if needed)
./build_local.sh build

# List available build targets
./build_local.sh list

# And build specific targets
./build_local.sh build [target_name]
```

The firmware files will be located at the `./build` directory.

#### Build Script Features

- **Auto-Initialization**: Automatically sets up west workspace and fetches dependencies
- **Dynamic Configuration**: Reads build targets from `build.yaml` and dependencies from `config/west.yml`
- **Incremental Builds**: Fast rebuilds with `INCREMENTAL=true ./build_local.sh build [target_name]`
- **Smart Cleaning**:
  - `./build_local.sh clean` - Remove build artifacts only
  - `./build_local.sh clean_all` - Remove all dependencies and build artifacts
- **Artifact Management**: `./build_local.sh copy [destination]` - Copy all firmware files
- **List Targets**: `./build_local.sh list` - Show all available build targets

#### Environment Variables

- `RUNTIME` - Container runtime (default: `podman`, can use `docker`)
- `ZMK_IMAGE` - ZMK build image (default: `docker.io/zmkfirmware/zmk-build-arm:4.1-branch`)
- `BUILD_CONFIG` - Build configuration file (default: `build.yaml`)
- `INCREMENTAL` - Skip pristine builds for faster rebuilds (default: `false`)

**Examples:**

```sh
# Use Docker instead of Podman
RUNTIME=docker ./build_local.sh build

# Faster incremental rebuild during development
INCREMENTAL=true ./build_local.sh build_left

# Copy artifacts to custom location
./build_local.sh copy ~/Desktop/firmware

# Use custom build configuration
BUILD_CONFIG=custom.yaml ./build_local.sh build
```

### Pairing with the Dongle

When using the central dongle, the battery widget assigns the battery indicators from left to right, based on the sequence in which the keyboard halves are paired to the dongle.

For split keyboards, it is essential to pair the left half first after flashing the dongle, followed by the right half. This ensures the correct mapping of battery status indicators and avoids swapped displays in the widget.

The recommended procedure is as follows:

1. Switch off both keyboard halves.
2. Flash the dongle
3. Disconnect the dongle
4. Flash the left half
5. Flash the right half
6. Reconnect the dongle
7. Switch on the left half and wait until the battery indicator appears on dongle
8. Switch on the right half

### Manual Build (Advanced)

For manual control over the build process:

```sh
git clone https://github.com/carlosedp/zmk-sofle.git
cd zmk-sofle

podman run -it --rm --security-opt label=disable --workdir /zmk -v $(pwd):/zmk -v /tmp:/temp docker.io/zmkfirmware/zmk-build-arm:4.1-branch /bin/bash

west init -l config
west update

export "CMAKE_PREFIX_PATH=/zmk/zephyr:$CMAKE_PREFIX_PATH"

# Build left half
west build -d ./build/left -p \
  -b "eyelash_sofle_left" \
  -s /zmk/zmk/app \
  -S studio-rpc-usb-uart \
  -- -DSHIELD="nice_view_gem" -DCONFIG_ZMK_STUDIO=y -DCONFIG_ZMK_STUDIO_LOCKING=n \
  -DZMK_CONFIG="/zmk/config"

# Build right half
west build -d ./build/right -p \
  -b "eyelash_sofle_right" \
  -s /zmk/zmk/app \
  -- -DSHIELD="nice_view_gem" \
  -DZMK_CONFIG="/zmk/config"

# Settings Reset
west build -d ./build/settings_reset -p \
  -b "nice_nano_v2" \
  -s /zmk/zmk/app \
  -- -DSHIELD="settings_reset" \
  -DZMK_CONFIG="/zmk/config"
```

To flash the firmware, follow the same procedure as described in the "Building and Flashing" section above, using the generated `.uf2` files.
