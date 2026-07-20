# Smart speaker

A voice-assistant role for the Trio C60's microphone array and speaker.
**C60-only** — it needs that microphone and speaker hardware, so it is
not offered on the TC8.

!!! warning "Placeholder role — no voice application"
    No voice application package exists. Selecting this role boots
    the device to a serviceable console (SSH enabled) with the audio
    stack configured — it does not listen, answer, or play anything on
    its own.

## What the role does

The role is backed by `poly-c60-profile-smart-speaker`, which ships the
**audio baseline**: ALSA utilities and the routing for the C60's codec,
microphone array, and speaker. A device flashed with this role:

- boots to a console (`multi-user.target`) with SSH enabled — the same
  serviceable state as the [developer role](developer.md), plus the
  audio stack ready to use;
- enables `poly-smart-speaker.service` **when that package is
  installed** — the hook a voice application package activates; no
  package providing it exists.

This makes the role a working base for experimenting with voice software
by hand: the microphone array and speaker are usable from the console
(`arecord`, `aplay`, `amixer`) out of the box.

## Configuration

The role is selected with `PROFILE=smart-speaker`; the `VOLUME_MASTER` /
`VOLUME_SPEAKER` mixer levels apply. Key semantics:
[configuration reference](../configure.md).
