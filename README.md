# ZMK config: Eyelash Sofle

My personal ZMK config for the Eyelash Sofle, with Swedish symbol and number layers. The OS layout must be set to Swedish.

![Eyelash Sofle Keymap](keymap-drawer/eyelash_sofle.svg)

- Keymap: `config/eyelash_sofle.keymap`
- Builds: GitHub Actions builds both the dongle and standalone (left central) firmware from `build.yaml`. To build locally, run `./build_local.sh build`.
- With the dongle, flash the dongle first, then the left half, then the right half, and power on the left half before the right.
