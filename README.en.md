# CUMOB Image Generation for Codex

[中文](README.md) | **English**

An image-generation Skill for Codex that uses the active Codex provider to call
an OpenAI-compatible Images API or Responses API. It supports image generation,
editing, inpainting, and restyling.

The project includes dependency-free Node.js and Python scripts. They read
Codex's `config.toml` and `auth.json` directly, so API keys do not need to be
placed on the command line.

Current version: `0.2.0`

## Features

- Automatically selects the Images API or Responses API from the provider's
  `image_api` setting.
- Supports generation, editing, multiple input images, and mask-based
  inpainting.
- Supports size, quality, transparent backgrounds, output formats, and input
  fidelity options.
- Uses the provider, models, endpoint, and authentication already configured
  for Codex.
- Prefers Node.js 18+ and provides Python 3 as a fallback.
- Uses only built-in runtime modules; no `npm install` or `pip install` is
  required.
- Prints long-running progress to stderr while keeping stdout available for
  result summaries.
- Provides `--dry-run` to inspect configuration and request structure without
  exposing API keys or image contents.
- Compresses reference images larger than 4 MB into temporary uploads with a
  maximum dimension of 1536 pixels.
- Preserves transparent PNG inputs and never modifies originals or mask files.

## Repository Layout

```text
cumob-image-generation4codex/
├── SKILL.md
├── README.md
├── README.en.md
├── LICENSE
├── VERSION
├── evals/
│   └── evals.json
└── scripts/
    ├── generate-image.mjs
    └── generate-image.py
```

`SKILL.md` contains the instructions loaded by Codex. The two scripts under
`scripts/` expose substantially the same command-line interface.

## Requirements

Install at least one of the following runtimes:

- Node.js 18 or newer, recommended.
- Python 3.

The scripts support macOS, Linux, and Windows and do not require the OpenAI SDK.

## Installation

### Personal Codex Skill

macOS or Linux:

```bash
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
git clone https://github.com/66964432/cumob-image-generation4codex.git \
  "${CODEX_HOME:-$HOME/.codex}/skills/cumob-image-generation4codex"
```

Windows PowerShell:

```powershell
$codexHome = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $HOME ".codex" }
New-Item -ItemType Directory -Force (Join-Path $codexHome "skills") | Out-Null
git clone https://github.com/66964432/cumob-image-generation4codex.git (Join-Path $codexHome "skills\cumob-image-generation4codex")
```

Restart Codex or start a new Codex task after installation so the Skill list is
reloaded.

### Project-local Skill

To make the Skill available only inside one repository, clone it into that
project's `.codex/skills` directory:

```bash
mkdir -p .codex/skills
git clone https://github.com/66964432/cumob-image-generation4codex.git \
  .codex/skills/cumob-image-generation4codex
```

Project-local Skill availability depends on the current Codex version and
workspace policy. Use the personal Skills directory if Codex does not discover
the project-local installation.

## Configuration

By default, the scripts read:

- `$CODEX_HOME/config.toml` when `CODEX_HOME` is set.
- Otherwise, `~/.codex/config.toml`.
- `auth.json` in the same directory for `OPENAI_API_KEY`.

### CUMOB Images API

Add a provider to the Codex `config.toml`:

```toml
model_provider = "cumob"
model = "your-response-model"

[model_providers.cumob]
name = "CUMOB"
base_url = "http://api.cumob.com/v1"
image_api = "images"
image_model = "gpt-image-2-ref"
```

The scripts call:

- `<base_url>/images/generations`
- `<base_url>/images/edits`

### Responses API

For a provider that supports the Responses API `image_generation` tool:

```toml
model_provider = "openai-compatible"
model = "your-response-model"

[model_providers.openai-compatible]
name = "OpenAI Compatible"
base_url = "https://example.com/v1"
image_api = "responses"
image_model = "gpt-image-1"
```

When `image_api` is not configured, the scripts default to `responses`.

### Environment-variable Fallback

When Codex configuration is unavailable, use:

```bash
export OPENAI_BASE_URL="https://example.com/v1"
export OPENAI_MODEL="your-response-model"
export OPENAI_IMAGE_MODEL="gpt-image-1"
export OPENAI_IMAGE_API="responses"
export OPENAI_API_KEY="<your-api-key>"
```

Never place a real API key in the repository, chat messages, command-line
arguments, or Git commits.

## Usage

In normal use, ask Codex for an image:

```text
Generate a 1024x1024 product photo of a black ceramic mug and save it to outputs/mug.png.
```

Codex uses `SKILL.md` to select and run the appropriate script. The CLI can also
be invoked directly.

### Automatic Reference-image Compression

For image edits, the scripts inspect each `--image` file by default. Inputs
larger than 4 MB are prepared as temporary local upload copies:

- The longest side is reduced to `1536px`.
- Ordinary images are converted to JPEG at quality `85`.
- Images with an alpha channel remain PNG.
- Original images and `--mask` files are not modified.
- Temporary files are removed when the command exits.

On macOS, the scripts use the built-in `sips` command. Other platforms try
ImageMagick's `magick` command. When neither optimizer is available, the scripts
continue with the original input instead of failing.

Override the defaults:

