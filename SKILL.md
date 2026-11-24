---
name: nanopub-claim-evidence-binder
description: 把论文段落拆成可独立判断真假的原子断言，为每条断言绑定最小充分证据片段并标注原文位置，补全论文/作者/年份/版本来源链路，导出符合约定字段、可被下游直接解析的 nanopublication 结构。当用户要求"拆断言""绑定证据""抽取 nanopublication""来源追溯""结构化导出断言""核验断言与证据是否对得上"时使用。输出含原子断言表、断言-证据绑定表、来源元数据链、质量检查表与结构化 result.json。
version: 1.0.0
license: MIT
---

# 纳米出版：原子断言-证据绑定（Atomic Claim-Evidence Binding, ACEB）

> 面向真实论文材料，把"读一段结论"从整段塞进一个字段，变成**一条条可独立判真、可回原文、可机器解析的断言工程**。核心不是复述段落，而是让每条断言回答四个问题：拆得细吗？对得上吗？追得回吗？读得懂吗？

## 1. 用途与适用边界

**解决什么**：基础 Agent 做 Nanopublication 类任务的常见失败——把整段摘要/结论塞进一个"断言"字段、断言里混入多个可独立判真的事实、证据片段过大或过小而不"最小充分"、把间接推论当直接证据、主体与关系方向搞反、来源元数据缺失或与论文对不上、导出的 JSON 字段名自定义导致下游无法解析。本 Skill 用"拆解—绑定—溯源—导出四道独立工序 + 逐条自检"消除这些失败。

**触发场景**：从论文段落/摘要拆原子断言；为断言绑定证据片段与原文位置；核验断言-证据是否对得上；补全论文/作者/年份/版本/DOI 来源元数据；导出符合约定字段与序列化形态的 nanopublication 结构；检测断言过宽、证据不足、主体混淆、关系方向错误。

**不该用**：材料为空且无法访问原文时（应报告材料缺口，不凭标签补造断言）；用户只要语言润色时（转用学术语言润色 Skill）；要求"完整 Nanopublication 系统/批量知识库"时（本 Skill 只做单段落级断言-证据单元的可用最小成果）。

**输入**：`/input/case.json`（含 `case_id`、任务说明、论文材料或段落、来源元数据、输入约束）与 `task.md`。**输出**：`/app/output/result.json`（字段见第 4 节）+ 可编辑的 `nanopub_units.md`。

## 2. 核心方法（ACEB = Atomic Claim-Evidence Binding）

本 Skill 的方法名为 **ACEB = Atomic Claim-Evidence Binding**，四个不可跳过的机制逐一对应命题的四个关键问题：

1. **机制 A｜原子断言拆解（Atomic Claim Decomposition）→ 拆得细**。把一段结论拆成若干条**各自可独立判断真假**的断言，每条只承载一个主体-谓词-客体/一个可判定的命题；保留原文的条件、限定词、范围与否定，不丢"在 X 条件下""约""可能""显著"等限定。复合句按并列/因果/转折拆开，禁止"一段话一个断言"。
2. **机制 C｜证据绑定（Claim–Evidence Binding）→ 对得上**。每条断言绑定**最小充分证据片段**（能支撑该断言的最短连续原文），并标出原文位置（段落 ID / 章节 / 图表号 / 字段或公式位置）。证据分 `direct`（直接证据）与 `indirect`（间接推论）两类；推论不得冒充直接证据，无支撑片段的断言标记 `unsupported` 并进人工复核，不写成事实。
3. **机制 E｜来源链路补全（Evidence–Source Provenance Chain）→ 追得回**。每条断言-证据单元补全来源链：论文标识（DOI/标题）、作者、年份、版本、数据来源、标识符；保证他人能据此回到原文。跨论文/跨版本不得混用元数据；材料缺失项标 `unverified`，不臆造。
4. **机制 B｜受控字段导出（Bound-Schema Serialized Export）→ 读得懂**。按 `references/nanopub_fields.md` 定义的受控字段与序列化形态导出，字段名固定、类型固定、标识符规则固定，下游程序可直接解析，无需人工清洗。

