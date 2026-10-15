# LLM-Agent-MCP-VehicleDialogue

面向智能座舱的 **LLM 任务型多轮对话 Agent**：把用户的一句语音分流到「车控技能」或「闲聊百科」两条链路，前者走意图召回 + Function Calling + MCP 工具闭环，后者走拒识门禁 + 流式回答。

设计主线是**在延迟约束下做分层决策**：高频、确定性的判断交给本地微调的小模型，结构化工具调用和自然语言生成交给云端大模型。

---

## 核心链路

```
用户 query
   │
   ├─ ① Query 改写（串行前置，豆包 + Redis 历史）
   │     指代消解 / 信息补全：「介绍下他」→「介绍下汪苏泷」
   │
   ├─ ② 五路投机并行（ThreadPoolExecutor）
   │     仲裁 │ NLU │ 拒识 │ 相关性 │ 闲聊预热
   │
   ├─ ③ 先等仲裁结果（流式首 Token 早停）
   │
   ├── task ──► NLU 服务
   │             ├─ RoBERTa 意图召回 Top-5（本地，439 类）
   │             ├─ 按 Top-5 动态收缩工具池（400+ → 数个）
   │             ├─ 豆包 Function Calling：选定技能 + 抽取槽位
   │             ├─ 槽位规范化（中文化 / 枚举 / 数值归一）
   │             └─ MCP 工具执行（天气 / 地图 / 音乐）+ NLG
   │                 → 一次性返回结构化结果
   │
   └── chat/faq ──► 拒识门禁（本地 RoBERTa-Tiny）
                     ├─ 判拒时用大模型做上下文相关性纠偏
                     └─ 通过则调用豆包 Bot 流式回答
                         → SocketIO 分帧推送（status 0/1/2）
```

**两个关键设计点**

| 设计 | 解决的问题 |
|------|------------|
| 投机并行 + 仲裁首 Token 早停 | 路由未定时先把各路结果算出来，仲裁一返回即取用，压掉串行等待 |
| 小模型召回 + 动态 Function Calling | 400+ 工具无法一次塞进 Prompt，先用本地模型收缩候选，再让大模型精排抽槽 |

---

## 系统架构

```
┌──────────────────┐
│  dialog.py 客户端 │  python-socketio
└────────┬─────────┘
         │ SocketIO 事件（双向，支持多帧推送）
         ▼
┌─────────────────────────────────────────────┐
│  start.py  网关 :8080                        │
│  Flask-SocketIO + 线程池编排 + Redis 会话     │
└───┬──────────┬──────────┬───────────┬───────┘
    │ HTTP     │ HTTP     │ HTTPS     │
    ▼          ▼          ▼           ▼
 拒识:8007  NLU:8009   火山方舟豆包   Redis:6379
 (Tiny)        │       (改写/仲裁/
               │        FC/相关性/
               ▼        闲聊 Bot)
          意图召回:8008
          (RoBERTa-ext)
               │
               ▼
          MCP Server（高德地图 / QQ 音乐，stdio）
```

| 服务 | 端口 | 框架 | 职责 |
|------|------|------|------|
| `start.py` | 8080 | Flask-SocketIO | 编排、路由、流式推送 |
| `chatnlu_infer.py` | 8009 | FastAPI + Uvicorn | 意图召回 → FC → 槽位 → DM/MCP |
| `intent_infer.py` | 8008 | FastAPI + Uvicorn | 439 类意图 Top-5 召回 |
| `reject_infer.py` | 8007 | FastAPI + Uvicorn | 拒识二分类 |
| Redis | 6379 | — | 短 TTL 会话状态 |

**为什么网关用 SocketIO 而非 FastAPI**：闲聊需要服务端连续推送多帧（`status=0/1/2`），需要的是双向事件通道；而模型推理是一问一答，用 FastAPI 更合适。两者分层，不是替代关系。

---

## 多轮状态设计

全部存 Redis，按 `sender_id` 隔离，TTL 贴合车载短会话：

| Key | TTL | 内容 | 用途 |
|-----|-----|------|------|
| `voice:last_service:{id}` | 40s | `域#query#拒识位#回答` | 改写取上轮回答、相关性取上轮问句 |
| `voice:rewrite_history:{id}` | 40s | 最近约 3 轮对话 | 指代消解 |
| `voice:arbitration_history:{id}` | 60s | 仲裁历史 | 结合上文判领域 |
| `voice:chat_history:{id}` | 45s | 闲聊历史 | 闲聊连贯性 |

