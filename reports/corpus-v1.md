# NodeCoda Corpus v1 — 概要

- 生成时间: 2026-08-24T09:04:27
- 原始 zip 数: **772**
- 解码成功: **501**
- 平台 id 去重后: **581**
- 结构指纹去重后（唯一工作流）: **501**

## 信封/格式

| 来源格式 | 解码数 | 唯一数 |
|---|---|---|
| standard | 262 | 237 |
| wrapped | 510 | 264 |

## 节点类型谱（唯一工作流合计）

| 节点类型 | 数量 |
|---|---|
| plugin | 6442 |
| llm | 1112 |
| code | 960 |
| comment | 912 |
| text | 550 |
| start | 501 |
| end | 501 |
| output | 413 |
| batch | 332 |
| loop | 308 |
| if | 182 |
| drawing_board | 145 |
| variable_merge | 139 |
| image_generate | 53 |
| intent | 9 |
| http | 4 |
| video_frame_extractor | 2 |

## 空壳标注（empty-shells-v2 口径）

| empty_shell | 数量 |
|---|---|
| HARD | 3 |
| MINIMAL | 14 |
| normal | 484 |

> `empty_shell` 字段: HARD=纯骨架(无业务节点) / MINIMAL=单执行节点 /
> normal=有实质业务。判定见 tools/empty_shell_scan.py 与
> corpus/reports/empty-shells-v2.md。


## 规模分布（唯一工作流）

- 节点总数: 12565，单工作流 min=2 max=150 median=19
- 桶分布: 1-10=132, 11-30=251, 31-60=81, 61-100=31, 100+=6

_生成自 tools/index.py；原始 zip 保留在私有 corpus/raw，公开层仅含元数据与自研内容。_