```bash
node scripts/generate-image.mjs \
  --prompt "Restyle while preserving composition" \
  --image reference.png \
  --max-input-dimension 1536 \
  --input-jpeg-quality 85 \
  --input-optimize-threshold-mb 4 \
  --out outputs/result.png
```

Upload original input files without preprocessing:

```bash
node scripts/generate-image.mjs \
  --prompt "Use the exact original input bytes" \
  --image reference.png \
  --no-input-optimization \
  --out outputs/result.png
```

### Generate an Image

```bash
node scripts/generate-image.mjs \
  --prompt "A matte black ceramic mug on a walnut desk, soft window light" \
  --out outputs/mug.png \
  --size 1024x1024 \
  --quality high
```

Python fallback:

```bash
python3 scripts/generate-image.py \
  --prompt "A matte black ceramic mug on a walnut desk, soft window light" \
  --out outputs/mug.png \
  --size 1024x1024 \
  --quality high
```

### Transparent Background

```bash
node scripts/generate-image.mjs \
  --prompt "A centered folded paper crane app icon, no text" \
  --out outputs/crane.png \
  --background transparent \
  --format png
```

### Edit an Image

```bash
node scripts/generate-image.mjs \
  --prompt "Restyle as a polished editorial illustration while preserving composition" \
  --image reference.png \
  --action edit \
  --input-fidelity high \
  --out outputs/restyled.png
```

### Inpaint with a Mask

```bash
node scripts/generate-image.mjs \
  --prompt "Replace the masked area with a glass vase of yellow flowers" \
  --image room.png \
  --mask mask.png \
  --action edit \
  --out outputs/inpainted.png
```

### Inspect Resolved Configuration

`--dry-run` does not send an API request:

```bash
node scripts/generate-image.mjs \
  --prompt "Configuration check" \
  --out outputs/test.png \
  --dry-run
```

The output reports whether a key was found and where it came from, but never
prints the key value. Base64 data from input images is also redacted.

Display all CLI options:

```bash
node scripts/generate-image.mjs --help
```

## Troubleshooting

### Codex Does Not Discover the Skill

Confirm that `SKILL.md` exists directly under the installed Skill directory:

```text
~/.codex/skills/cumob-image-generation4codex/SKILL.md
```

Then restart Codex or start a new task.

### API Key Not Found

First check that Codex's `auth.json` contains `OPENAI_API_KEY`. Alternatively,
provide the key through an environment variable or use
`--api-key-env VARIABLE_NAME` to name a different variable.

Do not use `--api-key`. The scripts reject that option to keep secrets out of
shell history.

### The Request Is Taking a Long Time

Image generation can take several minutes. A
`Still waiting for image result` message means the original command is still
waiting normally. Do not start a duplicate request while it is running.

### Incorrect Backend Path

Run `--dry-run` and inspect:

- `image_api`
- `base_url`
- `endpoint`
- `image_model`
- `response_model`

## Development and Validation

Syntax checks:

```bash
node --check scripts/generate-image.mjs
PYTHONPYCACHEPREFIX=/tmp/cumob-image-pycache \
  python3 -m py_compile scripts/generate-image.py
```

Inspect an Images API request offline:

```bash
node scripts/generate-image.mjs \
  --prompt "test" \
  --image-api images \
  --image-model gpt-image-2-ref \
  --dry-run
```

Inspect a Responses API request offline:

```bash
node scripts/generate-image.mjs \
  --prompt "test" \
  --image-api responses \
  --response-model test-response-model \
  --image-model test-image-model \
  --dry-run
```

`evals/evals.json` contains basic agent-behavior evaluation scenarios.

## Versioning

The project follows semantic versioning:

- `MAJOR`: incompatible CLI, configuration, or output-contract changes.
- `MINOR`: backward-compatible features or backend capabilities.
- `PATCH`: backward-compatible fixes and documentation changes.

The current version is stored in the root `VERSION` file. Git tags use a `v`
prefix, for example `v0.2.0`.

## Publishing to GitHub

For a directory that has not been initialized as a Git repository:

```bash
git init
git add .
git commit -m "Initial release v0.2.0"
git branch -M main
```

Create and push a public repository with GitHub CLI:

```bash
gh repo create cumob-image-generation4codex \
  --public \
  --source=. \
  --remote=origin \
  --push
```

Or add this repository as the remote:

```bash
git remote add origin git@github.com:66964432/cumob-image-generation4codex.git
git push -u origin main
```

Create a release tag:

```bash
git tag -a v0.2.0 -m "v0.2.0"
git push origin v0.2.0
```

Then create a GitHub Release from the corresponding tag.

### Subsequent Releases

1. Update the root `VERSION` file.
2. Check that `SKILL.md`, both README files, and the CLI options agree.
3. Run syntax checks and `--dry-run` validation.
4. Commit the version changes.
5. Create and push the corresponding `vX.Y.Z` tag.
6. Create a GitHub Release describing features, fixes, and compatibility
   changes.

Do not move, flatten, or rename `SKILL.md` or `scripts/` in release archives.
Changing that layout can prevent the installed Skill from working.

## Security

- Never commit `.env`, `auth.json`, or a real API key.
- Never expose an Authorization header in issues, logs, or screenshots.
- Prefer `--dry-run` when debugging configuration.
- Check Git history for secrets before publishing.

## License

Licensed under the [Apache License 2.0](LICENSE).
