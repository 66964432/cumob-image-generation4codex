# CUMOB Image Generation for Codex

**中文** | [English](README.en.md)

一个面向 Codex 的图片生成 Skill，通过当前 Codex provider 调用 OpenAI
兼容的 Images API 或 Responses API，支持图片生成、编辑、局部重绘和风格转换。

项目内置 Node.js 和 Python 两套零第三方依赖脚本，可直接读取 Codex 的
`config.toml` 与 `auth.json`，无需把 API Key 写进命令行。

当前版本：`0.1.1`

## 功能

- 根据 provider 的 `image_api` 自动选择 Images API 或 Responses API。
- 支持生成、编辑、多图片输入和蒙版局部重绘。
- 支持尺寸、质量、透明背景、输出格式和输入保真度等参数。
- 优先使用 Codex 已配置的 provider、模型与认证信息。
- Node.js 18+ 为首选运行时，Python 3 为后备运行时。
- 仅使用运行时标准库，无需执行 `npm install` 或 `pip install`。
- 长时间生成时在 stderr 输出进度，stdout 保留给结果摘要。
- 提供 `--dry-run` 检查配置和请求结构，不会显示 API Key 或图片内容。

## 项目结构

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

`SKILL.md` 是 Codex 加载的核心 Skill 指令。`scripts/` 中的两个实现提供基本
相同的命令行接口。

## 环境要求

至少安装以下一个运行时：

- Node.js 18 或更高版本，推荐。
- Python 3。

支持 macOS、Linux 和 Windows。无需安装 OpenAI SDK。

## 安装

### 安装到个人 Codex Skills

macOS 或 Linux：

```bash
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
git clone https://github.com/66964432/cumob-image-generation4codex.git \
  "${CODEX_HOME:-$HOME/.codex}/skills/cumob-image-generation4codex"
```

Windows PowerShell：

```powershell
$codexHome = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $HOME ".codex" }
New-Item -ItemType Directory -Force (Join-Path $codexHome "skills") | Out-Null
git clone https://github.com/66964432/cumob-image-generation4codex.git (Join-Path $codexHome "skills\cumob-image-generation4codex")
```

安装后重新启动 Codex，或新建一个 Codex 任务，使 Skill 列表重新加载。

### 安装到项目内

如果只希望某个仓库使用这个 Skill，可以将它克隆到项目的 `.codex/skills`
目录：

```bash
mkdir -p .codex/skills
git clone https://github.com/66964432/cumob-image-generation4codex.git \
  .codex/skills/cumob-image-generation4codex
```

项目级安装是否可用取决于当前 Codex 版本和工作区策略。如果 Codex 没有发现
该 Skill，请改用个人 Skills 目录安装。

## 配置

脚本默认读取：

- `$CODEX_HOME/config.toml`，如果设置了 `CODEX_HOME`。
- 否则读取 `~/.codex/config.toml`。
- 同目录下的 `auth.json` 用于获取 `OPENAI_API_KEY`。

### CUMOB Images API

在 Codex `config.toml` 中配置：

```toml
model_provider = "cumob"
model = "your-response-model"

[model_providers.cumob]
name = "CUMOB"
base_url = "http://api.cumob.com/v1"
image_api = "images"
image_model = "gpt-image-2"
```

脚本会调用：

- `<base_url>/images/generations`
- `<base_url>/images/edits`

### Responses API

对于支持 Responses API `image_generation` 工具的 provider：

```toml
model_provider = "openai-compatible"
model = "your-response-model"

[model_providers.openai-compatible]
name = "OpenAI Compatible"
base_url = "https://example.com/v1"
image_api = "responses"
image_model = "gpt-image-1"
```

如果没有配置 `image_api`，脚本默认使用 `responses`。

### 环境变量后备配置

Codex 配置不可用时，可以使用：

```bash
export OPENAI_BASE_URL="https://example.com/v1"
export OPENAI_MODEL="your-response-model"
export OPENAI_IMAGE_MODEL="gpt-image-1"
export OPENAI_IMAGE_API="responses"
export OPENAI_API_KEY="<your-api-key>"
```

不要把真实 API Key 写入仓库、聊天记录、脚本参数或 Git 提交。

## 使用

通常只需要在 Codex 中提出图片请求，例如：

```text
生成一张 1024x1024 的黑色陶瓷杯产品图，保存到 outputs/mug.png。
```

Codex 会根据 `SKILL.md` 调用对应脚本。也可以直接运行 CLI。

### 生成图片

```bash
node scripts/generate-image.mjs \
  --prompt "A matte black ceramic mug on a walnut desk, soft window light" \
  --out outputs/mug.png \
  --size 1024x1024 \
  --quality high
```

Python 后备命令：