改写与仲裁的输入有意不同：**NLU / 拒识用改写后的句子**（抽槽需要语义完整），**仲裁 / 相关性 / 闲聊用原句**（保留用户真实表达）。

---

## 目录结构

```
.
├── llm对话多轮/
│   ├── start.py                 # ★ 入口网关：并行编排 + 仲裁路由 + 流式推送
│   ├── dialog.py                # 交互式客户端（联调用）
│   ├── test.py / e2e_score.py   # 批量多轮测试与端到端打分
│   ├── prompts.py               # ★ 全部 System/User Prompt 集中管理
│   ├── server.sh                # 一键按依赖顺序启动四个服务
│   │
│   ├── client/                  # 各能力的调用封装
│   │   ├── rewrite.py           # 多轮改写（含误改回退）
│   │   ├── arbitration.py       # 领域仲裁（流式首 Token 早停）
│   │   ├── reject.py            # 拒识服务客户端
│   │   ├── correlation.py       # 相关性纠偏
│   │   ├── stream_chat.py       # 闲聊流式 + 标点切帧
│   │   └── nlg.py               # 工具结果 → 口语化回复
│   │
│   ├── function_call/
│   │   ├── chatnlu_infer.py     # ★ NLU 服务：召回 → 动态 FC → 槽位 → DM
│   │   ├── function.py          # 车载技能 tools schema（400+ function）
│   │   ├── slot_process.py      # 槽位规范化（中文化/枚举/数值）
│   │   └── dm/                  # 对话管理：weather / maps / music
│   │
│   ├── mcp_core/
│   │   ├── mcp_client.py        # MCP stdio 客户端
│   │   ├── amp_server.py        # 高德地图 MCP Server（13 个工具）
│   │   └── music_server.py      # QQ 音乐 MCP Server
│   │
│   ├── train/                   # 小模型微调与推理服务
│   │   ├── run.py               # 训练入口
│   │   ├── train_eval.py        # 训练 / 验证 / 测试循环
│   │   ├── data_helper.py       # 数据读取、tokenize、batch
│   │   ├── models/              # bert.py（意图）/ bert_tiny.py（拒识）
│   │   ├── core/                # BERT 实现与 BertAdam 优化器
│   │   ├── intent_infer.py      # 意图召回服务 :8008
│   │   ├── reject_infer.py      # 拒识服务 :8007
│   │   └── result/              # 训练日志（含最终指标）
│   │
│   ├── config/
│   │   ├── config.ini.example   # 环境变量模板（复制为 config.ini 后填密钥）
│   │   ├── class.txt            # 439 类意图：id:中文名:FunctionName
│   │   ├── slot_intent.json     # 槽位英文 → 中文映射
│   │   └── new_map.json
│   │
│   ├── test/                    # 单服务客户端 + locust 压测脚本 + 测试集
│   └── utils/                   # Redis 封装、带 TraceID 的 logger
│
├── .gitignore
└── README.md
```

> `模型文件/`（预训练权重、训练数据、ckpt，约 865MB）未随仓库分发，见下文「模型与数据」。

---

## 技术栈

| 层 | 技术 |
|----|------|
| 实时网关 | Flask 3 · Flask-SocketIO · python-socketio · ThreadPoolExecutor |
| 模型微服务 | FastAPI · Uvicorn |
| 大模型 | 火山方舟豆包（Chat Completions / Bots）· Function Calling · SSE 流式 |
| 小模型 | PyTorch 2.6 · chinese-RoBERTa-wwm-ext · RoBERTa-Tiny · BertAdam |
| Agent 工具 | MCP 1.7（FastMCP, stdio）· 高德地图 REST · QQ 音乐 API |
| 状态与中间件 | Redis 5 |
| 评测 | scikit-learn（P/R/F1、Top-K）· Locust 压测 · 端到端打分脚本 |

---

## 快速开始

### 1. 环境

```bash
# Python 3.10+（开发环境 3.12）
pip install -r llm对话多轮/requirements.txt

# 启动 Redis（默认 127.0.0.1:6379）
redis-server
```

### 2. 配置密钥

```bash
cd llm对话多轮
cp config/config.ini.example config/config.ini
# 编辑 config/config.ini，填入火山方舟 API Key 与高德 Key
source config/config.ini
```

