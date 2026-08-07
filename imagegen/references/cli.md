# CLI reference (`scripts/image_gen.py`)

## What this CLI does
- `generate`: generate a new image from a prompt
- `edit`: edit one or more existing images
- `generate-batch`: run many generation jobs from a JSONL file

Real API calls require **network access** + an API key (see configuration below). `--dry-run` does not.

## Quick start (works from any repo)
Set a stable path to the skill CLI (use the absolute path of this skill's directory):

```bash
export IMAGE_GEN="<skill-dir>/scripts/image_gen.py"
```

Install dependencies with the interpreter that will run the CLI. The scripts need `openai` (required) and `pillow` (chroma-key removal and downscaling). Resolution order:

1. If `<skill-dir>/.venv/bin/python` exists and has both packages, use it for every command below. It is machine-local, git-ignored, and persists across sessions.
2. Else if the active `python3` has both packages, use it.
3. Else create the skill-local virtualenv once. It must live inside the skill directory — never in the user's current project or any other workspace:

```bash
cd "<skill-dir>"   # the skill's own directory; symlinks resolve to the real location
uv venv .venv   # or: python3 -m venv .venv
uv pip install -p .venv/bin/python openai pillow   # or: .venv/bin/pip install openai pillow
# then run: "<skill-dir>/.venv/bin/python" "$IMAGE_GEN" ...
```

## Endpoint and API key configuration

Resolution order (first match wins):

1. `--api-key` / `--base-url` CLI flags
2. `IMAGE_GEN_API_KEY` / `IMAGE_GEN_BASE_URL` environment variables
3. `OPENAI_API_KEY` / `OPENAI_BASE_URL` environment variables (SDK default)

Relay/proxy example:

```bash
export IMAGE_GEN_BASE_URL="https://your-relay.example.com/v1"
export IMAGE_GEN_API_KEY="sk-..."
```

Per-invocation override (useful when one agent juggles multiple providers):

```bash
python "$IMAGE_GEN" generate \
  --base-url "https://your-relay.example.com/v1" \
  --api-key "sk-..." \
  --prompt "A cozy alpine cabin at dawn" \
  --out output/imagegen/alpine-cabin.png
```

Never paste real keys into shared docs, commits, or chat transcripts.

## Quick start

Dry-run (no API call; no network required; does not require the `openai` package):

```bash
python "$IMAGE_GEN" generate \
  --prompt "Test" \
  --out output/imagegen/test.png \
  --dry-run
```

Notes:
- One-off dry-runs print the API payload and the computed output path(s).
- Repo-local finals should live under `output/imagegen/`.

Generate (requires API key + network):

```bash
python "$IMAGE_GEN" generate \
  --prompt "A cozy alpine cabin at dawn" \
  --size 1024x1024 \
  --out output/imagegen/alpine-cabin.png
```

Edit:

```bash
python "$IMAGE_GEN" edit \
  --image input.png \
  --prompt "Replace only the background with a warm sunset" \
  --out output/imagegen/sunset-edit.png
```

## Guardrails
- Use the bundled CLI directly (`python "$IMAGE_GEN" ...`) after activating the correct environment.
- Do **not** create one-off runners (for example `gen_images.py`) unless the user explicitly asks for a custom wrapper.
- **Never modify** `scripts/image_gen.py`. If something is missing, ask the user before doing anything else.
- Do not silently downgrade from `gpt-image-2` to `gpt-image-1.5`; ask first unless the user already explicitly requested `gpt-image-1.5`.

## Defaults
- Model: `gpt-image-2`
- Supported model family for this CLI: GPT Image models (`gpt-image-*`)
- Size: `auto`
- Quality: `medium`
- Output format: `png`
- Default one-off output path: `output/imagegen/output.png`
- Background: unspecified unless `--background` is set

## gpt-image-2 size and model guidance

`gpt-image-2` is the default model.

- Use `--quality low` for fast drafts, thumbnails, and quick iterations.
- Use `--quality medium`, `--quality high`, or `--quality auto` for final assets, dense text, diagrams, identity-sensitive edits, and high-resolution outputs.
- Square images are typically fastest. Use `--size 1024x1024` for quick square drafts.
- If the user asks for 4K-style output, use `--size 3840x2160` for landscape or `--size 2160x3840` for portrait.
- Do not pass `--input-fidelity` with `gpt-image-2`; this model always uses high fidelity for image inputs.
- Do not use `--background transparent` with `gpt-image-2`; the default transparent-image workflow generates on a flat chroma-key background plus local removal. Use `gpt-image-1.5` only after the user explicitly confirms the true-transparent path, unless they already requested `gpt-image-1.5`.

Popular `gpt-image-2` sizes:
- `1024x1024`
- `1536x1024`
- `1024x1536`
- `2048x2048`
- `2048x1152`
- `3840x2160`
- `2160x3840`
- `auto`

`gpt-image-2` size constraints:
- max edge `<= 3840px`
- both edges multiples of `16px`
- long edge to short edge ratio `<= 3:1`
- total pixels between `655,360` and `8,294,400`
- outputs above `2560x1440` total pixels are experimental

Relay/proxy endpoints may not offer every model or size. If the endpoint rejects a model or parameter, report the error instead of silently changing the request.

Fast draft:

```bash
python "$IMAGE_GEN" generate \
  --prompt "A product thumbnail of a matte ceramic mug on a stone surface" \
  --quality low \
  --size 1024x1024 \
  --out output/imagegen/mug-draft.png
```

Final 2K landscape:

```bash
python "$IMAGE_GEN" generate \
  --prompt "A polished landing-page hero image of a matte ceramic mug on a stone surface" \
  --quality high \
  --size 2048x1152 \
  --out output/imagegen/mug-hero.png
```

4K landscape:

```bash
python "$IMAGE_GEN" generate \
  --prompt "A detailed architectural visualization at golden hour" \
  --size 3840x2160 \
  --quality high \
  --out output/imagegen/architecture-4k.png
```

True transparent request:

Ask for confirmation before using this command unless the user already explicitly requested `gpt-image-1.5` or true/native transparency.

```bash
python "$IMAGE_GEN" generate \
  --model gpt-image-1.5 \
  --prompt "A clean product cutout on a transparent background" \
  --background transparent \
  --output-format png \
  --out output/imagegen/product-cutout.png
```

When using this path, explain briefly that chroma-key generation plus local removal is the default transparent-image path, but this request needs true model-native transparency. `gpt-image-2` does not support `background=transparent`, so `gpt-image-1.5` is required here.

## Quality, input fidelity, and masks

- `--quality` works for `generate`, `edit`, and `generate-batch`: `low|medium|high|auto`
- `--input-fidelity` is **edit-only** and validated as `low|high`; it is not supported for `gpt-image-2`
- `--mask` is **edit-only**

Example:

```bash
python "$IMAGE_GEN" edit \
  --model gpt-image-1.5 \
  --image input.png \
  --prompt "Change only the background" \
  --quality high \
  --input-fidelity high \
  --out output/imagegen/background-edit.png
```

Mask notes:
- For multi-image edits, pass repeated `--image` flags. Their order is meaningful, so describe each image by index and role in the prompt.
- The CLI accepts a single `--mask`.
- Image and mask must be the same size and format and each under 50MB.
- Masks must include an alpha channel.
- If multiple input images are provided, the mask applies to the first image.
- Masking is prompt-guided; do not promise exact pixel-perfect mask boundaries.
- Use a PNG mask when possible; the script treats mask handling as best-effort and does not perform full preflight validation beyond file checks/warnings.
- In the edit prompt, repeat invariants (`change only the background; keep the subject unchanged`) to reduce drift.

## Output handling
- Use `tmp/imagegen/` for temporary JSONL inputs or scratch files.
- Use `output/imagegen/` for final outputs.
- Reruns fail if a target file already exists unless you pass `--force`.
- `--out-dir` changes one-off naming to `image_1.<ext>`, `image_2.<ext>`, and so on.
- Downscaled copies use the default suffix `-web` unless you override it.

## Common recipes

Generate with augmentation fields:

```bash
python "$IMAGE_GEN" generate \
  --prompt "A minimal hero image of a ceramic coffee mug" \
  --use-case "product-mockup" \
  --style "clean product photography" \
  --composition "wide product shot with usable negative space for page copy" \
  --constraints "no logos, no text" \
  --out output/imagegen/mug-hero.png
```

Generate + also write a downscaled copy for fast web loading:

```bash
python "$IMAGE_GEN" generate \
  --prompt "A cozy alpine cabin at dawn" \
  --size 1024x1024 \
  --downscale-max-dim 1024 \
  --out output/imagegen/alpine-cabin.png
```

Generate multiple prompts concurrently (async batch):

```bash
mkdir -p tmp/imagegen output/imagegen/batch
cat > tmp/imagegen/prompts.jsonl << 'EOF'
{"prompt":"Cavernous hangar interior with a compact shuttle parked near the center","use_case":"stylized-concept","composition":"wide-angle, low-angle","lighting":"volumetric light rays through drifting fog","constraints":"no logos or trademarks; no watermark","size":"1536x1024"}
{"prompt":"Gray wolf in profile in a snowy forest","use_case":"photorealistic-natural","composition":"eye-level","constraints":"no logos or trademarks; no watermark","size":"1024x1024"}
EOF

python "$IMAGE_GEN" generate-batch \
  --input tmp/imagegen/prompts.jsonl \
  --out-dir output/imagegen/batch \
  --concurrency 5

rm -f tmp/imagegen/prompts.jsonl
```

Notes:
- `generate-batch` requires `--out-dir`.
- Use `--concurrency` to control parallelism (default `5`).
- Per-job overrides are supported in JSONL (for example `size`, `quality`, `background`, `output_format`, `output_compression`, `moderation`, `n`, `model`, `out`, and prompt-augmentation fields).
- `--n` generates multiple variants for a single prompt; `generate-batch` is for many different prompts.
- In batch mode, per-job `out` is treated as a filename under `--out-dir`.
- For many requested deliverable assets, provide one prompt/job per distinct asset and use semantic filenames when possible.

## CLI notes
- Supported sizes depend on the model. `gpt-image-2` supports flexible constrained sizes; older GPT Image models support `1024x1024`, `1536x1024`, `1024x1536`, or `auto`.
- True transparent outputs require `output_format` to be `png` or `webp` and are not supported by `gpt-image-2`.
- `--prompt-file`, `--output-compression`, `--moderation`, `--max-attempts`, `--fail-fast`, `--force`, and `--no-augment` are supported.
- This CLI is intended for GPT Image models. Do not assume older non-GPT image-model behavior applies here.

## See also
- API parameter quick reference: `references/image-api.md`
- Prompt examples: `references/sample-prompts.md`
- Transparent image workflow: `SKILL.md` and `scripts/remove_chroma_key.py`
