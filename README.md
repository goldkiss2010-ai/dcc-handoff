![DCC Handoff — From computation to composition.](branding/key-visual.jpg)

# DCC Handoff

**From computation to composition.**

DCC Handoff is the umbrella project for moving externally computed state into digital content creation tools without baking the creative decisions into the simulation step.

The core idea is deliberately small:

```text
simulation / procedural computation / analysis
                    |
                    |  state
                    v
                   FLD1
                    |
          +---------+---------+
          |                   |
          v                   v
     AE Handoff         Fusion Handoff
     After Effects      Resolve / Fusion
```

The external process computes **what the particles are doing**.  
The DCC remains responsible for **how that state is viewed, timed, rendered, and composed**.

DCC Handoff is not a new scene format and is not a replacement for USD, Alembic, a DCC particle system, or a physics solver. It is a deliberately narrow handoff layer for cases where a compact state cache is enough.

## Implementations

| Project | Host | Status | Role |
|---|---|---|---|
| [AE Handoff](https://github.com/goldkiss2010-ai/ae-handoff) | Adobe After Effects | Preview | FLD1 reader + interpolation + 3D view/projection + particle rasterizer inside one AE effect |
| [Fusion Handoff](https://github.com/goldkiss2010-ai/fusion-handoff) | Blackmagic Fusion / DaVinci Resolve | Experimental | Fuse implementation of the same state-to-presentation model |
| [FLD1](https://github.com/goldkiss2010-ai/fld1) | Host-independent | Experimental contract | Compact particle-state interchange format and reference tooling |

The host implementations are intentionally separate repositories. This repository is the map and architectural boundary between them; it does not duplicate their binaries, installers, caches, or host-specific source.

A host-independent [FLD1 Asset Pack v01](https://github.com/goldkiss2010-ai/ae-handoff/releases/download/v1.15-preview.1/FLD1_Asset_Pack_v01.zip) provides the same six example fields for AE Handoff, Fusion Handoff, or another FLD1 reader. The package contains five 20K fields and a 1M-particle Vortex Ring.

## What is handed off

The current FLD1 `point3-pv` profile stores, per particle and per saved state:

```text
x y z  vx vy vz  visibility  scalar0
```

Particle identity is implicit in stable particle order. New normalized caches use saved-sample index

```text
q = 0, 1, 2, ...
```

as the state coordinate, with velocity expressed as `dx/dq`. Presentation time belongs to the receiving DCC.

This is the important separation:

```text
full simulation state S
        |
        | extract only what presentation needs
        v
display state V
        |
        | FLD1
        v
DCC presentation
```

Pressure fields, solver grids, constraints, temporary buffers, or other internal state do not need to cross the boundary unless the DCC genuinely needs that degree of freedom.

## Shared presentation model

AE Handoff and Fusion Handoff currently share the same conceptual pipeline:

```text
FLD1
  -> neighboring state samples
  -> cubic Hermite reconstruction
  -> object / view transform
  -> perspective projection
  -> Dot / Sprite / Image Dots presentation
  -> optional velocity streak
  -> RGBA image for the host compositor
```

The cache stays 3D. The current adapters perform the projection themselves and return an image to the host rather than creating one native DCC object per particle.

That choice is intentional: a cache containing hundreds of thousands or millions of particles should not require hundreds of thousands or millions of host objects.

## Presentation belongs to the DCC

The same state cache can be reinterpreted without rerunning the simulation:

- presentation speed and sample offset
- viewpoint, rotation, scale, dolly, and screen placement
- density selection
- particle size, color, opacity, depth cue, and velocity streak
- Sprite and Image Dots presentation
- **View Depth Split** using a screen-parallel `Focus Depth` plane

View Depth Split is a presentation operation, not a full Z-buffer. A Front and Back Handoff can read the same FLD1 cache, allowing an ordinary 2D layer to be composited between the two particle subsets.

## Design rules

1. **State, not frames.** Prefer handing off reconstructable state over pre-rendered image sequences.
2. **Keep presentation editable.** Timing, viewpoint, appearance, and compositing should remain in the DCC when practical.
3. **Keep the contract small.** Add an attribute only when the receiving DCC needs a degree of freedom that cannot or should not be reconstructed.
4. **Do not force host-native object counts.** A million particles should remain a compact cache plus a renderer, not a million layers or nodes.
5. **Separate format from implementation.** FLD1 is independent of AE Handoff and Fusion Handoff.
6. **Preserve semantics across hosts.** Host UI may differ; the meaning of time, interpolation, view transforms, and depth split should not drift casually.
7. **Measure before optimizing.** A simple CPU/Fuse path is acceptable until profiling shows where a host-specific acceleration path is justified.

## Repository boundaries

```text
dcc-handoff
  architecture, project map, cross-host design rules

fld1
  binary contract, reference reader/writer, validation

ae-handoff
  After Effects distribution, documentation, assets

fusion-handoff
  Fusion Fuse, documentation, verification
```

There is intentionally no common runtime dependency between the two host adapters today. Shared behavior is a **contract**, not a required shared library.

See [Architecture](docs/architecture.md) for the deeper model and [Host adapter guide](docs/host-adapter-guide.md) for the minimum questions a future adapter should answer.

## 日本語

DCC Handoffは、外部で計算した粒子状態をDCCへ渡し、**時間・視点・見た目・合成をDCC側に残す**ための上位プロジェクトです。

FLD1は「何を渡すか」、AE Handoff / Fusion Handoffは「各DCCでどう表示するか」を担当します。このリポジトリ自体は意図的に薄く保ち、各実装のコードや配布物を集約しません。

現在はAfter EffectsとFusion / DaVinci Resolveの2実装で、同じFLD1とほぼ同じ表示意味論が動作しています。新しいDCCへ移植する場合も、まずFLD1をネイティブオブジェクトへ展開するのではなく、状態を読み、再構成し、DCC内で観測・合成できる最小のアダプタを考えます。

## License

Documentation and original project material in this repository are released under the MIT License unless otherwise noted. Each implementation repository may have additional distribution or third-party terms; check that repository before redistributing binaries.
