---
name: img-forge
description: "Generate or edit images with fal.ai models from the command line, then place them on a Figma canvas. Use when a task needs artwork, a hero image, a texture, an icon or a retouched photo — especially when it should end up inside a Figma frame or file. Every parameter is validated against the model's own OpenAPI schema, cost is estimated before any request, and identical runs are cached."
whenToUse: "The user asks to generate an image, produce visual assets, restyle or retouch an existing image, or fill a Figma canvas with generated artwork."
---

# img-forge

CLI over fal.ai. The model's OpenAPI schema is the source of truth for parameters — ask the schema
instead of guessing what a model accepts.

## Preconditions

```bash
cd <tools>/img-forge
./img auth            # is a FAL_KEY available?
./img models          # which models are registered
```

No key → stop and ask the user for one (`FAL_KEY`, or `~/.config/img-forge/key`). Do not invent one
and do not silently skip the step.

## The loop

1. **Inspect.** `./img models inspect <endpoint-id>` lists the real parameters, enums and defaults.
   An unknown id returns 404, so verify before spending.
2. **Dry run.** `./img gen … --dry-run` prints the exact payload, the size and the cost estimate; it
   needs no key and spends nothing.
3. **Generate.** Drop `--dry-run`. If the estimate exceeds `--max-usd` (default `$0.5`) the CLI
   refuses before touching the network.
4. **Place (optional).** `cd ../figma-bridge && ./bin/figma image <nodeId> <file> FILL` renders it into
   the canvas; `./bin/figma export <nodeId> out/check.png PNG 2` verifies what came out.
5. **Report the manifest.** `<file>.json` next to the image holds the payload, seed and request id.

## Generating

```bash
./img gen --prompt "matte black desk lamp on concrete, soft window light" \
          --preset 16:9@2k --reasoning high --quality xhigh --dry-run
./img gen --prompt "…" --preset 1:1@1024 --quality medium --yes
```

`--preset` is `<aspect>@<long-edge>` (`1:1@1k`, `16:9@2k`, `9:16@1080`, `1:1@1024`) or an explicit
`--size 1536x864`. The tool maps it onto whatever the model actually uses (`image_size` dims, an
`image_size` enum, or `aspect_ratio`).

## Editing an existing image

```bash
./img edit --prompt "make this card glassmorphism, keep the text" --ref out/card.png
./img edit --prompt "recolor to brand green" --ref out/card.png --ref palette.png --dry-run
./img edit --prompt "remove the logo" --ref shot.png --mask mask.png
```

References are local files (`--ref`, uploaded to the fal CDN for you) or URLs (`--ref-url`). Masks and
annotations: `--mask`, `--markup`. To edit what the user selected in Figma, export it first —
`./bin/figma export <nodeId> out/sel.png PNG 2` — and pass that file; always prefer the user's actual
selection over a guess.

## Cost discipline

- Estimates come from `models.json` pricing (`$30/1M` output image tokens by default) and scale with
  pixel area; they are estimates, not invoices.
- `--yes` is only for accepting an unknown cost. Never raise `--max-usd` on your own initiative, and
  never run `--n 4` unless the user asked for variants.
- The cache keys on `sha256(model + payload + reference hashes)`: a repeat run costs nothing. Use
  `--no-cache` only when the user wants a fresh sample.
- `./img spend` summarises what was recorded.

## The Figma panel (for humans)

The Figma Bridge plugin window has a Generate / Edit panel that posts jobs to its local server, which
runs this same CLI. Agents should call the CLI directly — it is the full surface (mask, variants,
`--dry-run`, spend log), while the panel covers the everyday cases. Presets saved from the panel live
in `figma-bridge/presets.json` and are shared with this tool.

## Rules

- Dry run first when the model or the settings are new.
- Trust the schema, not memory: a rejected flag lists the parameters that do exist.
- Do not commit `out/`, `.env`, `.cache/` or `presets.json`.
- Report the file path and the manifest path, not just "it worked".

## Pitfalls

- `--preset` on an enum-size model only picks an aspect; the resolution in the name is ignored. Use
  `--size WxH` when exact pixels matter.
- Cached results return the old file. That is the point, but mention it to the user.
- `image_size` caps at 14142 px per side on most fal endpoints; the panel enforces 64–8192.
- A model without `costPerImageUsd` and without a `pricing` block makes the CLI refuse without
  `--yes` — that is deliberate.