```bash
python3 scripts/generate-image.py \
  --prompt "A matte black ceramic mug on a walnut desk, soft window light" \
  --out outputs/mug.png \
  --size 1024x1024 \
  --quality high
```

### 透明背景

```bash
node scripts/generate-image.mjs \
  --prompt "A centered folded paper crane app icon, no text" \
  --out outputs/crane.png \
  --background transparent \
  --format png
```

### 编辑图片

```bash
node scripts/generate-image.mjs \
  --prompt "Restyle as a polished editorial illustration while preserving composition" \
  --image reference.png \
  --action edit \
  --input-fidelity high \
  --out outputs/restyled.png
```

### 使用蒙版

```bash
node scripts/generate-image.mjs \
  --prompt "Replace the masked area with a glass vase of yellow flowers" \
  --image room.png \
  --mask mask.png \
  --action edit \
  --out outputs/inpainted.png
```

### 检查配置

`--dry-run` 不会发起 API 请求：

```bash
node scripts/generate-image.mjs \
  --prompt "Configuration check" \
  --out outputs/test.png \
  --dry-run
```

输出只会显示是否检测到密钥以及密钥来源，不会显示密钥值。输入图片的 base64
内容也会被隐藏。

查看全部参数：

```bash
node scripts/generate-image.mjs --help
```

## 常见问题

### Codex 没有发现 Skill

确认目录结构中直接包含 `SKILL.md`：

```text
~/.codex/skills/cumob-image-generation4codex/SKILL.md
```

然后重新启动 Codex 或新建一个任务。

### 找不到 API Key

优先检查 Codex 的 `auth.json` 是否包含 `OPENAI_API_KEY`。也可以通过环境变量
提供密钥，或者使用 `--api-key-env VARIABLE_NAME` 指定变量名。

不要使用 `--api-key`。脚本会主动拒绝该参数，避免密钥进入命令历史。

### 请求长时间没有返回

图片生成可能需要数分钟。出现 `Still waiting for image result` 表示原命令仍在
正常等待。不要因为暂时没有结果而重复启动相同请求。

### 后端路径不正确

运行 `--dry-run`，检查输出中的：

- `image_api`
- `base_url`
- `endpoint`
- `image_model`
- `response_model`

## 开发与验证

语法检查：

```bash
node --check scripts/generate-image.mjs
PYTHONPYCACHEPREFIX=/tmp/cumob-image-pycache \
  python3 -m py_compile scripts/generate-image.py
```

离线检查 Images API 请求：

```bash
node scripts/generate-image.mjs \
  --prompt "test" \
  --image-api images \
  --image-model gpt-image-2 \
  --dry-run
```

离线检查 Responses API 请求：

```bash
node scripts/generate-image.mjs \
  --prompt "test" \
  --image-api responses \
  --response-model test-response-model \
  --image-model test-image-model \
  --dry-run
```

`evals/evals.json` 包含基础代理行为评测场景。

## 版本

项目使用语义化版本：

- `MAJOR`：不兼容的命令行、配置或输出契约变更。
- `MINOR`：向后兼容的新功能或新后端能力。
- `PATCH`：向后兼容的错误修复和文档修正。

当前版本保存在仓库根目录的 `VERSION` 文件中。Git tag 使用 `v` 前缀，例如
`v0.1.1`。

## 发布到 GitHub

当前目录还不是 Git 仓库时，先执行：

```bash
git init
git add .
git commit -m "Initial release v0.1.1"
git branch -M main
```

使用 GitHub CLI 创建公开仓库并推送：

```bash
gh repo create cumob-image-generation4codex \
  --public \
  --source=. \
  --remote=origin \
  --push
```

或者先在 GitHub 创建空仓库，再手动添加远端：

```bash
git remote add origin git@github.com:66964432/cumob-image-generation4codex.git
git push -u origin main
```

创建首个版本标签：

```bash
git tag -a v0.1.1 -m "v0.1.1"
git push origin v0.1.1
```

然后在 GitHub 的 Releases 页面选择对应标签，填写发布说明并发布。

### 后续发布流程

1. 更新根目录 `VERSION`。
2. 检查 `SKILL.md`、README 和 CLI 参数是否一致。
3. 执行语法检查与 `--dry-run` 验证。
4. 提交版本变更。
5. 创建并推送对应的 `vX.Y.Z` tag。
6. 在 GitHub 创建 Release，并记录功能、修复和兼容性变化。

建议不要移动、压缩或重命名发布包中的 `SKILL.md` 与 `scripts/`，否则安装后的
Skill 可能无法正常工作。

## 安全

- 不要提交 `.env`、`auth.json` 或任何真实 API Key。
- 不要在 issue、日志或截图中展示 Authorization header。
- 调试时优先使用 `--dry-run`。
- 公开发布前检查 Git 历史中是否曾出现密钥。

## License

本项目采用 [Apache License 2.0](LICENSE) 开源许可证。
