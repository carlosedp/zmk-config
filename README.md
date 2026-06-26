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

## Timeless homerow mods

This section comes from Urob's [GitHub repo](https://github.com/urob/zmk-config#timeless-homerow-mods) on the topic. It is just reproduced here for convenience.

[Homerow mods](https://precondition.github.io/home-row-mods) (aka "HRMs") can be a game changer --
at least in theory. In practice, they require some finicky timing: In its most naive implementation,
in order to produce a "mod", they must be held _longer_ than `tapping-term-ms`. In order to produce
a "tap", they must be held _less_ than `tapping-term-ms`. This requires very consistent typing
speeds that, alas, I do not possess. Hence my quest for a "timer-less" HRM setup.

After months of tweaking, I eventually ended up with an HRM setup that is essentially timer-less,
resulting in virtually no misfires.[^1] Yet it provides a fluent typing experience with mostly no
delays.

One way to make HRMs effectively timer-less is to set `tapping-term-ms` to an extremely large value,
say 5 seconds. This removes the need for quick timing decisions, but it introduces two issues: (1)
To trigger a mod, you'd need to hold the HRM keys for what feels like an eternity. (2) During normal
typing, there's a noticeable delay between pressing a key and seeing it appear on the screen.[^2] To
address these, I use positive and negative exceptions that short-circuit the tapping term in most
scenarios.

- Specifically, to address the activation delay, I use ZMK's `balanced` flavor, which produces a
  "hold" if another key is both pressed and released within the tapping-term. Because that's exactly
  what I normally do with HRMs, there's virtually never a need to wait past my long tapping term (see
  below for two exceptions).
- To address the typing delay, I use ZMK's `require-prior-idle-ms` property,
  which immediately resolves an HRM as a "tap" when it's pressed shortly _after_
  another key has been tapped. This all but completely eliminates the delay.

This is great but there are still a few rough edges:

- When rolling keys, I sometimes unintentionally end up with "nested" key
  sequences: `key1` down, `key2` down and up, `key1` up. Because of the
  `balanced` flavor, this would falsely register `key1` as a mod. As a remedy,
  I use ZMK's "positional hold-tap" feature to force HRMs to always resolve as
  "tap" when the _next_ key is on the same side of the keyboard. Problem solved.
- ... or at least almost. By default, positional-hold-tap performs the
  positional check when the next key is _pressed_. This is not ideal, because it
  prevents combining multiple modifiers on the same hand. To fix this, I use the
  `hold-trigger-on-release` setting, which delays the positional-hold-tap
  decision until the next key's _release_. With this, mods can be combined when
  held while positional hold-tap continues to work as expected when keys are
  tapped.
- So far, nothing of the configuration depends on the duration of
  `tapping-term-ms`. In practice, there are two reasons why I don't set it to
  infinity:
  1. Sometimes, in rare circumstances, I want to combine a mod with a alpha-key
     _on the same hand_ (e.g., when using the mouse with the other hand). My
     positional hold-tap configuration prevents this _within_ the tapping term.
     By setting the tapping term to something large but not crazy large (I use
     280ms), I can still use same-hand `mod` + `alpha` shortcuts by holding the
     mod for just a little while before tapping the alpha-key.
  2. Sometimes, I want to press a modifier without another key (e.g., on
     Windows, tapping `Win` opens the search menu). Because the `balanced`
     flavour only kicks in when another key is pressed, this also requires
     waiting past `tapping-term-ms`.
- Finally, it is worth noting that this setup works best in combination with a
  dedicated shift for capitalization during normal typing (I like sticky-shift
  on a home-thumb). This is because shifting alphas is the one scenario where
  pressing a mod may conflict with `require-prior-idle-ms`, which may result in
  false negatives for fast typers.

Here's my configuration (I use a bunch of
[helper macros](https://github.com/urob/zmk-helpers) to simplify the syntax, but
they are not necessary):

```C++
#include "zmk-helpers/key-labels/36.h"                                      // Source key-labels.
#define KEYS_L LT0 LT1 LT2 LT3 LT4 LM0 LM1 LM2 LM3 LM4 LB0 LB1 LB2 LB3 LB4  // Left-hand keys.
#define KEYS_R RT0 RT1 RT2 RT3 RT4 RM0 RM1 RM2 RM3 RM4 RB0 RB1 RB2 RB3 RB4  // Right-hand keys.
#define THUMBS LH2 LH1 LH0 RH0 RH1 RH2                                      // Thumb keys.

/* Left-hand HRMs. */
ZMK_HOLD_TAP(hml,
    flavor = "balanced";
    tapping-term-ms = <280>;
    quick-tap-ms = <175>;
    require-prior-idle-ms = <150>;
    bindings = <&kp>, <&kp>;
    hold-trigger-key-positions = <KEYS_R THUMBS>;
    hold-trigger-on-release;
)

/* Right-hand HRMs. */
ZMK_HOLD_TAP(hmr,
    flavor = "balanced";
    tapping-term-ms = <280>;
    quick-tap-ms = <175>;
    require-prior-idle-ms = <150>;
    bindings = <&kp>, <&kp>;
    hold-trigger-key-positions = <KEYS_L THUMBS>;
    hold-trigger-on-release;
)
```

### Troubleshooting

Hopefully, the above configuration "just works". If it doesn't, here's a few
smaller (and larger) things to try.

- **Noticeable delay when tapping HRMs:** Increase `require-prior-idle-ms`. As a
  rule of thumb, you want to set it to at least `10500/x` where `x` is your
  (relaxed) WPM for English prose.[^3]
- **False negatives (same-hand):** Reduce `tapping-term-ms` (or disable
  `hold-trigger-key-positions`)
- **False negatives (cross-hand):** Reduce `require-prior-idle-ms` (or set
  flavor to `hold-preferred` -- to continue using `hold-trigger-on-release`, you
  must apply this
  [patch](https://github.com/celejewski/zmk/commit/d7a8482712d87963e59b74238667346221199293)
  to ZMK
- **False positives (same-hand):** Increase `tapping-term-ms`
- **False positives (cross-hand):** Increase `require-prior-idle-ms` (or set
  flavor to `tap-preferred`, which requires holding HRMs past tapping term to
  activate)
