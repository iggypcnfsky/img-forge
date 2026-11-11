---
name: img-forge
description: Generate images with fal.ai models from the command line, then place them on a Figma canvas. Use when a task needs artwork, a hero image, a texture, an icon or a mockup image generated — especially when it should end up inside a Figma frame. Schema-driven: every parameter is validated against the model's own OpenAPI spec.
whenToUse: The user asks to generate an image, produce visual assets, or fill/show something on the Figma canvas with generated artwork.
---

# img-forge

CLI over fal.ai. The model's OpenAPI schema is the source of truth for parameters — so always ask the
schema instead of guessing what a model accepts.

## Preconditions

```bash
cd <tools>/img-forge
./img auth            # does a FAL_KEY exist?
./img models          # which models are registered
```

No key → stop and ask the user for one (`FAL_KEY` or `~/.config/img-forge/key`). Do not invent a key
and do not silently skip the step.

## The loop

1. **Inspect.** `./img models inspect <endpoint-id>` — lists the real parameters, their enums and
   defaults. Unknown id → 404, so verify before spending.
2. **Dry run.** `./img gen ... --dry-run` prints the exact payload and the cost estimate, and needs
   no key.
3. **Generate.** Drop `--dry-run`. If the model has no `costPerImageUsd` in `models.json`, the CLI
   refuses unless you pass `--yes` — only do that when the user has agreed to spend.
4. **Place.** `cd ../figma-bridge && ./bin/figma image <nodeId> <file> FILL` (or `FIT`/`TILE`/`CROP`).
5. **Verify.** `./bin/figma export <nodeId> out/check.png PNG 1` and look at the image.

## Examples

```bash
# expensive/precise model, full control
./img gen --prompt "matte black desk lamp on concrete, soft window light" \
          --preset 16:9@2k --reasoning high --quality xhigh --dry-run

# fast cheap one, aspect based
./img gen --prompt "abstract purple gradient" --preset 1:1@1k --model fal-ai/nano-banana

# anything the schema offers, without a dedicated flag
./img gen --prompt "..." --set num_inference_steps=28 --set guidance_scale=3.5
```

## Rules

- **Dry run first.** It costs nothing and catches typos that would otherwise cost money.
- **Never spend without consent.** No `--yes` unless the user asked for a generation explicitly; no
  `--max-usd` raising on your own initiative.
- **Trust the schema, not memory.** If a flag is rejected, the error lists the real parameters — use
  it, do not retry with guesses.
- **Keep the manifest.** `<file>.json` next to the image holds the payload and seed; report the path,
  it is what makes a run reproducible.
- **One image at a time** unless the user asked for variants. `--n` costs `n ×` the estimate.
- **Do not commit** `out/`, `.env` or `.cache/`.

## Pitfalls

- `--preset` on an enum-size model only picks an aspect; the resolution name is ignored. Use
  `--size WIDTHxHEIGHT` when exact pixels matter.
- Cost estimates come from a registry, not from fal — they are approximate. `./img spend` reports what
  was *recorded*, not what was charged.
- Cached results mean zero cost: the same payload twice returns the cached file. Use `--no-cache` when
  you actually want a fresh sample.
- Figma `imageFill` is a fill, not a layer: `FILL` crops, `FIT` letterboxes. Choose deliberately.