另外贯穿一条铁律：**拆解与核验分离**——先完成拆解与绑定，再以"审阅者"身份独立执行第 5 节自检，两遍不可合并。

## 3. 工作流（SOP）

严格按顺序执行，每步都有检查点；检查点不过则回退该步。

**Step 0 · 输入解析与边界确认**
- 读 `case.json` 与 `task.md`，提取：材料类型（纯文本/结构化/PDF）、目标字段与序列化形态、来源元数据（论文/作者/年份/版本/DOI）、明确禁止事项。
- 检查点：材料是否可访问、来源元数据是否齐全？缺口登记到 `uncertainties`，不凭错误标签补造数据。

**Step 1 · 来源元数据登记（Provenance Setup）**
- 建立来源记录：论文标识、作者列表、年份、版本、数据来源、标识符。位置一律用材料给出的段落 ID/章节/图表号，不把自建页码当原文页码。
- 检查点：元数据逐项与材料比对；材料里没有的作者/年份/DOI 不填，标 `unverified`。

**Step 2 · 原子断言拆解（机制 A）**
- 通读材料，按"一个可独立判真命题 = 一条断言"拆分；复合句按并列/因果/转折拆；每条保留条件、限定词、范围、否定与 quantifier；分配 `claim_id`。
- 检查点：任意一条断言是否含两个及以上可独立判真的点？含则再拆；限定词强弱的改变要登记到 `change_log`。

**Step 3 · 证据绑定与定位（机制 C）**
- 为每条断言找**最小充分证据片段**（能支撑该断言的最短连续原文），标位置（段落 ID/章节/图表号）；判 `direct` 或 `indirect`。
- 检查点：证据片段是否既充分又最小？是否指向正确主体与关系方向？找不到支撑的断言标 `unsupported`，进 `human_review_items`，不写成事实。

**Step 4 · 关系与主体核对**
- 为每条断言标主体（subject）、关系类型（predicate，用受控词表）、客体（object）；核对关系方向是否与原文一致，不把共现/转述/推测当直接证据。
- 检查点：关系类型是否来自受控词表、方向是否与原文一致？主体是否张冠李戴？错则修正并记 `change_log`。

**Step 5 · 结构化导出（机制 B）**
- 按 `references/nanopub_fields.md` 的字段与序列化形态生成每条断言单元；补齐 `claim_id`、证据片段、位置、来源链、置信度依据、标识符。
- 检查点：字段名/类型/标识符规则是否与字段字典一致？用 `node -e "JSON.parse(...)"` 自检可解析性。

**Step 6 · 独立核验与输出**
- 以审阅者身份逐项走 `references/checklist.md`；生成 `quality_check` 表（断言过宽/证据不足/关系错误/主体混淆/来源缺失）；每处未通过项都有 `change_log`。
- 写 `nanopub_units.md`（可编辑）+ `result.json`（结构化结果，含 `case_id` 与全部输出契约字段）。

## 4. 输出契约（result.json）

```json
{
  "case_id": "<原样带回>",
  "status": "success | partial | blocked",
  "artifact": {
    "nanopublications": [
      {
        "claim_id": "A1",
        "claim": "<单条原子断言，含条件与限定>",
        "subject": "<主体>",
        "predicate": "<受控关系类型>",
        "object": "<客体>",
        "evidence": {
          "text": "<最小充分证据片段原文>",
          "location": "<段落ID/章节/图表号>",
          "evidence_type": "direct | indirect",
          "status": "supported | unsupported"
        },
        "provenance": {
          "paper_id": "<DOI或论文标识>",
          "authors": ["<作者>"],
          "year": "<年份>",
          "version": "<版本>",
          "data_source": "<数据来源>",
          "identifier": "<单元标识符>"
        },
        "confidence": "high | medium | low",
        "confidence_basis": "<置信度依据>"
      }
    ]
  },
  "claim_evidence_map": [
    {"claim_id": "A1", "evidence_location": "<位置>", "evidence_type": "direct", "sufficient": true}
  ],
  "provenance_chain": [
    {"claim_id": "A1", "paper_id": "<...>", "authors": ["..."], "year": "...", "version": "...", "identifier": "<...>"}
  ],
  "quality_check": [
    {"claim_id": "A1", "check": "claims_atomic | evidence_minimal_sufficient | relation_direction | subject_identity | provenance_complete", "result": "pass | fail", "detail": "..."}
  ],
  "uncertainties": ["<材料缺失/无法判断的项，逐条说明>"],
  "human_review_items": ["<需人工确认的 unsupported/indirect 项>"],
  "change_log": [{"step": 6, "action": "...", "reason": "..."}],
  "self_check": {
    "claims_atomic": true,
    "evidence_bound_and_located": true,
    "provenance_complete": true,
    "schema_parseable": true
  }
}
```

