# LLM 外调审阅与事实搜证工作流

> 诞生背景：审阅 OpenMAIC 英语课件方案时，V4-Flash 凭训练数据误判 PPTX 导出缺失。
> 根本原因：审阅者无实时搜索验证能力，只能凭训练数据推断。
> 本工作流将"我 vs 审阅者"的主观对抗，变成"证据卡 vs 审阅者"的客观校准。

---

## 完整流程 V3（含事实搜证）

```
Step 0         Step 1         Step 2         Step 3         Step 4         Step 5
声明提取  ──→  Hermes事实搜证  ──→  构造带证据Prompt  ──→  V4-Pro审阅  ──→  Hermes事后复核  ──→  自检验证
```

| Step | 动作 | 执行者 | 产出物 |
|------|------|--------|--------|
| **0** | 从方案/回答中提取待验证事实声明 | factcheck.py `--claims` | 结构化声明列表（EASY/MEDIUM/HARD） |
| **1** | Web搜索 + GitHub API 直查 + LLM判定 | factcheck-judge.py（Operit搜索→WSL判定） | Markdown 证据卡 + 事实校准段 |
| **2** | 将证据卡嵌入审阅Prompt的"已验证事实校准"段 | Operit（手动/脚本拼接） | 带搜证结果的完整审阅Prompt |
| **3** | DeepSeek-V4-Pro 深度审阅 | 脚本→process_start后台→写文件→轮询 | 审阅结果JSON |
| **4** | 对审阅者否认的声明，用 Hermes 搜证复核 | factcheck.py 单项验证 | 补充证据卡 |
| **5** | 人工+AI自检：审阅者是否误判、方案是否需修正 | Operit | 最终修正方案 |

---

## 核心组件

### 1. factcheck.py（全流程版）

- **路径**：`~/.hermes/skills/research/factcheck/factcheck.py`
- **功能**：声明提取 → 搜索 → LLM判定 → 证据卡
- **搜索引擎**：Bing + 百度 + DuckDuckGo（多引擎去重）
- **LLM判定**：DeepSeek-V4-Flash（温度0，500tokens）
- **输出**：Markdown证据卡 + 审阅用事实校准段

### 2. factcheck-judge.py（分离式，解决WSL网络问题）

- **路径**：`~/.hermes/skills/research/factcheck/factcheck-judge.py`
- **设计**：搜索在 Operit(Android) 侧执行，判定在 WSL 侧执行
- **原因**：WSL 受 localhost 代理限制，DDG/Bing 搜索质量差
- **用法**：
  ```bash
  # Step 1: Operit 侧搜索（various_search）
  # Step 2: 搜索结果写入 JSON 文件
  # Step 3: WSL 侧判定
  python3 factcheck-judge.py \
    --claims "声明1,声明2" \
    --search-results /path/to/results.json \
    --output /tmp/evidence.md
  ```

### 3. V4-Pro 审阅 SOP

| 决策 | 原因 | 实现方式 |
|------|------|----------|
| 输出写文件再读取 | V4-Pro 深度思考60-120s，超过 Agent 130s 超时 | `process_start` + `/tmp/` 写文件 + 轮询 `done` 标记 |
| Prompt 必须含事实校准段 | V4-Flash 曾凭训练数据误判PPTX导出缺失 | Step 1 证据卡输出 → 拼入 Prompt |
| 用后即删脚本 | API Key 硬编码在脚本中 | 审阅完成后 `rm -f` 清理 |
| V4-Flash 先跑一轮 | V4-Flash 快（~15s），先拿初步结果 | V4-Flash → 自检 → V4-Pro → 自检 |

**V4-Pro 审阅脚本模板**：
```bash
#!/bin/bash
API_KEY=$(grep DEEPSEEK_API_KEY ~/.bashrc | head -1 | cut -d'=' -f2 | tr -d '"' | tr -d "'")
OUTPUT="/tmp/deepseek-review-result.json"
DONE_MARKER="/tmp/deepseek-review-done"
rm -f "$OUTPUT" "$DONE_MARKER"

PROMPT='审阅Prompt内容...'

curl -s --max-time 300 https://api.deepseek.com/v1/chat/completions \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n \
    --arg prompt "$PROMPT" \
    '{model:"deepseek-chat",temperature:0.3,max_tokens:8000,messages:[{role:"user",content:$prompt}]}'
  )" \
  -o "$OUTPUT"

echo "DONE" > "$DONE_MARKER"
```

---

## 两轮审阅的关键发现

### V4-Flash 审阅（6/10）

