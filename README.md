# 纳米出版 Skill（nanopub-claim-evidence-binder）

一个把论文段落转成"可核验 nanopublication 断言单元"的 Agent Skill。核心：**原子断言拆解 + 最小充分证据绑定 + 来源链路补全 + 受控字段导出 + 拆解核验分离**（ACEB）。

## 目录结构
- `SKILL.md` —— Skill 本体（YAML frontmatter、工作流、输出契约）
- `references/` —— ACEB 方法细则、自检清单、字段 schema、受控字段字典
- `assets/` —— 断言-证据单元可编辑模板
- `examples/` —— 最小调用示例（已标注为示例，非真实案例）
- `GITHUB_URL.txt` —— 提交时填 GitHub 公开链接（当前为 REPLACE_ME 占位）

## 使用方式
1. 将本目录作为 Skill 安装到 Agent（平台约定路径 `/skills/weekly-skill`）。
2. 输入 `/input/case.json` 与 `task.md`，自然语言下指令如"把这段结论拆成断言并绑定证据"。
3. 按 SKILL.md 的 SOP 执行：来源登记 → 拆断言 → 绑定证据 → 核对关系主体 → 结构化导出 → 独立核验。
4. 输出到 `/app/output/result.json` 与 `nanopub_units.md`。

## 依赖
仅需文本处理能力与 `node`（用于校验 JSON 可解析）；无外部运行时依赖。受控词表与字段字典内置于 `references/`。

## 适用与限制
- 适用：论文摘要/结论段落的断言-证据单元构建、证据绑定核验、来源追溯与结构化导出。
- 限制：材料为空或无法访问原文时标记 partial/blocked，不凭标签补造数据；只做段落级最小成果，不替代完整 Nanopublication 系统；不编造来源元数据与 benchmark 分数。

## Runtime contract
Skill 加载到 `/skills/weekly-skill`，输入 `/input/case.json` 与 `task.md`，输出到 `/app/output/result.json`（含全部输出契约字段与 `case_id`）。资源 ≤50 个 UTF-8 文本文件、单文件 ≤500KB、总计 ≤2MB、无软链。执行预算 64 次模型请求 / 900 秒 / 每次 16384 token。位置一律用材料给出的段落 ID/章节/图表号，不把自建页码当原文页码。

## 评分对应
实际效果(40)、使用增益(25)、创新增益(15)、可靠体验(10)、生态贡献(10) 的支撑材料分别见 SKILL.md 第 3/2/2/6/8 节与 `references/`；对照需由平台用固定案例与白板完成，作品不自带答案与分数。
