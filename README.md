# Nature 本机科研工作台

安装程序，配置自己的模型和研究方向，再建立个人文献库。支持 Windows、macOS 原生运行；本地嵌入设备可选 CPU、CUDA 或 Apple MPS。LLM 推理使用你配置的远端服务。QQ 是可选入口，未配置 QQ 时网页照常运行。

## 安装

先安装 Python 3.11–3.13 和 [uv](https://docs.astral.sh/uv/getting-started/installation/)。需要批量 OA PDF 下载时安装 Node.js 22+。Mac 的 Python 必须支持 SQLite 扩展；`nature doctor` 会实际加载 FTS5 和 sqlite-vec 检查，可使用 Homebrew Python。

下载源码发行 ZIP，解压后在目录打开终端：

```powershell
# Windows PowerShell
.\install.ps1
.\.venv\Scripts\Activate.ps1
```

```bash
# macOS / Linux
bash install.sh
source .venv/bin/activate
```

也可以用 `uv sync --frozen --extra tools` 安装锁定环境。维护者构建好的 wheel 可通过 `python -m pip install "ai_skill_nature-0.1.0-py3-none-any.whl[tools]"` 安装；wheel 的依赖版本范围见 pyproject，严格复现使用源码发行包中的 uv.lock。

## 首次使用

```bash
nature init
nature doctor
nature serve
```

初始化会询问模型接口、模型名称、API 地址、Token、研究关键词及群白名单。打开 <http://127.0.0.1:8088/app/>。默认工作区是 `~/.nature-workbench/default`，程序不携带原作者论文、数据库、笔记或登录状态。

每条命令都可指定独立工作区：

```bash
nature --workspace "./my-research" init
nature --workspace "./my-research" doctor
nature --workspace "./my-research" serve
```

`--workspace` 放在子命令前。路径可包含空格或中文。多个工作区各自保存凭据、语料与产物；同时运行时应设置不同端口。

## 配置自己的模型、研究方向与 QQ

工作区 `.env` 保存模型/QQ 配置；`corpus.json` 保存关键词、期刊、日期和下载参数。更新配置后重启服务。Token 可隐藏输入：

```bash
nature config set OPENAI_API_KEY
nature config set OPENAI_MODEL your-model
nature config set OPENAI_BASE_URL https://your-provider.example/v1
nature config set RAG_DEVICE auto
nature doctor --live-llm
```

`doctor` 默认不请求 LLM；`--live-llm` 发起真实短请求，会使用自己的账户额度。模型接口支持 `openai`、`anthropic` 兼容服务和自行安装/登录的 `codex_cli`。不要将账号凭据提交到 Git。

`corpus.json` 的示例结构：

```json
{
  "contact_email": "you@example.org",
  "date_range": {"from": "2022-01-01", "to": "2026-10-03"},
  "download": {"timeout_sec": 60, "retries": 3, "rate_per_sec": 2, "workers": 2},
  "keyword_sets": {"research": ["battery recycling", "lithium recovery"]},
  "feeds": [{"group": "research", "journal": "", "kw": "research", "cap": 30}]
}
```

`journal` 留空表示不限定期刊；可增加 feed 配置多个期刊/关键词组。更换整个方向建议新建工作区，避免把旧库误当成新方向的覆盖。

QQ 需要自己安装并登录 NapCat，设置 HTTP API 和消息上报地址 `/onebot/event`，然后填写：

```bash
nature config set NAPCAT_ACCESS_TOKEN
nature config set ALLOWED_GROUPS 123456789,987654321
nature config set ADMIN_QQS 12345678
nature config set QQ_ENABLED 1
```

确认白名单后启用 QQ。跨机器 NapCat 不能直接读取本机文件路径；本版 QQ 文件工作流要求网关与 NapCat 共享可访问的文件系统。NapCat 的 macOS 本机登录尚未验证。核心 Mac 工作台与 MPS 不依赖 QQ；不将远程 QQ 桥接标为已完成能力。自启动可以在验证前台启动后，按自己的系统设置，原仓库 Windows 私有启动脚本不进入发行包。

## 下载自己的文献并建库

先预览，再下载：

```bash
nature corpus preview --source europepmc --limit 10
nature corpus build --source europepmc --limit 10 --vault
```

Europe PMC 流程适合其覆盖领域的 OA/JATS 全文；其他方向可选 OpenAlex 候选发现与现有 Node OA 下载器：

```bash
nature config set OPENALEX_API_KEY
nature corpus preview --source openalex --limit 10
nature corpus build --source openalex --limit 10 --vault
```

OpenAlex 的凭据、网络与配额以自己的账户为准。单次发现量受 `--limit` 限制；无全文时保留题录/摘要，报告不会把它们算成全文。下载器可能找到 HTML/XML；本版 OpenAlex 批量入库只处理有效 PDF，其他格式在 manifest 中保留记录。

也可以导入自己下载的 PDF：

```bash
nature corpus import "/path/to/paper.pdf"
nature corpus index --vault
```

导入复用 DOI 识别和冲突保护，不伪造缺失 DOI。`build` 会为尚未嵌入的段落建索引；`--vault` 额外生成引用边、标题摘要相似边、笔记与图谱。每次建库报告位于工作区 `nature_article/logs/<run-id>/report.json`，包含真实文件量、段落量、向量覆盖和逐篇缺口；`partial` 表示有缺口。重复运行按 DOI 去重。

## Mac MPS

`auto` 按 CUDA → MPS → CPU 选择可用设备；显式 `mps` 或 `cuda` 不可用时报告错误。检索、段落嵌入、论文相似边及 PDF 关联报告使用相同设备策略。默认小批次运行，可在 `.env` 设置 `RAG_BATCH_SIZE`。初次嵌入会下载 BGE-M3；缓存后可设置 `RAG_LOCAL_FILES_ONLY=1`。

MPS 真机兼容性以验收记录为准，模拟设备测试不等于真机通过。Mac 安装时须同时检查 SQLite 扩展和 PyTorch MPS，不能仅根据 `import torch` 成功判断。

## 能力与外部工具

网页复用现有文献检索、证据问答、润色/翻译/审稿/回复、PDF 入库、DOCX 修订和批注、任务中心、绘图及 Obsidian 工作流。引用和修订校验沿用已有实现；摘要与全文证据仍区分。

Nature Skills 与个人 Skill 的规则、模板、工具随包提供。需要外部 Agent、图像服务、OCR、R 或学术数据库凭据的工具仍需自行配置；网页能力目录继续显示其实现及依赖状态。图片转 PPTX、多源引用终审等预留流程不会因安装包存在就自动成为完整服务。

## 升级、备份与发行

停止服务和建库任务后备份整个工作区，包括 `.env`、`corpus.json`、`kb/`、论文、产物和手写笔记。保存 SQLite 数据库时同时保留同名 WAL/SHM 文件。升级程序不覆盖工作区；跨版本恢复先在副本运行 doctor 和检索验证。

维护者命令：

```bash
uv lock
uv build
python -m nature_workbench.release
python -m pytest tests
```

`dist/` 包含源码压缩包、wheel、干净 Git 源码 ZIP、文件清单及 SHA-256。发行文件由 `release_files.py` 白名单决定，包含当前工作目录中已完成的代码；不包含原仓库的 Git 历史。先检查清单，再把源码 ZIP 解压到新目录创建发行 Git 仓库。尚未自动推送到 GitHub/PyPI。

维护者另提供独立 Git bundle 时，可直接克隆，不需要 GitHub 地址：

```bash
git clone ai-skill-nature-0.1.0.bundle my-workbench
cd my-workbench
uv sync --frozen --extra tools
```

`.github/workflows/portable.yml` 提供 Windows/Linux/macOS 安装矩阵，并从源码目录外验证 wheel。CI 的 macOS 安装测试不能替代 Apple Silicon MPS 真机测试。原项目的一部分旧测试依赖个人稿件或直接脚本运行，维护者回归时应按其原运行方式执行，发行测试不依赖个人数据。