字段原则：`status` 诚实反映完成情况；`partial` 必须写清 `uncertainties`；`unsupported`/`indirect` 断言进 `human_review_items`；不得编造来源元数据或 benchmark 分数；材料缺失、无法判断、运行失败分别说明。

## 5. 质量自检清单

执行前完整读取 `references/checklist.md`，逐项打钩。关键硬项：

- [ ] 每条断言只承载一个可独立判真命题，条件与限定词保留
- [ ] 每条断言有最小充分证据片段与原文位置
- [ ] 证据区分 direct/indirect，无支撑的标 unsupported 并进人工复核
- [ ] 主体、关系类型（受控词表）、方向均与原文一致
- [ ] 来源链（论文/作者/年份/版本/标识）齐全，缺项标 unverified
- [ ] 字段名/类型/标识符符合 `references/nanopub_fields.md`
- [ ] result.json 可被 JSON.parse 解析
- [ ] 拆解与核验分离，所有改动有 change_log

## 6. 异常处理与边界

| 情况 | 处理 |
|---|---|
| 扫描版无文本层 | `status=blocked`，说明缺 OCR 文本层，不凭标签补造断言 |
| 断言找不到支撑片段 | 标 `unsupported`，进 `human_review_items`，不写成事实 |
| 证据跨图表/公式 | 位置标到图表号/公式号，片段保留线性化原文并注明 |
| 断言与证据冲突 | 不强行绑定，标 `fail` 并写明冲突点，进人工复核 |
| 来源元数据缺失/冲突 | 缺项标 `unverified`，冲突项不改原文，进 `uncertainties` |
| 同一段落多结论 | 全部拆成独立断言，逐条绑定证据，不合并 |
| 关系类型不在词表 | 用 `other` + 原文表述，并在 `change_log` 说明 |

## 7. 最小调用示例（示意，非真实案例）

**输入**（`case.json` 片段）：
```json
{"case_id":"DEMO-NOT-A-REAL-CASE","task":"把给定结论段落拆成原子断言并绑定证据","material":"……（示意段落）","source":{"paper_id":"10.xxxx/demo","authors":["A. Author"],"year":"2024","version":"v1"}}
```
**调用**：加载本 Skill 后，用户说"用 nanopub-claim-evidence-binder 拆这段并绑定证据"。

**示意输出**（节选）：
```json
{"case_id":"DEMO-NOT-A-REAL-CASE","status":"success","artifact":{"nanopublications":[{"claim_id":"A1","claim":"……（示意断言，含限定）","subject":"...","predicate":"reports","object":"...","evidence":{"text":"……（示意片段）","location":"para-3","evidence_type":"direct","status":"supported"},"confidence":"high"}]},"self_check":{"claims_atomic":true,"evidence_bound_and_located":true,"provenance_complete":true,"schema_parseable":true}}
```
> 上述为演示接口用的虚构示例，不代表任何真实案例结果。

## 8. 资源清单

- `references/method.md` —— ACEB 方法细则与反例库
- `references/checklist.md` —— 逐项自检清单（覆盖拆得细/对得上/追得回/读得懂）
- `references/schema.json` —— 输出字段字典（draft-07）
- `references/nanopub_fields.md` —— 受控字段字典、关系类型词表、证据类型与置信度规则
- `assets/nanopub_units_template.md` —— 断言-证据单元可编辑模板
- `examples/` —— 最小调用示例（标注为示例，非真实案例）
