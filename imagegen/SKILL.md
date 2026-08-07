---
name: "imagegen"
description: "Generate or edit raster images when the task benefits from AI-created bitmap visuals such as photos, illustrations, textures, sprites, mockups, or transparent-background cutouts. Use when the agent should create a brand-new image, transform an existing image, or derive visual variants from references, and the output should be a bitmap asset rather than repo-native code or vector. Do not use when the task is better handled by editing existing SVG/vector/code-native assets, extending an established icon or logo system, or building the visual directly in HTML/CSS/canvas."
---

# Image Generation Skill

Generates or edits images for the current project (for example website assets, game assets, UI mockups, product mockups, wireframes, logo design, photorealistic images, or infographics) via the bundled `scripts/image_gen.py` CLI.

This skill talks to the OpenAI Image API through the `openai` Python SDK. It works against the official API or any OpenAI-compatible relay/proxy endpoint — no vendor-specific login or built-in tooling required. Any agent that can run Python can use it.

## Mode and rules

This skill has a single execution mode: the `scripts/image_gen.py` CLI, with three subcommands:

- `generate`
- `edit`
- `generate-batch`

Rules:
- Use the bundled CLI directly (`python "$IMAGE_GEN" ...`). Do not create one-off SDK runners.
- Never modify `scripts/image_gen.py`. If something is missing, ask the user before doing anything else.
- Do not silently downgrade from `gpt-image-2` to `gpt-image-1.5`. Treat this as a model downgrade and ask the user first, unless the user has already explicitly requested `gpt-image-1.5`.
- For transparent-image requests, default to the chroma-key workflow below (generate on a flat key color, remove locally). Use `gpt-image-1.5 --background transparent` only when the user asks for true/native transparency, local removal fails validation, or the subject is too complex for clean keying — see the transparent-image section.
- For many assets or variants, use `generate-batch` with one job per distinct asset. Do not use `n` as a substitute for separate prompts: `n` is for variants of one prompt.
- Do not overwrite an existing asset unless the user explicitly asked for replacement; otherwise create a sibling versioned filename such as `hero-v2.png` or `item-icon-edited.png`.

