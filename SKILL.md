---
name: light-transfer
description: Automate image light transfer using RunningHub AI. Scans two image directories, generates all combinations (Cartesian product), processes each with the RunningHub workflow, and saves outputs with naming convention "A_B.ext". Supports dry-run, simulation, and resume modes.
---

# Light Transfer Skill

Automatically process image pairs with AI-powered light transfer effects.

## 🎯 When to Use

- You have a set of base images (e.g., portraits, subjects) in one directory
- You have a set of lighting/effect images in another directory
- You want to automatically apply all light combinations to all base images
- You need to use the RunningHub AI application (workflow ID: 2037342126949797890)
- You want progress tracking, retry logic, and detailed reporting

## 🚀 Quick Start

```bash
# 1. Prepare directories
mkdir -p ~/光线迁移/pic1  # base images
mkdir -p ~/光线迁移/pic2  # light/effect images

# 2. Copy images into the directories

# 3. Preview combinations (no processing)
light-transfer --dry-run

# 4. Run full processing
light-transfer

# 5. Resume if interrupted
light-transfer --resume
```

## 📦 What This Skill Provides

- **Complete workflow**: Scanning → combination generation → RunningHub API submission → polling → download → archiving
- **Robust error handling**: Auto-retry (up to 3 times), exponential backoff
- **Progress persistence**: Save/load completed tasks to resume from interruption
- **Simulation mode**: Test the entire pipeline without API calls
- **Comprehensive logging**: Per-run logs, success/failure reports

## 🔧 Configuration

### Default Configuration

The skill uses a default configuration suitable for most cases:

```yaml
workflow:
  workflow_id: "2037342126949797890"
  base_url: "https://www.runninghub.cn"

paths:
  pic1_dir: "/home/lee/光线迁移/pic1"   # base images
  pic2_dir: "/home/lee/光线迁移/pic2"   # light/effect images
  output_base: "/home/lee/光线迁移/outputs"
  log_dir: "logs"

processing:
  output_extension: "png"
  max_retries: 3
  retry_delay: 5
  task_timeout: 600
  delay_between_tasks: 5

resume:
  enabled: true
  state_file: "logs/completed.json"
```

### Override Configuration

Create your own config file (e.g., `my_config.yaml`) and pass it:

```bash
light-transfer --config my_config.yaml
```

### Environment Variables

- `RUNNINGHUB_API_KEY`: Required for actual API calls (set in your environment)

## 🎛️ Command Line Interface

```bash
light-transfer [OPTIONS]
```

**Options:**

| Flag | Description |
|------|-------------|
| `--config <path>` | Configuration file path (default: `config.yaml` in current directory) |
| `--output <dir>` | Override output directory |
| `--concurrent <N>` | Number of concurrent tasks (default: 1, keep 1 to avoid rate limits) |
| `--resume` | Skip already completed combinations (loads from `logs/completed.json`) |
| `--simulate` | Simulate mode (creates placeholder files instead of real API calls) |
| `--dry-run` | Only scan combinations, do not process |
| `--log-level <LEVEL>` | Logging level: DEBUG, INFO, WARNING, ERROR (default: INFO) |

**Examples:**

```bash
# Preview only
light-transfer --dry-run

# Full run with custom config
light-transfer --config /path/to/config.yaml --output /custom/output

# Simulate to test workflow
light-transfer --simulate

# Resume interrupted run
light-transfer --resume

# Debug logging
light-transfer --log-level DEBUG
```

## 📂 File Structure

Expected project layout:

```
光线迁移/
├── light_transfer.py     (main script - already present)
├── SKILL.md              (this file - optional after skill is installed)
├── config.yaml           (configuration - can be anywhere)
├── scripts/
│   ├── runner.py         (RunningHub API wrapper)
│   ├── scanner.py        (directory scanner + combination generator)
│   ├── utils.py          (logging, retry helpers)
│   └── poll_task.py      (smart polling logic)
├── pics/
│   ├── pic1/             (base images)
│   └── pic2/             (lighting images)
├── outputs/              (generated outputs, auto-created)
│   └── YYYY-MM-DD/
└── logs/                 (run logs, progress files, auto-created)
```

