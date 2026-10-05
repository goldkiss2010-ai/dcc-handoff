# Host adapter guide

A new DCC Handoff adapter should answer these questions before adding host-specific features.

## Required decisions

**Input contract** — Which FLD1 profile is accepted? How are invalid headers, unsupported strides, missing files, and large particle counts handled?

**State coordinate** — How does host time map to FLD1 sample coordinate `q`? Can the user drive `q` directly?

**Interpolation** — How are position and velocity reconstructed between saved samples? Are endpoint and out-of-range rules explicit?

**Coordinate convention** — Where is the FLD1 right-handed XYZ / Z-up convention converted, if conversion is needed?

**View model** — Does the adapter use its own lightweight view/projection, or hand points to the host's native 3D system? The choice should be explicit.

**Presentation modes** — Which of Dot, Sprite, Image Dots, velocity streak, density selection, opacity, and depth cue are supported?

**Depth split** — If implemented, `Focus Depth` means camera-space Z and the split plane remains parallel to the image plane.

**Host output** — Does the adapter return RGBA, native geometry, a point cloud, or another host object? Avoid accidental semantic differences between hosts.

## Compatibility principle

Host controls do not need identical names or layout, but equivalent controls should preserve equivalent meaning.

A new adapter should first reproduce a small common path:

```text
FLD1 -> sample -> interpolate -> transform -> project -> dots -> host output
```

Only after that path is verified should host-specific conveniences be added.

## Performance principle

Do not assume that host-native objects or GPU execution are automatically faster. Benchmark the actual path at representative particle counts, resolution, frame rate, particle size, and streak/sprite settings.

The current project has intentionally used simple implementations first so that semantics remain inspectable while the architecture is still evolving.