| # | 问题 | 严重性 | 修正 |
|---|------|--------|------|
| 1 | 遗漏 AGPL 合规风险 | 🔴 Blocker | 补充 AGPL 传染性警告 |
| 2 | PPTX 导出支持被错误否定 | 🔴 误判 | 凭训练数据推断，搜证后驳回复原 |
| 3 | L2/L3/L4 合并方式不清晰 | 🟡 | 明确 L2=语法+词汇 L3=题目 L4=TTS |
| 4 | 时间估算不够精确 | 🟡 | 细化到分钟级 |
| 5 | 5%标红阈值缺乏依据 | 🟡 | 补充校准方案 |
| 6 | 成本模型缺失 | 🟡 | 补充 token 估算 |
| 7 | 缺少回滚方案 | 🟡 | 补充 Phase 0 |

### V4-Pro 审阅（7.4/10）

| # | 问题 | 严重性 | 修正 |
|---|------|--------|------|
| 1 | **Phase 1 目标与"课件工厂"定位断裂** | 🔴 Blocker | Phase 1 应聚焦"课件工厂"输出质量，不是"学生体验" |
| 2 | **5%标红阈值逻辑闭环缺陷** | 🔴 Blocker | 需独立外部抽检，不能自审自 |
| 3 | **内容安全风险完全未识别** | 🔴 Blocker | AI可能生成不当内容，需安全审核层 |

### 核心教训

> **V4-Flash 误判 PPTX 导出缺失**这一事件，直接催生了整个"事实搜证前置层"的搭建。
> 审阅者没有搜索能力，只能凭训练数据推断——训练数据中没有的信息会被默认否定。
> 事实校准段解决了这个问题：V4-Pro 审阅时因为有搜证证据，不再犯同类错误。

---

## 已知限制与改进

| 限制 | 现状 | 改进方向 |
|------|------|----------|
| DDG 在 WSL 环境搜索质量差 | 中文搜索结果相关度低，常返回 INSUFFICIENT_EVIDENCE | ✅ 已解决：分离式架构（Operit搜索→WSL判定） |
| scrapling 未安装 | PEP 668 限制 | 使用 venv 或 `--break-system-packages` |
| GitHub API 无 Token 限流 | 匿名 60次/小时 | 配置 Hermes 已有的 GITHUB_TOKEN |
| Bing HTML 抓取结果质量 | WSL 环境下受限 | ✅ 已解决：Operit various_search 替代 |
| 百度搜索反爬虫 | HTML 抓取易被封 | 改用 Bing API 或 Operit various_search |

---

## 分离式架构详解

```
┌──────────────────┐         ┌──────────────────┐
│  Operit (Android) │         │    WSL (PC)       │
│  各种搜索引擎     │  JSON   │  factcheck-judge  │
│  various_search   │ ──────→ │  DeepSeek LLM判定  │
│  Bing/Baidu/DDG   │         │  GitHub API       │
└──────────────────┘         └──────────────────┘
```

**优势**：
1. Android 端网络无代理限制，搜索质量高
2. WSL 侧只需调 DeepSeek API（走 HTTPS，不受代理影响）
3. GitHub API 走 HTTPS，同样不受代理影响
4. 架构清晰：搜索与判定解耦，可独立优化

---

## 产出物清单

| 产出物 | 位置 | 状态 |
|--------|------|------|
| factcheck SKILL.md | `~/.hermes/skills/research/factcheck/SKILL.md` | ✅ 完成 |
| factcheck.py (v2) | `~/.hermes/skills/research/factcheck/factcheck.py` | ✅ 完成，含 Bing/百度搜索 |
| factcheck-judge.py | `~/.hermes/skills/research/factcheck/factcheck-judge.py` | ✅ 完成，分离式架构 |
| duckduckgo-search 依赖 | WSL Python 全局安装 | ✅ 已安装 |
| V4-Pro 审阅 SOP | 本文档第三节 | ✅ 可复用 |
| OpenMAIC 本地实例 | WSL `/tmp/OpenMAIC` | ✅ 运行中 (port 3000) |

---

## 快速参考

### 一键执行事实搜证（分离式）
```bash
# Step 1: Operit 侧搜索
# 使用 various_search 搜索声明

# Step 2: 搜索结果写入 JSON
# 格式: {"声明1": [{"title":"...", "url":"...", "snippet":"..."}], ...}

# Step 3: WSL 侧判定
python3 ~/.hermes/skills/research/factcheck/factcheck-judge.py \
  --claims "声明1,声明2,声明3" \
  --search-results /path/to/results.json \
  --output /tmp/evidence.md
```

### V4-Pro 审阅快速模板
```bash
# 写审阅脚本到 bridge 目录
# process_start 后台启动
# 轮询 /tmp/deepseek-review-done 标记
# 读取 /tmp/deepseek-review-result.json
# 自检 → 修正
# rm -f 清理临时脚本和结果文件
```

---

*相关文档*：[[OpenMAIC评估]] | [[英语课件审核Pipeline]] | [[DeepSeek外调审阅SOP]]

*标签*：#工作流 #事实搜证 #LLM审阅 #DeepSeek #Hermes #AI教育