### 3. 准备模型

把预训练权重与微调 ckpt 放到 `train/` 下（目录不随仓库分发）：

```
train/
├── pretrained/
│   ├── chinese_roberta_wwm_ext/   # 意图模型底座
│   └── roberta_tiny_clue/         # 拒识模型底座
└── saved/
    ├── intent/bert.ckpt
    └── reject/bert_tiny.ckpt
```

### 4. 启动服务

```bash
bash server.sh     # 依次启动 8007 → 8008 → 8009 → 8080
```

依赖顺序不可颠倒：NLU 依赖意图服务，网关依赖 NLU 与拒识。

### 5. 对话联调

```bash
python dialog.py
# enter query: 导航去公司
# enter query: 北京明天天气怎么样
# enter query: 周杰伦有哪些歌       ← 走闲聊，流式多帧返回
```

批量多轮测试：`python test.py`，端到端打分：`python e2e_score.py`。

---

## 模型与数据

### 本地微调的两个模型

| | 意图召回 | 拒识门禁 |
|--|----------|----------|
| 底座 | chinese-RoBERTa-wwm-ext（12 层 / hidden 768） | RoBERTa-Tiny-CLUE（4 层 / hidden 312） |
| 任务 | 439 类多分类 | 二分类（0 拒识 / 1 放行） |
| 训练量 | 约 31 万条 | 约 32 万条 |
| 测试指标 | Acc 85.7% · **Acc@3 96.3%** · **Acc@5 97.5%** · macro-F1 0.845 | **Acc 91.1%** · macro P/R/F1 ≈ 0.911 |
| 线上输出 | Top-5 意图 id + 置信度 | 0/1（阈值 0.5） |
| 线上作用 | 收缩 Function Calling 的工具池 | 闲聊路径门禁 |

**训练配置**（两者共用 `train_eval.py`）：全参数微调、交叉熵、BertAdam（lr 5e-5 / warmup 5% / 梯度裁剪 / 解耦 weight decay）、batch 128、pad 32、3 epoch、按验证集 loss 存最优、1000 step 不提升早停。训练在单卡 48GB vGPU 上完成。

```bash
cd llm对话多轮/train
python run.py --model bert      --data intent   # 意图
python run.py --model bert_tiny --data reject   # 拒识
```

**为什么产品口径看 Acc@5 而不是 Top-1**：线上只用 Top-5 收缩工具池，真实意图进入前五即可，最终技能由大模型在小候选集内精排决定。Top-1 偏低主要来自 439 类的长尾分布与标签语义重叠。

### 数据格式

两个任务统一格式 `query \t label_id`：

```
导航到民乐澜庭	1
播放永远是朋友	2
建平	3            # 意图：3 = Unknown
```

意图标签定义在 `config/class.txt`，三段式 `id:中文名:FunctionName`，把模型标签与工具本体对齐——训练用 id，线上用 FunctionName 去 `function.py` 取 tools schema。

> 训练数据与权重未入库：体积大（单文件近 400MB，超过 GitHub 限制），且来源于课程配套资料。

---

## 已知局限

工程上尚未补齐的部分，如实列出：

- **FAQ 分支未落地**：仲裁能产出 `faq`（手册/故障/操作方法类），但入口与 `chat` 走同一路径，缺少手册 RAG。
- **DM 覆盖有限**：天气、地图、音乐三个域会真正调用工具；其余车控 function 只做意图与槽位解析，执行留给车机侧。
- **MCP 连接未复用**：每次请求都重建 stdio 连接，缺少连接池与统一清理。
- **数据切分有泄漏**：意图测试集约 21% 的 query 与训练集重复，训练集自身精确重复率约 13%，因此 Top-1 指标略偏高；另有约 346 条标签冲突样本，反映类目语义重叠。
- **长尾未处理**：训练侧无重采样 / class weight，macro-F1 明显低于 Accuracy。
- **回归评测依赖人工**：`e2e_score.py` 需人工标注文件，缺少自动化回归集。

---

## 声明

本仓库源于「丁师兄大模型」课程配套项目，用于个人学习与技术复盘，相关代码版权归原作者所有。在此基础上我做的工作包括：链路梳理与文档化、密钥配置改为环境变量、数据质量取证分析（重复率 / 泄漏 / 标签冲突）、以及上述局限的定位。

如用于商业用途请联系原作者授权。
