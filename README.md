# img-forge

Image generation through [fal.ai](https://fal.ai), driven by each model's own **OpenAPI schema**.

Every fal endpoint exposes a different input contract: one model has `quality` + `reasoning_effort`,
another has `num_inference_steps` + `guidance_scale`, a third only knows `aspect_ratio`. Instead of
hard-coding a mapping per model, img-forge downloads the endpoint's OpenAPI spec and validates what
you pass against it — so you get every knob the model actually has, and an error listing them when
you typo one.

Zero dependencies. Node ≥ 20.

## Install

```bash
# key from https://fal.ai/dashboard/keys
export FAL_KEY="key-id:key-secret"
# or keep it on disk:
mkdir -p ~/.config/img-forge && printf '%s' "key-id:key-secret" > ~/.config/img-forge/key && chmod 600 ~/.config/img-forge/key
```

Optional `.env` next to the `img` script (gitignored) pins local paths and the key:

```
IMG_FORGE_CONFIG=./.config
IMG_FORGE_CACHE=./.cache
FAL_KEY=key-id:key-secret
```

## Use

```bash
./img models                       # registry + default model
./img models inspect openai/gpt-image-2.6/sunburst/text-to-image
./img gen --prompt "matte black desk lamp on concrete, soft window light" \
          --preset 16:9@2k --reasoning high --quality xhigh --dry-run
./img gen --prompt "..." --preset 16:9@2k --reasoning high --quality xhigh
./img spend --days 7
```

`--dry-run` prints the exact payload and the cost estimate without spending anything. Run it first;
it is the cheapest way to see whether a parameter is real.

### Model shortcuts

| flag | maps to | note |
|---|---|---|
| `--reasoning low\|medium\|high` | `reasoning_effort` | only if the model has it |
| `--quality auto…max` | `quality` | enum comes from the schema |
| `--background auto\|transparent\|opaque` | `background` | |
| `--format png\|jpeg\|webp\|gif\|svg` | `output_format` | |
| `--n 3` | `num_images` | |
| `--seed 42` | `seed` | |
| `--set key=value` | that literal field | validated like everything else |

### Editing an existing image

```bash
./img edit --prompt "make it a glassmorphism panel, keep the text" \
           --ref out/cover.png --reasoning high --quality high
./img edit --prompt "recolor to deep green" --ref a.png --ref b.png --dry-run
./img edit --prompt "remove the logo" --ref shot.png --mask mask.png
```

`edit` models take `image_urls` (required) plus optional `mask_url` / `markup_image_url`. Local files
are uploaded to the fal CDN first (initiate + PUT, no SDK), URLs pass through untouched. The cache key
includes a hash of every local reference, so re-running the same edit costs nothing.

### Size presets

`--preset <aspect>@<long-edge>` — e.g. `1:1@1k`, `16:9@2k`, `9:16@1080`. The tool converts it to
whatever the model wants:

- `sizeMode: "dims"` → `image_size: {width, height}` (GPT Image, FLUX)
- `sizeMode: "enum"` → `image_size: "landscape_16_9"`
- `sizeMode: "aspect"` → `aspect_ratio: "16:9"` (Nano Banana)

or pass pixels directly with `--size 1536x864`.

## Money and repetition

- **Cost estimate** comes from `costPerImageUsd` in `models.json`. If it is `null`, `gen` refuses to
  spend unless you pass `--yes`, and `--max-usd` (default `0.5`) blocks anything above the estimate.
- **Cache** keys on `sha256(endpoint + payload)`: an identical second run copies the cached bytes and
  costs nothing. `--no-cache` bypasses it.
- **Manifest** — every result lands next to a `<file>.json` with the full payload, seed, size and
  request id, so a run is reproducible.
- **Spend log** — `~/.config/img-forge/spend.jsonl`, summarised by `./img spend`.

## Pairing with Figma

Generation and canvas are separate on purpose: the plugin must never hold an API key or block the
Figma UI for 40 seconds. Compose the two tools instead:

```bash
./img gen --prompt "abstract gradient cover, purple to black" --preset 16:9@2k --out out/cover.png
cd ../figma-bridge && ./bin/figma image 12:34 ../img-forge/out/cover.png FILL
```

`./bin/figma image <nodeId> <file> [FILL|FIT|TILE|CROP]` base64-encodes the file and applies it as a
fill through `figma.createImage` — no upload to Figma's cloud.

There is also a UI for the same engine: the Figma Bridge plugin window has a Generate / Edit panel
that posts jobs to its local server, which in turn runs this CLI — so the panel and the terminal share
one implementation. Presets saved from the panel live in `figma-bridge/presets.json` and appear here
too. See [the bridge README](../figma-bridge/README.md#image-generation).

## Adding a model

`models.json` in this directory (or `~/.config/img-forge/models.json`, which wins):

```json
{
  "models": {
    "fal-ai/some/new-model": {
      "label": "Some New Model",
      "sizeMode": "dims",
      "sizeParam": "image_size",
      "costPerImageUsd": 0.04
    }
  }
}
```

Nothing else is needed — presets, validation and help text come from the schema. Verify the id with
`./img models inspect <id>`; a 404 means the endpoint does not exist.

## Security

- The key never enters this repo: `.env`, `.config/` and `.cache/` are gitignored.
- Dry runs and schema fetches work without a key; only `gen` (and only when spending) requires it.
- Generated files and manifests stay under `out/` unless you pass `--out`.
