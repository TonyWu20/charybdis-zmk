# Research: Protecting the Charybdis from Abnormal Repeated Key Strokes

## Question

Can the firmware be modified to add a protection that prevents abnormal
repeated key strokes, possibly caused by a Bluetooth instability issue or
the aging of the mechanical switches?

## Answer

Yes. Three protection layers exist or were added for this:

1. Per-key scan debouncing (already in this ZMK fork, now configured for
   the Charybdis matrix): stops chatter from aging switches.
2. A key event guard (new, added to the ZMK fork, enabled for the
   Charybdis): drops duplicate and too-fast position events at the keymap.
3. A BLE position state resync (new, added to the ZMK fork, enabled for
   the central half): corrects stale key state when a BLE notification is
   lost, instead of letting the host auto-repeat a stuck key.

## Root cause analysis

### Switch aging (chatter)

An aging mechanical switch bounces: one physical press produces several
contact transitions. Each transition pair can register as an extra key
press, so one tap types "aa".

ZMK's GPIO matrix scanner already debounces every switch with a
press/release latch (`zmk_debounce`, integrated style: the switch must
stay closed for `debounce-press-ms` and open for `debounce-release-ms`
before the state latches). The default is 5 ms each, which is too short
for chattering switches.

### Bluetooth instability

This keyboard is a BLE split: the left half (peripheral) sends its key
state to the right half (central) as full-state GATT notifications. The
central is the only half that runs the keymap and the HID endpoint, so
all host key events flow through the central.

Failure modes:

- A lost key release notification leaves the central tracking that key
  as pressed. The host then auto-repeats that key while the user presses
  nothing. This is the "stuck repeated key" symptom (see zmkfirmware/zmk
  issue 2463).
- Out-of-order or duplicated notifications during link hiccups can
  register one press twice (see zmkfirmware/zmk issue 1633).
- On a full split disconnect, ZMK already releases every tracked
  peripheral position, so a complete link drop does not leave stuck
  keys. The gap is the lost notification without a disconnect.

## What was implemented

### Layer 1: kscan debounce tuning (this repo)

`boards/shields/charybdis/charybdis.dtsi`: the shared `kscan0` node now
sets:

- `debounce-press-ms = <10>` — a press latches after 10 ms of
  continuous closure.
- `debounce-release-ms = <15>` — a release latches after 15 ms of
  continuous opening.

This applies to both halves, because both include `charybdis.dtsi`.
Chatter bursts shorter than these windows never register as a key stroke.
Cost: up to 10/15 ms of press/release latency, not perceptible. Raise
the values toward 20-30 ms if double presses persist; lower them if the
keys feel slow.

### Layer 2: key event guard (ZMK fork change)

`zmk/app/src/keymap.c` + `zmk/app/Kconfig`:

- New Kconfig `CONFIG_ZMK_KEY_EVENT_GUARD` (default off) and
  `CONFIG_ZMK_KEY_EVENT_GUARD_MIN_INTERVAL_MS` (default 40).
- `zmk_keymap_position_state_changed` now runs a per-position latch:
  - a press on a position that is already pressed is dropped
    (duplicate press, e.g. from a lost-release resync or a BLE hiccup);
  - a press that arrives less than the configured interval after the
    previous press of the same position is dropped (chatter or link
    noise that survived the kscan debounce);
  - a release with no active press is dropped (phantom release).

The keymap only compiles on the central half in a BLE split, so this
guard filters every key event of both halves in one place. Sensor and
combo positions are virtual (position >= ZMK_KEYMAP_LEN) and are not
guarded, so the PMW3610 trackball and combos are unaffected. Intended
key repeat (holding a key, `&kpr`, host auto-repeat) is a behavior-level
mechanism and is not affected.

The 40 ms interval is safe: no human presses one key twice within 40 ms.
Enabled in `boards/shields/charybdis/charybdis_right.conf`:

- `CONFIG_ZMK_KEY_EVENT_GUARD=y`
- `CONFIG_ZMK_KEY_EVENT_GUARD_MIN_INTERVAL_MS=40`

### Layer 3: BLE position state resync (ZMK fork change)

`zmk/app/src/split/bluetooth/central.c` +
`zmk/app/src/split/bluetooth/Kconfig`:

- New Kconfig `CONFIG_ZMK_SPLIT_BLE_POS_RESYNC_MS` (default 0 = off).
- Every `ZMK_SPLIT_BLE_POS_RESYNC_MS` milliseconds the central re-reads
  the peripheral's `position_state` characteristic and compares it with
  its own copy. Divergence raises the same position state change events
  as a normal notification, so a key that the peripheral released but
  the central still tracks as pressed is released on the host.
- Enabled with `CONFIG_ZMK_SPLIT_BLE_POS_RESYNC_MS=500` in
  `boards/shields/charybdis/charybdis_right.conf`.

Cost: one 16-byte GATT read per peripheral every 500 ms. The read
callback and the notification handler both run in the Bluetooth work
queue context, so no new locking is needed.

## Verification

- Both halves build to `.uf2` (see Build commands below).
- `build/*/zephyr/.config` confirms: guard on the right, resync 500 on
  the right, and `debounce-press-ms`/`debounce-release-ms` in the
  compiled devicetree of both halves.
- The right half ELF contains the `key_event_guard` state array,
  `zmk_keymap_position_state_changed`, and the resync work items.
- All new options default off, so other ZMK users and other configs are
  unaffected.

## Build commands (local, verified)

```sh
export PATH="/nix/store/<cmake-3.31>/bin:/nix/store/<ninja>/bin:$HOME/nrfutil/zephyr-sdk-0.16.3/x86_64-unknown-elf:$PATH"
export ZEPHYR_SDK_INSTALL_BASE="$HOME/nrfutil/zephyr-sdk-0.16.3"
export ZEPHYR_TOOLCHAIN_VARIANT=zephyr

west build -p -s zmk/app -d build/right -b nice_nano_v2 \
  -- -DZMK_CONFIG="$PWD" -DSHIELD=charybdis_right \
     -DKEYMAP_FILE="$PWD/config/charybdis.keymap"
# same for charybdis_left
```

Note: the repo keeps `boards/` at the repo root and `config/` separate,
so the build points `ZMK_CONFIG` at the repo root (board root) and
passes the keymap explicitly.

## Applying the ZMK fork changes

The `zmk/` checkout is the TonyWu20 fork. The pin `b5ae616e` is the tip
of the fork branch `feat/pointers-move-scroll-smooth`. The guard and
resync changes live in that checkout as commit `6cffa804`:

- `zmk/app/Kconfig` — `ZMK_KEY_EVENT_GUARD`, `ZMK_KEY_EVENT_GUARD_MIN_INTERVAL_MS`
- `zmk/app/src/keymap.c` — the guard
- `zmk/app/src/split/bluetooth/Kconfig` — `ZMK_SPLIT_BLE_POS_RESYNC_MS`
- `zmk/app/src/split/bluetooth/central.c` — the resync

To make the config build pick them up:

1. Push the commit: `git -C zmk push TonyWu20 6cffa80437685abfb517436d33328c862d349c6a:refs/heads/feat/pointers-move-scroll-smooth`
2. In `config/west.yml`, set the `zmk` revision to
   `6cffa80437685abfb517436d33328c862d349c6a` (or the branch name `feat/pointers-move-scroll-smooth` to track it).
3. Run `west update` from the repo root.

Until the pin is bumped, a fresh `west update` resets the fork to
`b5ae616e` and the two `CONFIG_ZMK_KEY_EVENT_GUARD` /
`CONFIG_ZMK_SPLIT_BLE_POS_RESYNC_MS` lines in the right conf are inert
(Kconfig ignores unknown symbols).
The kscan debounce (Layer 1) works with the pinned fork as-is, since
`debounce-press-ms` / `debounce-release-ms` already exist in it.

## Residual risks

- A physically stuck switch (closed circuit, e.g. a short) still holds
  the key down; the guard throttles it to at most one press per 40 ms,
  and the host then auto-repeats normally. Inspect the switch.
- The resync only reconciles the two halves. A lost HID report to the
  host (USB/BT to the computer) is outside this design.
- Values to tune, in order: kscan debounce (10/15 ms), guard interval
  (40 ms), resync interval (500 ms).
