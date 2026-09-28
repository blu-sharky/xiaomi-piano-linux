# GLM5.3 Report — native display + GPU + touch: from broken CI image to device-validated boot

Date: 2026-09-28. Scope: debian-piano / xiaomi-piano-linux / piano-firmware, session covering the blu-sharky org migration, the first CI image of the milestone-GPU stack, and the on-device debugging that followed. Every item marked *verified* was reproduced and measured on the device unless stated otherwise.

## Outcome

The final image boots straight to the native panel with the Adreno 830 under freedreno, GNOME responds to touch, and every fix below is merged (or pending merge with checks green) in the repos:

- Panel: msm-kms, NT36532 DSC, 3200x2136, stage 5 completes in ~1 s at boot (PLL transient waits included).
- GPU: `renderD128` driven by piano-mesa 26.1.6 `+piano1` under freedreno; the desktop is hardware-accelerated.
- Touch: THP frames flow under the live DSC display after the TDDI rebind; 105,528 bytes of events in a 10 s capture; GNOME responds (user-confirmed).
- Access: every build generates a fresh random ed25519 key, published as a small artifact (`piano-access-key-<sha>`) plus a public-key fingerprint in the run summary.

## Root causes and fixes

| # | Symptom | Root cause | Fix | Confidence |
|---|---|---|---|---|
| 1 | CI build failed `Unknown option: --mesa-dir` | debian-piano PR #21 pushed 61 s before the umbrella PR #20 carrying the option landed | Re-run after the merge; race, not code | Certain (log timestamps) |
| 2 | First image: no GPU, no native display | The image dtbo was built from `dtbo-piano-rootfs.dts`, whose include chain has no display/GPU nodes; the service cannot pass stage 1 | xiaomi-piano-linux #22/#23: dtbo source = `dtbo-piano-display.dts` | Certain (include chain, build log) |
| 3 | Any CI image SSH-unreachable | The ephemeral access key never left the runner ("deliberately not published"); a fixed `PIANO_SSH_PUBLIC_KEY` variable pointed at a key whose private half was lost | debian-piano #23: delete the variable (per-build random keys), upload the keypair artifact + summary fingerprint | Certain (used live for this session) |
| 4 | `piano-display` SEGV loop, `session-N.scope` spam (90+ sessions) | The overlay still forced `LIBGL_ALWAYS_SOFTWARE=1`/`GALLIUM_DRIVER=llvmpipe`; forced llvmpipe-on-GBM dies in mutter's first frame on the msm card. The device-validated session (f68bd290) had commented these out **on the device only** — the tweak never reached the repo | debian-piano #27: drop the forcing; #28: native display on by default (opt-out), matching the validated semantics | Certain (validated-state transcript + live boot) |
| 5 | `stage 5` FAIL: `DSI output never enabled` | The first DSI PLL lock after a cold dispcc enable fails and self-heals (kit log: 5 failures, output enabled ~80 s in; later boots ~2 s) | debian-piano #25: enable wait 30 s → 150 s, service timeout 180 s → 240 s | High (kit + device logs) |
| 6 | Touch dead after the display takeover (0 bytes from `/dev/input/event0`) | The panel takeover resets the TDDI touch IC; the THP stream wedges (reference capture never completes). Userspace restart is not enough | debian-piano #29: `NVT-ts-spi` unbind/bind in `touch-start`, `piano-touch` ordered after `piano-display` | Certain (0 → 105 KB measured; GNOME responds) |

## Wrong turns (kept for the record)

- #22 shipped a literal `+` at the start of the edited line (exit 127); fixed by #23. The lesson: `bash -n` belongs in the local checklist, not just the target compiler.
- #26 quarantined `renderD128` (root-only) on the theory that unimplemented Gen8 Mesa paths crashed the compositor. Wrong layer: freedreno drives the A830 fine once the llvmpipe forcing is gone. Withdrawn before merge; no image ever carried it.

## Notable mechanics

- **Where the "it used to work" went**: session f68bd290 validated the full stack on the device but left two state changes uncommitted — the commented-out llvmpipe forcing and the (functionally equivalent) default-on display service. Every artifact built from the repos therefore booted into the broken path, including images built from the very components that had been validated. The repo was one commit away from the validated state; #27/#28 are that commit.
- **TDDI ordering invariant**: the touch reference must be captured under the *final* display state. The original ordering (touch before the takeover) is unsatisfiable by design: the panel reset invalidates whatever the touch side established.
- **NCM/SSH is best-effort at boot** (`rootfs-init` warns and continues), then `piano-usb.service` re-establishes the gadget after switch_root; 10.42.0.2 is reachable in both phases.

## Remaining

- #29 merge + reflash to make the touch fix persistent (the on-device manual fix is session-local).
- `docs/native-display-gpu.md` §5 stage table and the "not yet validated" status predate this session's results; refresh when the milestone is re-baselined.
- GPU bring-up depth (devfreq, GX GDSC recovery, seamless rate switching) remains open per the original design notes.
