---
title: "RasterLab: Building an Image Editor with Claude Code"
date: 2026-04-23
tags: [fedora, rust, claude-code, image-editing, ai, experiment]
draft: false
---

The question was simple enough: *How good of an image editor can you build with $20 worth of
Claude Code Pro subscription?*

The answer, after one month and roughly that budget, is: surprisingly good, occasionally wrong
about performance, and frustratingly confident about things it hadn't measured.

RasterLab is a non-destructive RAW image editor written in Rust, built almost entirely by Claude
Code. Not prototyped by it, not scaffolded by it --- actually built by it, with me driving
direction and reviewing the output. One month, four weekly usage blocks, one image editor.

## What Got Built

The feature list grew faster than expected. By end of week one: a working non-destructive pipeline
with a background render thread, undo/redo, histogram, crop, rotate, and a custom binary project
format (`.rlab`) with Blake3 integrity hashes baked in from the start.

Weeks two and three piled on: virtual copies (multiple independent edit stacks per image), a spot
heal tool, noise reduction, clarity/texture, local adjustments, split before/after view, panorama
stitching, focus stacking, HDR merge, and support for basically every RAW format the rawler crate
handles.

Week four added a full photo library --- import, EXIF indexing, ratings, keywords, collections,
thumbnail grid, the works. Plus adaptive Reed-Solomon error correction on `.rlab` files, so bitrot
is survivable up to the parity budget.

The supported-ops table in the README runs to 35 entries. That's more tools than I expected to
reach.

## The Architecture Claude Got Right

The overall design landed well on the first pass: a non-destructive op stack, a background render
thread that owns all the pixel work, `Arc`-based image sharing between pipeline stages, and a
generation counter to prevent stale writes from landing after newer results. The step cache ---
which avoids re-running ops 0 through N-1 when you're adjusting op N --- was designed correctly
from the start.

The downsampled preview path (25% resolution during slider drags, ~16&times; fewer pixels) was
added in week one after initial render latency was sitting at 800ms per edit. Claude was confident
the cache alone would make sliders feel responsive. It was wrong. The cache helps with undo and
final commit, but dragging a levels slider at 800ms per frame is not real-time preview. The
downsampled path got the round-trip down to something workable.

Rayon parallelism showed up in all the right places. The project notes have a rule now about large
fold accumulators --- learned the hard way when a histogram implementation was moving an 8 KiB
accumulator by value once per pixel. An 8 KiB accumulator &times; 35 million pixels is roughly
143 GB of stack traffic, turning a 5ms operation into 400ms. Once you understand why that's bad
you can't unsee it.

## The `.rlab` Format

The native project format ended up being one of the more interesting design decisions. It's a
chunked binary container: original source bytes verbatim, the full edit stack, metadata, an
optional thumbnail. Every chunk carries a Blake3 hash. The whole file is sealed with a trailing
hash. Any mutation is detected on open.

That was week one. Week four added a Reed-Solomon parity chunk --- ~10% overhead, written twice
for small files, surviving corruption anywhere up to the parity budget. `verify_and_repair`
reconstructs damaged chunks in place without rewriting the whole file.

This is probably overkill for a personal image editor. It's also the kind of thing that's nearly
impossible to retrofit, so building it in early was the right call.

## What Claude Got Wrong

Consistently confident about performance before measuring. The cache would have made things fast
--- eventually --- but 800ms is 800ms. The downsampled preview fixed what the cache couldn't.

The `Arc<Image>` Deref coercion explanation was needed twice. Small thing, but you notice
patterns.

The initial noise reduction had its detail-preservation mask computed from the noisy input instead
of the denoised output. This caused the mask to classify noise as edge detail and blend it back
in, making noise reduction nearly invisible at default settings. That one took a minute to
diagnose.

## Where Things Landed

The plugin API exists and has an example. The arbitrary rotation is still slower than it should be
because bilinear interpolation has terrible cache locality. The mouse wheel scroll is still a
little jumpy.

Everything else works. The pipeline is fast enough that I stopped complaining about it, which is
close enough to satisfied. The library covers what a digital asset manager (DAM) should cover:
thumbnail grids, EXIF search, collections, ratings, import sessions, the full set.

For $20[^1] and one month, with Claude writing the vast majority of the code, the result is a
usable RAW editor with a real architecture --- not a toy. That's the honest answer to the original
question.

Code is at [github.com/tasleson/rasterlab](https://github.com/tasleson/rasterlab). It's alpha ---
treat it that way.

[^1]: I wasn't diligent enough to get the full $20 worth. That's Anthropic's loss; it diminishes
what this could have been.
