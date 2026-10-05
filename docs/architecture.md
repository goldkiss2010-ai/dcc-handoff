# Architecture

## 1. Two systems, one boundary

DCC Handoff treats simulation and presentation as different systems.

The producer may be Python, C++, a numerical solver, procedural geometry, image analysis, or another source. Its job is to produce state. The DCC adapter's job is to reconstruct a view of that state inside a creative environment.

This avoids coupling the producer to a particular host API.

```text
Producer                       DCC adapter
------------------------       --------------------------------
state evolution                sample selection
simulation / analysis    ->    interpolation
procedural generation          view transform
                               projection
                               rasterization
                               compositing
```

## 2. FLD1 is the state boundary

The current base profile is intentionally small:

```text
position      x y z
velocity      vx vy vz
visibility    visibility
free scalar   scalar0
```

The format does not attempt to preserve the complete solver state. It preserves a presentation-relevant subspace.

If the full simulation state is `S`, the exported state can be viewed as a projection or extraction

```text
E : S -> V
```

where `V` is the state needed by the receiving renderer. DCC Handoff is concerned with the contract on `V`, not with reproducing the entire dynamics inside the DCC.

## 3. Time is not baked into the cache

For normalized caches, the saved state coordinate is sample index `q`.

```text
q = SampleOffset + elapsed_seconds * SamplesPerSecond
```

The DCC chooses the mapping from presentation time to `q`. Setting SamplesPerSecond to zero allows SampleOffset itself to become the authored state coordinate.

This is why a cache can be retimed without regenerating the underlying simulation.

## 4. Reconstruction

With position and velocity available at neighboring saved states, current adapters use cubic Hermite reconstruction. The format carries enough local state for a receiver to reconstruct smooth motion between saved samples.

The interpolation policy belongs to the receiver contract. FLD1 stores state; it does not store rendered in-between frames.

## 5. View and projection

The current adapters transform reconstructed 3D positions into a camera/view space and apply perspective projection. The host receives an RGBA result rather than a native 3D point cloud.

This is a pragmatic architecture for compositing hosts:

- stable cost model
- no explosion of host objects
- same cache can be viewed differently
- host-native effects can continue after the Handoff render

A future native-3D adapter is possible, but it is a different adapter strategy rather than a requirement of FLD1.

## 6. View Depth Split

View Depth Split compares transformed particle camera-space Z against one scalar `Focus Depth`.

```text
Front: camera_z <= FocusDepth
Back:  camera_z >= FocusDepth
```

The split plane is always parallel to the image / sensor plane. It does not rotate with the particle object.

This deliberately solves a compositing problem rather than a geometry problem: two Handoff renders can surround an ordinary 2D layer.

## 7. What should become shared code?

Nothing needs to become shared code merely because two adapters implement the same behavior.

A shared library becomes useful only when it reduces maintenance without making host integration harder. Until then, the cross-host contract is the important shared artifact.

That keeps each adapter deployable in the simplest form appropriate to its host: an AE plugin on one side, a Fuse on the other.
