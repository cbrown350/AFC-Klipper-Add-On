# Elegoo Centauri Carbon + CANVAS vendor baseline

Snapshot of the AFC modules shipped by OpenCentauri COSMOS on this machine, kept
so the local patches can be re-derived and re-applied after a firmware update.

Device: Elegoo Centauri Carbon (upgraded to CANVAS), OpenCentauri COSMOS
firmware (Klipper/Kalico), reachable at `http://192.168.50.26`, SSH as `root`.

## Why this directory exists

The device does **not** run upstream ArmoredTurtle AFC. It runs an Elegoo vendor
fork: `AFC_VERSION="1.2.1"` vs upstream `1.2.0`, and it adds CANVAS-specific
modules (`AFC_canvas.py`, `AFC_canvas_lane.py`) that do not exist upstream.
So a fix that touches CANVAS-only code cannot be a normal PR against
ArmoredTurtle — it has to live here.

Files
- `AFC.py.vendor-orig` — vendor AFC.py exactly as shipped (md5 `9d37a9657228c6f2207c40efc7fc9d71`)
- `AFC.py.deployed` — vendor AFC.py with the local patches applied (md5 `4139199dca7b808c0f5a389650841640`)
- `AFC_lane.py.vendor-orig` — vendor AFC_lane.py, unmodified (no local patch needed)
- `AFC_canvas_lane.py.vendor-orig` — vendor CANVAS lane module, for reference

## How the patches are deployed

The root filesystem is a read-only squashfs, so vendor modules cannot be edited in
place. An init script bind-mounts the patched file over the vendor path at boot:

- `/data/afc-patch/AFC.py` — patched module (writable `/data` partition)
- `/etc/init.d/afc-patch` — bind-mounts it before Klipper starts (`S93afc-patch`)

Revert with `/etc/init.d/afc-patch stop`, or delete the init script and reboot.

**Gotcha:** `mount --bind` binds the *inode*. Replacing `/data/afc-patch/AFC.py`
with `mv` or `rm` leaves the mount serving the old content — the file on disk
looks updated while Klipper keeps running the previous code. Always edit in
place with `cp`. The init script detects this and remounts, but only if you run
it.

## Local patches applied to `AFC.py.deployed`

1. **`LANE_UNLOAD` warning wording** (~line 1346). Vendor said
   `"Unloading is not supported on {cur_lane.unit}"`. Two problems: `LANE_UNLOAD`
   ejects (the state is already `State.EJECTING_LANE` = "Ejecting"), and
   `cur_lane.unit` for CANVAS is the whole 4-lane box — literally always spelled
   `CANVAS_1` — so one lane's limitation read as if the entire unit were
   affected. Now: `"Ejecting is not supported on lane {cur_lane.name}"`.

2. **Two silent `TOOL_UNLOAD` no-ops** — the same two bugs fixed in the
   upstream branch of this repo (see the branch `fix/tool-unload-silent-noop`
   and its commit message). The fix hunks are identical; only the example lane
   name in the error text differs (`CANVAS_1` vs `lane1`).

Patches 2 are upstream-eligible. Patch 1 is vendor-only because
`supports_lane_unload` does not exist upstream at all.

## Unrelated but worth knowing

`PREP` reads lane `map` and `runout_lane` from `AFC.var.unit` and **overrides the
config file** (`AFC_prep.py`). The save/load pair means an infinite-spool swap
persists across prints, and if that var file is missing at startup, PREP logs
`AFC.var.unit file not found`, continues with config defaults, and `runout_lane`
defaults to `None` — silently removing the failover setting. `runout_lane` is the
setting that decides whether a runout pauses for manual intervention or switches
lanes.