Set a stable path to the CLI once per session (use the absolute path of this skill's directory):

```bash
export IMAGE_GEN="<skill-dir>/scripts/image_gen.py"
```

Shared prompt guidance lives in `references/prompting.md` and `references/sample-prompts.md`.

CLI/API reference docs:
- `references/cli.md`
- `references/image-api.md`

Local post-processing helper:
- `scripts/remove_chroma_key.py`: removes a flat chroma-key background from a generated image and writes a PNG/WebP with alpha. Prefer auto-key sampling, soft matte, and despill for antialiased edges.

## Endpoint and API key configuration

The CLI needs an API key, and optionally a custom base URL for relay/proxy endpoints. Resolution order (first match wins):

1. `--api-key` / `--base-url` CLI flags
2. `IMAGE_GEN_API_KEY` / `IMAGE_GEN_BASE_URL` environment variables
3. `OPENAI_API_KEY` / `OPENAI_BASE_URL` environment variables (the SDK also honors these natively)

For a relay/proxy setup, point the CLI at the relay's OpenAI-compatible base URL, for example:

```bash
export IMAGE_GEN_BASE_URL="https://your-relay.example.com/v1"
export IMAGE_GEN_API_KEY="sk-..."
```

Never ask the user to paste the full key in chat. Ask them to set it locally and confirm when ready. If no key is configured, explain the resolution order above and let the user choose where to set it.

## When to use
- Generate a new image (concept art, product shot, cover, website hero)
- Generate a new image using one or more reference images for style, composition, or mood
- Edit an existing image (inpainting, lighting or weather transformations, background replacement, object removal, compositing, transparent background)
- Produce many assets or variants for one task

## When not to use
- Extending or matching an existing SVG/vector icon set, logo system, or illustration library inside the repo
- Creating simple shapes, diagrams, wireframes, or icons that are better produced directly in SVG, HTML/CSS, or canvas
- Making a small project-local asset edit when the source file already exists in an editable native format
- Any task where the user clearly wants deterministic code-native output instead of a generated bitmap

## Decision tree

Think about two separate questions:

1. **Intent:** is this a new image or an edit of an existing image?
2. **Execution strategy:** is this one asset or many assets/variants?

Intent:
- If the user wants to modify an existing image while preserving parts of it, treat the request as **edit** and use the `edit` subcommand with `--image`.
- If the user provides images only as references for style, composition, mood, or subject guidance, treat the request as **generate**. To actually condition on those images, pass them via `edit --image` with a generation-style prompt (the Image API has no separate reference-based generation endpoint).
- If the user provides no images, treat the request as **generate**.
- For edits, preserve invariants aggressively and save non-destructively by default.

Execution strategy:
- One asset or a few variants of one prompt: `generate` (use `--n` only for variants of a single prompt).
- Many distinct assets: `generate-batch` with one JSONL job per asset.

Assume the user wants a new image unless they clearly ask to change an existing one.

## Workflow
1. Decide the intent: `generate` or `edit`.
2. Decide whether the output is preview-only or meant to be consumed by the current project.
3. Decide the execution strategy: single `generate` vs `generate-batch`.
4. Confirm credentials: an API key (and base URL for relay setups) must resolve per the configuration section. Use `--dry-run` to validate the payload without a key or network.
5. Collect inputs up front: prompt(s), exact text (verbatim), constraints/avoid list, and any input images.
6. For every input image, label its role explicitly:
   - reference image
   - edit target
   - supporting insert/style/compositing input
7. If the user asked for a photo, illustration, sprite, product image, banner, or other explicitly raster-style asset, generate a real bitmap rather than substituting SVG/HTML/CSS placeholders. If the request is for an icon, logo, or UI graphic that should match existing repo-native SVG/vector/code assets, prefer editing those directly instead.
8. Augment the prompt based on specificity:
   - If the user's prompt is already specific and detailed, normalize it into a clear spec without adding creative requirements.
   - If the user's prompt is generic, add tasteful augmentation only when it materially improves output quality.
   - The CLI's structured augmentation fields (`--use-case`, `--style`, `--constraints`, ...) are on by default; pass `--no-augment` to send the raw prompt only.
9. For transparent-output requests, follow the transparent image guidance below: generate on a flat chroma-key background, run `scripts/remove_chroma_key.py`, and validate the alpha result before using it. If this path looks unsuitable or fails, ask before switching to `gpt-image-1.5`.
10. Inspect outputs and validate: subject, style, composition, text accuracy, and invariants/avoid items.
11. Iterate with a single targeted change, then re-check.
12. For project-bound work, save the artifact into the workspace (default `output/imagegen/`) and update any consuming code or references.
13. For batches, persist every requested deliverable final in the workspace unless the user explicitly asked to keep outputs preview-only. Discarded variants do not need to be kept unless requested.
14. Always report the final saved path(s), plus the final prompt or prompt set.

## Transparent image requests

Default path: chroma-key generation plus local removal. No special model required.

1. Generate the requested subject on a perfectly flat solid chroma-key background (see the prompt template below).
2. Choose a key color that is unlikely to appear in the subject: default `#00ff00`, use `#ff00ff` for green subjects, and avoid `#0000ff` for blue subjects.
3. Run the bundled helper:
   ```bash
   python "<skill-dir>/scripts/remove_chroma_key.py" \
     --input <source> \
     --out <final.png> \
     --auto-key border \
     --soft-matte \
     --transparent-threshold 12 \
     --opaque-threshold 220 \
     --despill
   ```
4. Validate that the output has an alpha channel, transparent corners, plausible subject coverage, and no obvious key-color fringe. If a thin fringe remains, retry once with `--edge-contract 1`; use `--edge-feather 0.25` only when the edge is visibly stair-stepped and the subject is not shiny or reflective.
5. Save the final alpha PNG/WebP in the project if the asset is project-bound.

Prompt transparent requests like this:

```text
Create the requested subject on a perfectly flat solid #00ff00 chroma-key background for background removal.
The background must be one uniform color with no shadows, gradients, texture, reflections, floor plane, or lighting variation.
Keep the subject fully separated from the background with crisp edges and generous padding.
Do not use #00ff00 anywhere in the subject.
No cast shadow, no contact shadow, no reflection, no watermark, and no text unless explicitly requested.
```

Do not automatically use `gpt-image-1.5 --background transparent --output-format png` instead of chroma keying. Ask the user first when the user asks for true/native transparency, when local removal fails validation, or when the requested image is complex: hair, fur, feathers, smoke, glass, liquids, translucent materials, reflective objects, soft shadows, realistic product grounding, or subject colors that conflict with all practical key colors.

Use a concise confirmation like:

```text
This likely needs true native transparency. The default path uses a chroma-key background plus local removal, but true transparency requires gpt-image-1.5 because gpt-image-2 does not support background=transparent. Should I proceed with gpt-image-1.5?
```

Note: relay/proxy endpoints do not always support every model or parameter. If `background=transparent` or a specific model fails against the configured endpoint, report the API error plainly instead of silently switching models or dropping the parameter.

## Prompt augmentation

Reformat user prompts into a structured, production-oriented spec. Make the user's goal clearer and more actionable, but do not blindly add detail.

Treat this as prompt-shaping guidance, not a closed schema. Use only the lines that help, and add a short extra labeled line when it materially improves clarity.

### Specificity policy

Use the user's prompt specificity to decide how much augmentation is appropriate:

- If the prompt is already specific and detailed, preserve that specificity and only normalize/structure it.
- If the prompt is generic, you may add tasteful augmentation when it will materially improve the result.

Allowed augmentations:
- composition or framing hints
- polish level or intended-use hints
- practical layout guidance
- reasonable scene concreteness that supports the stated request

Not allowed augmentations:
- extra characters or objects that are not implied by the request
- brand names, slogans, palettes, or narrative beats that are not implied
- arbitrary side-specific placement unless the surrounding layout supports it

## Use-case taxonomy (exact slugs)

Classify each request into one of these buckets and keep the slug consistent across prompts and references.

Generate:
- photorealistic-natural — candid/editorial lifestyle scenes with real texture and natural lighting.
- product-mockup — product/packaging shots, catalog imagery, merch concepts.
- ui-mockup — app/web interface mockups and wireframes; specify the desired fidelity.
- infographic-diagram — diagrams/infographics with structured layout and text.
- scientific-educational — classroom explainers, scientific diagrams, and learning visuals with required labels and accuracy constraints.
- ads-marketing — campaign concepts and ad creatives with audience, brand position, scene, and exact tagline/copy.
- productivity-visual — slide, chart, workflow, and data-heavy business visuals.
- logo-brand — logo/mark exploration, vector-friendly.
- illustration-story — comics, children’s book art, narrative scenes.
- stylized-concept — style-driven concept art, 3D/stylized renders.
- historical-scene — period-accurate/world-knowledge scenes.

Edit:
- text-localization — translate/replace in-image text, preserve layout.
- identity-preserve — try-on, person-in-scene; lock face/body/pose.
- precise-object-edit — remove/replace a specific element (including interior swaps).
- lighting-weather — time-of-day/season/atmosphere changes only.
- background-extraction — transparent background / clean cutout. Use chroma-key removal first for simple opaque subjects; ask before using `gpt-image-1.5` true transparency for complex subjects.
- style-transfer — apply reference style while changing subject/scene.
- compositing — multi-image insert/merge with matched lighting/perspective.
- sketch-to-render — drawing/line art to photoreal render.

## Shared prompt schema

Use the following labeled spec as shared prompt scaffolding (the CLI's `--augment` fields mirror this structure):

```text
Use case: <taxonomy slug>
Asset type: <where the asset will be used>
Primary request: <user's main prompt>
Input images: <Image 1: role; Image 2: role> (optional)
Scene/backdrop: <environment>
Subject: <main subject>
Style/medium: <photo/illustration/3D/etc>
Composition/framing: <wide/close/top-down; placement>
Lighting/mood: <lighting + mood>
Color palette: <palette notes>
Materials/textures: <surface details>
Text (verbatim): "<exact text>"
Constraints: <must keep/must avoid>
Avoid: <negative constraints>
```

Notes:
- `Asset type` and `Input images` are prompt scaffolding, not dedicated CLI flags.
- `Scene/backdrop` refers to the visual setting. It is not the same as the CLI `--background` parameter, which controls output transparency behavior.

Augmentation rules:
- Keep it short.
- Add only the details needed to improve the prompt materially.
- For edits, explicitly list invariants (`change only X; keep Y unchanged`).
- If any critical detail is missing and blocks success, ask a question; otherwise proceed.

## Examples

### Generation example (hero image)
```text
Use case: product-mockup
Asset type: landing page hero
Primary request: a minimal hero image of a ceramic coffee mug
Style/medium: clean product photography
Composition/framing: wide composition with usable negative space for page copy if needed
Lighting/mood: soft studio lighting
Constraints: no logos, no text, no watermark
```

### Edit example (invariants)
```text
Use case: precise-object-edit
Asset type: product photo background replacement
Primary request: replace only the background with a warm sunset gradient
Constraints: change only the background; keep the product and its edges unchanged; no text; no watermark
```

## Prompting best practices
- Structure prompt as scene/backdrop -> subject -> details -> constraints.
- Include intended use (ad, UI mock, infographic) to set the mode and polish level.
- Use camera/composition language for photorealism.
- Only use SVG/vector stand-ins when the user explicitly asked for vector output or a non-image placeholder.
- Quote exact text and specify typography + placement.
- For tricky words, spell them letter-by-letter and require verbatim rendering.
- For multi-image inputs, reference images by index and describe how they should be used.
- For edits, repeat invariants every iteration to reduce drift.
- Iterate with single-change follow-ups.
- If the prompt is generic, add only the extra detail that will materially help.
- If the prompt is already detailed, normalize it instead of expanding it.
- See `references/cli.md` and `references/image-api.md` for model, `quality`, `input_fidelity`, masks, output format, and output-path guidance.
- For transparent images, use the chroma-key workflow by default; ask before switching to `gpt-image-1.5` true transparency.

More principles: `references/prompting.md`.
Copy/paste specs: `references/sample-prompts.md`.

## Guidance by asset type
Asset-type templates (website assets, game assets, wireframes, logo) are consolidated in `references/sample-prompts.md`.

## gpt-image-2 guidance

The CLI defaults to `gpt-image-2`.

- Use `gpt-image-2` for new workflows unless the request needs true model-native transparent output.
- If a transparent request may need true transparency, ask before using `gpt-image-1.5` unless the user already explicitly requested it. Explain that the chroma-key path is the default, but true transparency requires `gpt-image-1.5` because `gpt-image-2` does not support `background=transparent`.
- `gpt-image-2` always uses high fidelity for image inputs; do not set `input_fidelity` with this model.
- `gpt-image-2` supports `quality` values `low`, `medium`, `high`, and `auto`.
- Use `quality low` for fast drafts, thumbnails, and quick iterations. Use `medium`, `high`, or `auto` for final assets, dense text, diagrams, identity-sensitive edits, or high-resolution outputs.
- Square images are typically fastest to generate. Use `1024x1024` for fast square drafts.
- If the user asks for 4K-style output, use `3840x2160` for landscape or `2160x3840` for portrait.
- `gpt-image-2` size may be `auto` or `WIDTHxHEIGHT` if all constraints hold: max edge `<= 3840px`, both edges multiples of `16px`, long-to-short ratio `<= 3:1`, total pixels between `655,360` and `8,294,400`.
- Relay/proxy endpoints may not offer every model or size. If the endpoint rejects a model or parameter, report the error instead of silently changing the request.

Popular `gpt-image-2` sizes:
- `1024x1024` square
- `1536x1024` landscape
- `1024x1536` portrait
- `2048x2048` 2K square
- `2048x1152` 2K landscape
- `3840x2160` 4K landscape
- `2160x3840` 4K portrait
- `auto`

## CLI mode

### Temp and output conventions
- Use `tmp/imagegen/` for intermediate files (for example JSONL batches); delete them when done.
- Write final artifacts under `output/imagegen/`.
- Use `--out` or `--out-dir` to control output paths; keep filenames stable and descriptive.

### Dependencies
Prefer `uv` for dependency management.

The CLI needs the `openai` package; local chroma-key removal and optional downscaling need `pillow`. Resolve the interpreter to run the scripts in this order:

1. If `<skill-dir>/.venv/bin/python` exists and can `import openai, PIL`, use it for all script invocations. This venv is machine-local and git-ignored; it persists across sessions.
2. Otherwise, if the active `python3` can `import openai, PIL`, use it.
3. Otherwise, create the skill-local virtualenv once and use it thereafter. It must live inside the skill directory — never create it in the user's current project or any other workspace:

```bash
cd "<skill-dir>"   # the skill's own directory; symlinks resolve to the real location
uv venv .venv   # or: python3 -m venv .venv
uv pip install -p .venv/bin/python openai pillow   # or: .venv/bin/pip install openai pillow
```

Then invoke the CLI as `"<skill-dir>/.venv/bin/python" scripts/image_gen.py ...` instead of bare `python3`.

Creating `.venv` only adds git-ignored local files; never modify tracked skill files to work around dependency issues. If installation is not possible, tell the user which dependency is missing and how to install it.

### Script-mode notes
- CLI commands + examples: `references/cli.md`
- API parameter quick reference: `references/image-api.md`

## Reference map
- `references/prompting.md`: shared prompting principles.
- `references/sample-prompts.md`: copy/paste prompt recipes.
- `references/cli.md`: CLI usage via `scripts/image_gen.py`.
- `references/image-api.md`: API/CLI parameter reference.
- `scripts/image_gen.py`: CLI implementation. Do not load or use it unless image generation/editing was actually requested.
- `scripts/remove_chroma_key.py`: local post-processing helper for transparent-image requests.
