# AIOps 故障诊断和修复 Agent

> **项目简介**：基于 **Claude Agent SDK** 的 AIOps 故障诊断与自愈系统。告警进来，Agent 用 kubectl/curl/git 这些**真实 CLI** 去连真实可观测栈（Prometheus/Jaeger）与真实公开 GitHub 代码仓，完成「告警 → 根因诊断 → 风险分级处置 → 自动修复」全链路闭环：低风险线上操作自动止血、高风险操作发飞书卡片交人工、代码 bug 自动改码提 PR。诊断过程带 Skill 排查手册按需加载 + Agentic RAG 历史工单语义检索，每次诊断沉淀进 Milvus 向量库形成经验积累。这不是"假装调用工具"的 demo，而是**真机跑通**的端到端 Agent。

把告警变成**根因诊断 + 分级处置 + 自动修复**：低风险线上操作（滚动重启/迁移/抬高资源上限单个工作负载）Agent 自动执行止血、高风险操作（redis/db/删除/改集群/缩容）发飞书卡片交人工，代码 bug 类自动改码并提 PR（人工评审合并），混合根因两条腿都走。「能自动执行哪些操作」由代码层一份可枚举的白名单硬控（不靠提示词）。基于 **Claude Agent SDK** 单引擎、两趟、按风险分级处置。配套教学文档见飞书《[AIOps-Agent-简历项目教学文档](https://zcnii5t1bwjj.feishu.cn/docx/LJSUdmaZrop80UxCGg9ch986nRg)》。

**运行环境：以 Kubernetes 为主**——生产运维基本盘就是 k8s，诊断 Agent 的观测（`kubectl get/describe/top`）与自动止血（`kubectl rollout restart` / `delete pod` / `set resources`）都以 kubectl 为主形态。为了让学员**零门槛先跑通**，默认后端是 **docker**（一台机器 `docker compose up` 即可，无需集群）；想体验生产形态就 `export AIOPS_BACKEND=k8s` 并起一个 kind 集群（见 [快速开始](#快速开始)）。两套后端语义一一对应，切换只改一个环境变量，代码其余部分无感知。

已完成从告警到 PR 的**完整端到端真实闭环验证**（内存泄漏 capstone）：真工作负载跑到 OOM 重启、Agent 实地观测定位到发版 commit、改成有界结构、测试从红转绿、提出一个推不上 master 的 PR。

```
告警 → 【故障诊断处置 Agent(只读诊断+低风险自动处置)】定位根因 + 给方案
        ├─ online_op 低风险(滚动重启/迁移Pod/抬资源上限,命中白名单) → Agent 自动执行止血 → 飞书卡片知会人工
        ├─ online_op 高风险(回滚版本/redis/db/删除/改集群/缩容)   → 飞书卡片(HITL) → 人工执行
        │    └─ + also_code_fix (混合根因)   → 上面止血 **并** 触发 代码修复 Agent 提 PR 根治(双走)
        ├─ code_fix  (代码 bug)              → 【代码修复 Agent(写)】clone→改→build+test→提PR
        ├─ info_only                         → 直接报告
        └─ confidence < 阈值                  → 飞书卡片，降级人工
诊断完成后 ─────────────────────────────────→ 编排层把工单向量化存入 Milvus (经验沉淀)
```

## 核心能力

- **双 Agent 分级诊断-修复系统**：同一个 Claude Agent SDK 引擎跑两趟——故障诊断处置 Agent（只读排查 + 低风险自动止血）→ 代码修复 Agent（改码 + build+test + 提 PR），按诊断产出的 `remediation_type` 分流；混合根因走「先止血再根治」双走。
- **工具层风险分级白名单**：「能自动执行哪些线上操作」由代码层一份**可枚举白名单**硬控（锚定正则 + 目标授权），低风险（滚动重启/删 Pod 重建/抬资源上限）自动放行，高风险（动 redis/db/删除/改集群/缩容）一律拦截交人工。
- **Skill 排查手册按需加载**：内存泄漏/OOM、依赖故障/5xx、CPU 打满、队列积压四类故障的 SRE 排查方法论沉淀为 4 份 Agent Skills，按告警语义自动加载（省 token）。
- **Memory 动态沉淀 + Agentic RAG**：每次诊断向量化沉淀为工单存入 Milvus（BGE 中文 embedding），封装为 in-process MCP 工具挂给诊断 Agent，由模型自主决定何时检索——真正的 Agentic RAG，支持跨服务故障模式迁移。
- **纵深防御安全边界**：allowed_tools 白名单 + disallowed_tools 黑名单 + PreToolUse hook 三层防御；执行记录以 hook 台账为准；代码修复强制「build+test 通过才提 PR」。
- **真实端到端闭环**：不是 mock——Agent 用 kubectl/curl/git 连真实可观测栈（Prometheus/Jaeger）和真实公开 GitHub 仓，真的自动止血、真的提出可通过 build+test 的 PR。

## 目录结构

```
AIopsAgent/
├── agent/                     # 核心实现（编排 + 两个 Agent + 安全 + 集成）
│   ├── run.py                 # 编排入口：告警 → 去重 → 诊断 → 分流 → 报告（先读这个）
│   ├── config.py              # 集中配置，全部可用环境变量覆盖
│   ├── agents/
│   │   ├── diagnose.py        # 故障诊断处置 Agent：只读排查 + 根因定位 + 低风险自动处置
│   │   ├── code_fix.py        # 代码修复 Agent：clone → 改码 → build+test → 提 PR
│   │   └── prompts.py         # 两个 Agent 的系统/用户提示词
│   ├── core/                  # 运行底座（两个 Agent 共用）
│   │   ├── sdk_runner.py      # Claude Agent SDK 薄封装 + 成本/超时/降级控制
│   │   ├── schema.py          # pydantic 结构化输出（Diagnosis / FixResult / ExecutedAction）
│   │   ├── remediation.py     # 低风险操作白名单（可枚举）+ 命令分级 + 执行台账
│   │   └── hooks.py           # PreToolUse 安全 hook（放行/拦截/台账）
│   └── integrations/          # 外部系统对接
│       ├── ingest.py          # 告警接入 + 指纹幂等去重
│       ├── feishu.py          # 飞书交互卡片（HITL 落地）
│       ├── memory.py          # 工单向量化存 Milvus + BGE 语义检索
│       └── rag_tool.py        # Agentic RAG 工具 search_past_incidents（in-process MCP）
├── .claude/skills/            # 4 份排查手册（memory-oom / dependency-error / resource-cpu / queue-backlog）
├── alerts/                    # 13 条告警 fixture（s1~s12 + s_lowconf），评测场景输入
├── eval/                      # 评测 harness
│   ├── run.py                 # 单次评测（比对 expected.json）
│   ├── multi_run.py           # 多次运行聚合，落盘 JSONL
│   ├── aggregate.py           # 聚合出 metrics.json（7 条效果指标）
│   ├── expected.json          # 13 场景的 gold 期望（人工审校）
│   └── metrics.py             # 指标定义
├── reports/                   # 评测报告与 metrics.json（Opus 基线 / Qwen 三档）
├── scripts/                   # demo 栈与故障注入脚本
│   ├── fetch-otel-demo.sh     # 拉取 OTel Demo compose 到 vendor/
│   ├── inject.sh              # 故障注入（flagd 开关 / 泄漏镜像）
│   ├── seed-github.sh         # 建/刷新公开上游 recommendation 仓 + 分支保护
│   ├── start-milvus.sh        # 仅起 Milvus 向量库
│   └── k8s/                   # k8s 后端：start-kind.sh + recommendation-leak.yaml
├── tests/                     # 单元测试
├── docker-compose.yml         # OTel Demo + Milvus（etcd+minio+milvus）
├── docker-compose.override.yml# 固定 flagd/jaeger 端口 + 评测期内存上限修正
├── .env.example               # 全部环境变量样例
└── requirements.txt
```

> `vendor/otel-demo/` 由 `scripts/fetch-otel-demo.sh` 拉取，`docker compose` 通过 `include:` 引用，不随仓库提交。

## 环境依赖

**Python 包**（`requirements.txt`）：

| 包 | 用途 |
|---|---|
| `claude-agent-sdk>=0.1.0` | Agent 引擎（走本机已登录的 `claude` CLI，无需 API key） |
| `pydantic>=2.0` | 结构化输出数据模型 |
| `python-dotenv>=1.0` | 加载 `.env` |
| `pymilvus>=2.4` | 诊断工单向量库（Milvus 客户端） |
| `sentence-transformers>=2.2` | BGE 本地 embedding |
| `modelscope>=1.9` | 国内友好的 BGE 模型下载（HF 国内不可达） |

**运行时**：
- **Python 3.11+**；鉴权走本机**已登录的 `claude` CLI**（Node + claude CLI）。
- **docker 后端（默认，零门槛）**：Colima/Docker + standalone `docker-compose`；`docker compose up -d` 拉起 OTel Demo 全栈（Prometheus/Jaeger/Grafana/flagd）+ Milvus 三容器。
- **k8s 后端（生产形态，可选）**：本地 kind 集群（含 metrics-server，`scripts/k8s/start-kind.sh`），`export AIOPS_BACKEND=k8s` 切换。
- **国内镜像**：docker.io 走 registry-mirror；ghcr 走 `ghcr.nju.edu.cn`；quay 用代理前缀；BGE 模型走 ModelScope 缓存（详见「本机环境备注」）。

## 快速开始（Quick Start）

鉴权走本机**已登录的 `claude` CLI**（需已装 Node + claude CLI 并登录），无需配 `ANTHROPIC_API_KEY`。分两档，按想要的"含金量"选。

```bash
# 安装（Python 3.11+）
cd AIops-agent
python3.11 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # FEISHU/GITHUB 等按需填；不填也能本地跑通逻辑
```

**档位 A：快速逻辑演示（1 分钟，无需 Docker）** —— 不依赖真实监控栈，Agent 据告警文本 + 排查手册 Skill 给出诊断，飞书卡片直接打印到终端：

```bash
source .env
python3 -m agent.run --alert alerts/s2.json --json-out reports/s2.json
# 终端打出诊断过程 + 路由决策；reports/s2.json 为结构化结果
```

**档位 B：全真栈端到端闭环（招牌 capstone）** —— 真跑内存泄漏服务到 OOM，Agent 实地观测→定位泄漏 commit→改码→build+test→提 PR（需 Colima/Docker，提 PR 还需配 `GITHUB_TOKEN`，详见 [下方完整快速开始](#快速开始)）：

```bash
./scripts/fetch-otel-demo.sh        # 拉 OTel Demo compose 到 vendor/
docker compose up -d                # OTel Demo + Prometheus + Jaeger + Milvus
./scripts/inject.sh s4              # 注入内存泄漏（等 1-2 分钟让内存爬升）
python3 -m agent.run --alert alerts/s4.json
```

> 完整的后端切换（docker / k8s）、GitHub fork 配置、Milvus 记忆等见 [下方「快速开始」详解](#快速开始)。

## 架构

代码按职责分三层，先读 `agent/run.py`（总控），再顺着分层往下看：

```
agent/
├── run.py              编排入口：把告警一路串到最终报告（最适合先读）
├── config.py           集中配置（全部可用环境变量覆盖）
├── agents/             ← 两个 Agent 主体（业务核心）
│   ├── diagnose.py     故障诊断处置 Agent：只读排查 + 定位根因 + 低风险自动处置
│   ├── code_fix.py     代码修复 Agent：改码 + build+test + 提 PR
│   └── prompts.py      两个 Agent 的提示词
├── core/               ← 运行底座（两个 Agent 共用）
│   ├── sdk_runner.py   对 Claude Agent SDK 的薄封装 + 成本/超时控制
│   ├── schema.py       结构化输出数据模型（Diagnosis / FixResult / ExecutedAction）
│   ├── remediation.py  低风险线上操作白名单（可枚举）+ 命令分级判定 + 执行台账
│   └── hooks.py        安全 hook：诊断按白名单放行低风险/拦其余、修复禁动线上
└── integrations/       ← 外部系统对接
    ├── ingest.py       告警接入 + 指纹幂等去重
    ├── feishu.py       飞书交互卡片
    ├── memory.py       诊断工单向量化存 Milvus + BGE 语义检索
    └── rag_tool.py     Agentic RAG 工具 search_past_incidents（只给 故障诊断处置 Agent）
```

| 层 | 模块 | 职责 |
|---|---|---|
| 编排 | [`agent/run.py`](agent/run.py) | 接入告警 → 幂等去重 → 故障诊断处置 Agent → 低置信度逃生口 → 路由 → report（CLI 入口） |
| 配置 | [`agent/config.py`](agent/config.py) | 集中配置：阈值/成本上限/各服务地址/凭证，全可环境变量覆盖 |
| 故障诊断处置 Agent | [`agent/agents/diagnose.py`](agent/agents/diagnose.py) | 只读调查 + schema 校验 + 重试 + 兜底降级 |
| 代码修复 Agent | [`agent/agents/code_fix.py`](agent/agents/code_fix.py) | clone 仓 → 改码 → build+test → 提 PR |
| 提示词 | [`agent/agents/prompts.py`](agent/agents/prompts.py) | 两个 Agent 的系统提示与用户提示 |
| SDK 封装 | [`agent/core/sdk_runner.py`](agent/core/sdk_runner.py) | `query()` + max_turns/超时/预算/usage |
| 结构化输出 | [`agent/core/schema.py`](agent/core/schema.py) | pydantic `Diagnosis` / `FixResult` + JSON schema |
| 安全 hook | [`agent/core/hooks.py`](agent/core/hooks.py) | PreToolUse 按白名单放行低风险 / 拦其余写变更 / 修复禁线上变更 |
| **处置白名单** | [`agent/core/remediation.py`](agent/core/remediation.py) | 可枚举的低风险线上操作（k8s/docker 两套，锚定正则 + 目标授权）+ 命令分级 + 执行台账 |
| ingest | [`agent/integrations/ingest.py`](agent/integrations/ingest.py) | 读告警 + 指纹幂等去重（内存 TTL） |
| 飞书 | [`agent/integrations/feishu.py`](agent/integrations/feishu.py) | 交互卡片（未配 webhook 时打印到 stdout） |
| **记忆/RAG** | [`agent/integrations/memory.py`](agent/integrations/memory.py) | 诊断工单向量化存 Milvus + BGE 语义检索历史工单（失败降级 no-op） |
| **Agentic RAG 工具** | [`agent/integrations/rag_tool.py`](agent/integrations/rag_tool.py) | in-process MCP 工具 `search_past_incidents`，故障诊断处置 Agent 自主检索历史相似工单 |
| Eval | [`eval/run.py`](eval/run.py) | 比对 `expected.json`，记录路由/成本/降级 |

## 安全边界

- **诊断只读 + 低风险白名单**：故障诊断处置 Agent `allowed_tools=[Bash,Read,Grep,Glob]` + PreToolUse hook。写/变更命令默认全拦（`kubectl apply/scale/exec`、`docker rm`、`redis-cli`、`git push`、`rm -rf`…）；只有命中 [`remediation.py`](agent/core/remediation.py) 里**可枚举的低风险白名单**才放行自动执行。白名单按后端各一套、语义对应：k8s 为 `kubectl rollout restart deploy/<t>` / `kubectl delete pod -l app=<t>` / `kubectl set resources deploy/<t> --limits=...`，docker 为 `docker restart` / `compose up --force-recreate` / `docker update`；均为锚定正则 `^…$` + 目标授权。锚定杜绝 `&&`/`;`、以及藏在 `-n <ns>` flag 里的命令注入绕过。
- **风险分级处置**：低风险操作 Agent 自动执行（记入执行台账，事后发卡知会人工复核）；高风险操作（回滚版本/redis/db/删除/改集群/缩容/动有状态组件）绝不自动执行，只发卡交人工。「执行了什么」以台账为准（`AIOPS_REMEDIATION_ENABLED=0` 可一键退回纯只读）。
- **代码修复**：代码修复 Agent 仅推特性分支 + 提 PR；master **分支保护** + PR 人工评审；hook 禁止线上变更/强推/推 master。
- **build+test 通过才提 PR**：未通过则降级飞书卡片。
- **低置信度转人工**：`confidence < CONFIDENCE_THRESHOLD`（默认 0.6）。
- **成本上限**：每次 `query()` 设 `max_turns` / `asyncio.timeout` / `max_budget_usd`。
- **隔离**：故障诊断处置 Agent 为加载 Skill 开 `setting_sources=["project"]`（仅项目作用域）；`CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`（用自建 Milvus 记忆，不用 SDK auto-memory）、独立 cwd。

## 记忆 + Agentic RAG + Skill

让 Agent 从"每次从零诊断"升级为"带经验积累、能跨服务迁移相似故障模式"的运维 Agent。

- **Agent Skills**：4 个排查手册（内存/依赖/CPU/队列）以 `.claude/skills/<name>/SKILL.md` 形式存在，故障诊断处置 Agent 按告警类型**按需自动加载**（比一次性全塞进系统提示省 token）。开关：`skills="all"` + `setting_sources=["project"]`。
- **记忆（Milvus 向量库）**：每次诊断完，编排层把工单（告警+根因+证据+处置）用 **BGE 中文 embedding** 向量化，存进 **Milvus**（`agent/integrations/memory.py`）。写入在编排层做，不破坏 故障诊断处置 Agent 只读边界。
- **Agentic RAG**：历史工单检索做成 故障诊断处置 Agent 的 **in-process MCP 工具** `search_past_incidents`（`agent/integrations/rag_tool.py`）——Agent 自主决定何时检索、查什么（真 agentic，而非编排层强制注入）。工具只读 Milvus。
- **优雅降级**：`AIOPS_RAG_ENABLED=0` 或 Milvus/模型不可用时，记忆子系统整体降级为 no-op，**不影响核心诊断闭环**（与"流程永不崩溃"一致）。

技术栈：Milvus standalone（etcd+minio+milvus 三容器）+ BGE（`BAAI/bge-small-zh-v1.5`，本地离线，ModelScope 下载）+ 自定义 MCP 工具 + Agent Skills。

```bash
# 起 Milvus（仅向量库,不含 OTel 全栈）
./scripts/start-milvus.sh
# 下载 BGE 模型（国内走 ModelScope,一次即可）
modelscope download BAAI/bge-small-zh-v1.5
```

## 快速开始

### 1. 安装
```bash
python3.11 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # FEISHU/GITHUB 等按需填；不填也能本地跑通逻辑
```
> 鉴权走本机已登录的 `claude` CLI（需已安装 Node + claude CLI 并登录），无需单独配 `ANTHROPIC_API_KEY`。

### 2. 不依赖真实栈先跑通逻辑
未配飞书 webhook 时，卡片直接打印到 stdout；故障诊断处置 Agent 会尝试用 curl 连 Prometheus/Jaeger，连不上则据告警文本与排查手册 Skill 给出（较低置信度的）诊断。
```bash
source .env  # 导出环境变量
python3 -m agent.run --alert alerts/s2.json --json-out reports/s2.json
```

### 3. 全真栈端到端闭环

被修复的目标服务仓托管在**公开 GitHub**（真实开源协作流），学员各自 fork 后由 Agent 提跨仓 PR，无需被邀请为协作者。

**（老师，一次性）** 建/刷新公开上游仓：
```bash
./scripts/seed-github.sh            # 用 gh 建公开 recommendation 仓 + 多 bug 历史（基线 + 每个 bug 一个
                                    # commit：ranking/dedupe/stats/pagination/cache/pricing + 内存泄漏 +
                                    # BUGS.md）+ master 分支保护。所有 bug 都在 master 上，学员 clone 即见
                                    # 默认上游：github.com/fielharald602-cmyk/recommendation
```

**（学员）** fork 上游仓 → 配自己的 token：
```bash
# 1) 在网页把 https://github.com/fielharald602-cmyk/recommendation Fork 到自己账号
# 2) https://github.com/settings/tokens 建 PAT（勾 public_repo）
# 3) 在 .env 里填：
#    GITHUB_TOKEN=<你的 PAT>
#    GITHUB_UPSTREAM_OWNER=HuaiNan54321   GITHUB_REPO=recommendation
#    GITHUB_FORK_OWNER=<你的 GitHub 用户名>
```

**跑闭环（默认 docker 后端，零门槛）：**
```bash
./scripts/fetch-otel-demo.sh        # 拉取 OTel Demo compose 到 vendor/
docker compose up -d                # OTel Demo + Prometheus + Jaeger + Milvus
source .env                         # 导出 GITHUB_* 等环境变量

# 内存泄漏端到端闭环（capstone）
./scripts/inject.sh s4                       # 从泄漏 commit 真 build 镜像 → docker run --memory=128m，内存开始单调上升
# 等 1-2 分钟让内存爬升：docker stats recommendation / curl localhost:8080/metrics
python3 -m agent.run --alert alerts/s4.json   # 故障诊断处置 Agent 实地观测泄漏→定位 commit→代码修复 Agent build+test→提 PR
# → 你的 fork 上出现修复分支，并向上游仓发起一条跨仓 PR（推不上上游 master、含 build+test 验证摘要）
```

**跑闭环（k8s 后端，生产形态，可选进阶）：**
```bash
# 1) 起一个本地 kind 集群（含 metrics-server，让 kubectl top 有数据）
./scripts/k8s/start-kind.sh
export AIOPS_BACKEND=k8s AIOPS_K8S_NAMESPACE=otel-demo
source .env

# 2) 注入 s4：build 泄漏镜像 → kind load → kubectl apply Deployment(memory limit 128Mi)
./scripts/inject.sh s4
# 等 1-2 分钟让内存爬升：kubectl top pod -l app=recommendation -n otel-demo
python3 -m agent.run --alert alerts/s4.json   # 诊断 Agent 用 kubectl describe/top 观测 OOMKilled+重启→定位 commit→提 PR
# 混合根因下会先 kubectl rollout restart 授权工作负载止血、再提根治 PR → route=auto_remediated_and_code_fix_pr
```

内存泄漏服务是自带的轻量 Python 服务（`scripts/fixtures/recommendation-master/`，纯 stdlib，自驱负载使内存真实爬升），不依赖 OTel demo 镜像。`seed-github.sh` 保证 **GitHub 仓码 = 运行的工作负载码 = 代码修复 Agent 的修复目标** 三方一致，构成逻辑自洽的真实闭环。

### 端到端闭环实测（Colima 全真栈，docker 后端）

故障诊断处置 Agent 用 `docker stats`/`docker inspect`（k8s 后端则是 `kubectl top`/`kubectl describe`）实地观测到真实运行的 recommendation 工作负载内存从 ~15MiB 单调爬升趋向 128MiB 上限 → 触碰上限被 OOMKilled → 重启（RestartCount=1）；据镜像 tag + `deploys.log` + `git show` 定位泄漏 commit（confidence 0.9，诊断全程只读）；代码修复 Agent clone 仓改有界 deque、`pytest`（含 `test_memory_is_bounded`）由失败转 2 passed → 向上游仓提出 GitHub PR（推不上 master，verified=true）。诊断与修复两半都基于真实运行的服务。

## Eval
```bash
python3 -m eval.run --dry-run        # 只校验 fixture（不调用 Agent）
python3 -m eval.run                  # 全部
python3 -m eval.run --only s2 s4     # 跑指定子集，输出 通过/成本/降级
```

## 评测指标（Eval）

在**自建 13 场景告警回归集**（场景矩阵覆盖 9 条业务路由，含重复告警、低置信度、混合根因、修复失败、三条独立代码定位场景）上多次运行聚合（`eval/multi_run.py` + `eval/aggregate.py` → `reports/metrics.json`），关键口径：

- **7 条效果指标**：故障服务定位准确率、故障类型识别准确率、路由决策准确率、代码修复成功率（含 build+pytest 验证）、问题代码文件级定位准确率、MTTR（p50）、PR 提交成功率。
- **评价原则**：不测"精确输出"而测"关键决策"（措辞变化不影响判定）；`kind_any_of` / `route_any_of` 容忍合理模糊空间；gold 集（`eval/expected.json`）全部人工审校，评测代码不做自动生成 gold。

**真实运行结果**（`reports/` 下 metrics.json，模型档位对比）：

| 配置 | service | kind | route | fix_success | code_loc | mttr_p50 (min) | pr_rate |
|---|---|---|---|---|---|---|---|
| **基线（Full，13 场景 23 次运行，Sonnet 4.5）** | 0.8696 | 0.8261 | 0.7826 | 1.000 | 1.000 | 3.258 | 1.000 |
| A：关 Skill | 0.8261 | 0.7826 | 0.7391 | 1.000 | 1.000 | 5.900 | 1.000 |
| B：关指纹去重 | 0.8696 | 0.8261 | 0.6957 | 1.000 | 1.000 | 3.300 | 1.000 |
| C：Haiku 兜底 | 0.6957 | 0.6522 | 0.6087 | 0.7143 | 0.600 | 1.800 | 1.000 |

> 完整配置下 MTTR p50 = 3.26 分钟；对比 Google SRE Book / DORA 报告行业中位数（30–60 分钟），约加速 **9–18×**（用 low 端算约 9.2 倍）。行业数据为**引用公开均值**，非本项目内人工对照实验——话术口径见教学文档。

**训练侧三档对比**（`reports/` 下 Qwen 档，由 [`aiops-agentic-rl`](../aiops-agentic-rl/) 仓库训出的 checkpoint 评测）：zero-shot 25 rows = 0.48/0.52/0.36；v9-SFT 37 rows = 0.5676/0.6757/0.4595；GRPO step15 25 rows = **0.80/0.72/0.64**；Opus 参照 23 rows 全线约 0.913。

## 配置（环境变量，见 `.env.example`）
`AIOPS_BACKEND`（`docker` 默认 / `k8s`）、`AIOPS_K8S_NAMESPACE`（默认 `otel-demo`）、`AIOPS_REMEDIATION_ENABLED`、`AIOPS_REMEDIATION_TARGETS`、`FEISHU_WEBHOOK_URL`、`GITHUB_TOKEN/UPSTREAM_OWNER/REPO/FORK_OWNER`、`PROMETHEUS_URL`、`JAEGER_URL`、`CONFIDENCE_THRESHOLD`、`DIAGNOSE_MAX_TURNS/TIMEOUT_S`、`FIX_MAX_TURNS/TIMEOUT_S`、`AIOPS_MODEL`、`AIOPS_RAG_ENABLED`、`MILVUS_URI`、`AIOPS_EMBEDDING_MODEL`。

## 本机环境备注（Colima + 国内镜像）
- **后端选择**：默认 docker（本地零门槛）；生产形态用 `AIOPS_BACKEND=k8s` + 本地 kind 集群（`scripts/k8s/start-kind.sh`）。文档以 k8s 为主线，docker 是等价的本地默认实现。
- Docker 运行时用 **Colima**：`colima start --cpu 6 --memory 12 --disk 60`；compose 用 standalone `docker-compose`。kind 也跑在 Colima 的 docker 之上。
- 国内直连 docker.io/ghcr.io/quay.io 超时，已配镜像：Colima daemon.json 加 `registry-mirrors`（daocloud/1panel，仅 docker.io）；ghcr 镜像走 `ghcr.nju.edu.cn`，quay 用 docker.io 等价镜（`vendor/otel-demo/.env` 已改，备份 `.env.bak`）。
- Colima 下 collector 的 `docker_stats` receiver 会崩（Docker API 太旧），已从 `otelcol-config.yml` metrics pipeline 移除；内存观测走 `docker stats`/`docker inspect` + 服务自身 `/metrics`。生产换 cAdvisor/kube-state-metrics 即回 PromQL 路径。
- BGE 模型走 ModelScope（HuggingFace 国内不可达），缓存在 `~/.cache/modelscope`，离线可加载。

## 关键模块说明

| 模块 | 一句话职责 | 阅读入口 |
|---|---|---|
| `agent/run.py` | 编排总控：去重 → 诊断 → 低置信度逃生口 → 按 `remediation_type` 分流 → 写工单存记忆 | 先读它 |
| `agent/core/sdk_runner.py` | SDK `query()` 薄封装：max_turns / 超时 / 预算 / usage，失败一律转 degraded 不抛异常 | 理解「流程永不崩溃」 |
| `agent/core/schema.py` | `Diagnosis`/`FixResult`/`ExecutedAction` pydantic 模型 + 手写 JSON Schema（`additionalProperties: False`） | 结构化输出的唯一事实源 |
| `agent/core/remediation.py` | 可枚举低风险白名单（k8s/docker 两套，锚定正则 + 目标授权）+ 命令分级 + 执行台账 | 安全设计的核心 |
| `agent/core/hooks.py` | PreToolUse hook：白名单命中放行并记台账 / 其余写变更 deny | 安全闸门实现 |
| `agent/agents/diagnose.py` | 诊断 Agent 主流程：Skill 加载、工具调用、schema 校验重试、兜底降级 | 双 Agent 之一 |
| `agent/agents/code_fix.py` | 修复 Agent 主流程：clone 仓 → 改码 → build+test → 分支 → PR | 双 Agent 之二 |
| `agent/integrations/memory.py` | 工单向量化存 Milvus + BGE 检索，失败静默 no-op | 经验沉淀 |
| `agent/integrations/rag_tool.py` | in-process MCP 工具 `search_past_incidents` | Agentic RAG |
| `eval/run.py` | 跑场景比对 `expected.json`，输出通过/成本/降级 | 复现评测数字 |
| `scripts/inject.sh` | 故障注入：flagd 开关 / 泄漏镜像构建与运行 | 造故障 |

## 测试

```bash
python3 -m pytest tests/            # 单元测试（schema 等）
```

> 说明：本仓库以**端到端 eval 场景**为主要验证手段（`eval/`），单元测试覆盖结构化输出等核心数据模型。`tests/` 下与 AIops-agentic-rl 各子目录一样存在同名 `conftest.py` 冲突坑——不同目录的测试请分开跑。

## 与 aiops-agentic-rl 的关系

[`aiops-agentic-rl/`](../aiops-agentic-rl/) 是本项目（应用侧）的**训练侧延展仓库**：用 Cold Start SFT + GRPO 把「故障诊断处置 Agent」的决策底座从 Claude Opus 换成自训 Qwen3.5-9B + LoRA。它以 **git submodule** 形式只读消费本仓库的 SDK 接口与判定信号（hook allow/deny、schema 校验、白名单命中），并在其 `eval/run.py` + 13 个真实 docker 场景上完成三档对比评测。诊断出的 checkpoint 可通过 `AIOPS_MODEL` / `AIOPS_LLM_BASE_URL` 环境变量切回本仓库使用（见 `.env.example` 中 AIOPS_LLM_* 注释块）。