**Note:** This skill assumes you already have the `light_transfer.py` project set up. The skill files (`runner.py`, `scanner.py`, `utils.py`, `poll_task.py`) are bundled in the skill's `scripts/` directory and will be used automatically.

## 🔄 Workflow

1. **Scan**: Read all images from `pic1_dir` and `pic2_dir`
2. **Combine**: Generate Cartesian product → list of `(pic1, pic2, output_name)`
3. **Process**: For each combination:
   - Upload both images to RunningHub
   - Submit task with workflow ID
   - Poll until completion (three-stage polling)
   - Download output image
   - Save to `outputs/YYYY-MM-DD/{pic1_name}_{pic2_name}.png`
4. **Report**: Generate `report.json` with success/failure stats

## 🐛 Troubleshooting

### API Key Missing
```
RUNNINGHUB_API_KEY environment variable not set
```
**Fix**: Export your API key:
```bash
export RUNNINGHUB_API_KEY="your_api_key_here"
```

### Workflow Not Found
Task submission returns error code 805 (workflow not found).
**Fix**: Verify `workflow_id` in `config.yaml` matches the RunningHub AI app URL.

### Upload Fails
Images fail to upload (network or format issue).
**Fix**:
- Check image formats: JPG, PNG, WebP supported
- Ensure file sizes < 10MB each
- Retry automatically (max 3 attempts)

### Too Many Requests (Rate Limit)
**Fix**: Increase `delay_between_tasks` (default 5 seconds). If using concurrent >1, reduce to 1.

### Headless Chrome Issues (if using browser-based workflow)
The original browser-based workflow required Chrome CDP.
**Fix**: Use the pure API approach (default). No Chrome needed.

## 📊 Output Files

- **Images**: `outputs/2026-04-01/Alice_暖光斑1.png`
- **Report**: `outputs/2026-04-01/report.json`
- **Logs**: `logs/light_transfer.log`
- **Progress**: `logs/completed.json` (list of completed `pic1|pic2` keys)
- **Failures**: `logs/failed.json` (error details for failed tasks)

## 🔑 API Key 配置（必读）

运行前必须提供 RunningHub API Key。系统按优先级查找：

```
1. 命令行参数: --api-key YOUR_KEY
2. 环境变量:   RUNNINGHUB_API_KEY
3. OpenClaw 配置: ~/.openclaw/openclaw.json → skills.entries.runninghub.apiKey
```

### 快速设置
```bash
export RUNNINGHUB_API_KEY="your_key_here"
```

### 获取 API Key
- 访问 https://www.runninghub.cn
- 登录后进入 "企业API" 或 "API管理"
- 创建或复制 API Key

### 验证 API Key
```bash
python3 -m light-transfer-skill --check
```

## 🛠️ Advanced: Custom Node Mapping

If your RunningHub workflow uses different node IDs for the two image inputs, edit `scripts/runner.py`:

```python
# In submit_task method, adjust nodeInfoList:
node_info = [
    {"nodeId": "8", "fieldName": "image", "fieldValue": pic1_url},   # pic1 node
    {"nodeId": "9", "fieldName": "image", "fieldValue": pic2_url},   # pic2 node
]
```

Use the workflow's `/api/webapp/apiCallDemo?webappId={workflow_id}` to discover correct node IDs.

## 📝 Notes

- **Concurrency**: Default is 1. Higher concurrency may trigger rate limits. Use with caution.
- **Retries**: Upload failures retry up to `max_retries` (default 3) with exponential backoff.
- **Timeouts**: Single task timeout default 600s. Adjust via `task_timeout` if your workflow is slower.
- **Resume**: Always safe to use `--resume`; already completed combinations are skipped.
- **Simulate**: Creates empty placeholder files in output directory for quick pipeline testing.

## 🔗 Related Skills

- `runninghub` - Direct RunningHub API client
- `poll-task` - Smart polling logic used internally

## 📄 License

MIT

---

**Last updated**: 2026-03-28 (pure API version)
**Maintainer**: Claw助手
