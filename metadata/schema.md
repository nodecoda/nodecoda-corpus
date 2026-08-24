# manifest.jsonl 字段说明（nodecoda-corpus 元数据）

每行一个**唯一工作流**（platform workflow_id 去重;structural fingerprint
去重后的重复以 `duplicate_of_fp` 计数标注）。字段:

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| workflow_id | string | Coze 平台工作流 64 位 ID（雪花变体,高位 32bit 为 Unix 秒,见 reports/coze-id-spec.md） |
| name | string | 平台工作流名（slug 化） |
| desc | string | 用户提供的描述（可能含个人信息,按作者版权使用） |
| envelope | string | standard / wrapped（导出信封格式） |
| artifact | string | 解码产物 workflow.yaml / workflow.json |
| format | - | yaml = 现行格式;json = 早期 legacy 格式（type 数字编码） |
| node_count / nodes / edge_count / edges | int | 顶层节点/边规模 |
| node_profile | map | 全量递归节点类型计数（含 batch/loop 嵌套;json 容器键 blocks 已计入） |
| fingerprint | string | 结构指纹（id/位置/顺序无关,用于识别同图异构导出） |
| duplicate_of_fp | int | 被该结构指纹去重掉的副本数 |
| empty_shell | string? | HARD=纯骨架 / MINIMAL=单执行节点 / 缺省=normal（判定见 empty-shells-v2.md） |
| json_decode_errors | int | legacy json 解码错误计数 |

> 原始工作流内容（nodes/edges 数据）**不发布**（版权归原作者）。本清单仅含
> 公开元数据与自研统计,可用于检索/去重/规模分析。
