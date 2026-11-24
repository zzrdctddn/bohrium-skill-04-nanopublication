# Nanopublication 受控字段字典与判定规则

## 1. 顶层字段（固定，不得自定义别名）
| 字段 | 类型 | 含义 | 必填 |
|---|---|---|---|
| `case_id` | string | 原样带回输入标识 | 是 |
| `status` | enum | success/partial/blocked | 是 |
| `artifact` | object | 主产物，含 `nanopublications` | 是 |
| `claim_evidence_map` | array | 断言→证据定位映射 | 建议 |
| `provenance_chain` | array | 来源链路记录 | 建议 |
| `quality_check` | array | 质量检查项 | 建议 |
| `uncertainties` | array[string] | 材料缺失/无法判断项 | 是 |
| `human_review_items` | array[string] | 需人工确认项 | 是 |
| `change_log` | array | 核验阶段改动记录 | 建议 |
| `self_check` | object | 自检结论 | 是 |

## 2. 断言单元字段（artifact.nanopublications[]）
| 字段 | 类型 | 含义 |
|---|---|---|
| `claim_id` | string | 单元唯一标识，形如 `A1`/`A2` |
| `claim` | string | 单条原子断言文本，含条件与限定 |
| `subject` | string | 主体（材料真实对象） |
| `predicate` | string | 关系类型（受控词表，见下） |
| `object` | string | 客体 |
| `evidence.text` | string | 最小充分证据片段原文 |
| `evidence.location` | string | 段落ID/章节/图表号/公式号 |
| `evidence.evidence_type` | enum | `direct` / `indirect` |
| `evidence.status` | enum | `supported` / `unsupported` |
| `provenance.paper_id` | string | DOI 或论文标识 |
| `provenance.authors` | array[string] | 作者列表 |
| `provenance.year` | string | 年份 |
| `provenance.version` | string | 版本 |
| `provenance.data_source` | string | 数据来源 |
| `provenance.identifier` | string | 单元/来源唯一标识 |
| `confidence` | enum | high/medium/low |
| `confidence_basis` | string | 置信度依据 |

## 3. 关系类型受控词表（predicate）
`reports`（报告结果）、`shows`（表明）、`achieves`（达到某指标）、`outperforms`（优于）、`causes`（导致，需直接因果证据）、`correlates_with`（相关，不得写成因果）、`uses`（使用）、`applies_to`（适用于）、`is_limited_by`（受限于）、`contradicts`（与…矛盾）、`other`（其他，须在 confidence_basis 写原文表述）。**禁止**把共现、转述、推测标为 `causes`/`outperforms`。

## 4. 证据类型
- `direct`：原文直接陈述该断言，逐字可查。
- `indirect`：需跨句推断或由其他事实推出，必须保留推断链说明。

## 5. 置信度判定
- `high`：证据逐字可查，位置精确到段落/图表号，来源元数据齐全。
- `medium`：证据可定位但需跨句推断，或来源元数据有缺项（标 unverified）。
- `low`：证据模糊/冲突/无支撑（unsupported），必须进 `human_review_items`。

## 6. 序列化形态
- JSON（UTF-8），字段名严格照表，不得使用 `myClaim`/`text_snippet` 等自定义别名。
- 标识符规则：`claim_id` 用 `A<n>`；`provenance.identifier` 用 `<paper_id>#<claim_id>`。
- 不导出材料外新增事实、数值、单位；缺失项用空字符串或 `unverified` 标注，不臆造。
