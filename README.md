# nodecoda-corpus — Coze 工作流公开元数据 + ncoda 重写语料

NodeCoda 项目（ncoda 语言 → 多后端工作流编译器,见
[nodecoda-compiler](https://github.com/nodecoda/ncoda-compiler)）的公开语料库:
**市面上最大的 Coze 工作流集散地** —— 公开元数据 + 自研 ncoda 重写内容。

## 数据构成

| 目录 | 内容 | 许可 |
| --- | --- | --- |
| `metadata/manifest.jsonl` | 501 个唯一工作流公开元数据（ID/规模/节点谱/空壳标注） | CC0-1.0 |
| `workflows/` | 27 个 ncoda 语义重写 + `capabilities-ncoda/`（43 能力头文件）+ `coze-bindings.yaml` | Apache-2.0 |
| `reports/` | roundtrip 行为等价报告 / 空壳清单 / ID 规范 / 编译器 gap 裁决 | Apache-2.0 |
| `tools/` | 复现与解码工具 | Apache-2.0 |

## 为什么公开元数据而非原始工作流

- **版权**: 原始工作流（`workflow.yaml/json`）归各作者所有,不在此分发。
  本仓库仅发布**事实性元数据**（CC0）与**自研内容**（Apache-2.0）。
- **验证**: `workflows/` 下的 ncoda 重写是「ncoda → 编译器 → coze-biz」
  双向 roundtrip 的输入,`reports/roundtrip-*.json` 记录与原始的行为等价
  判定（L0-L4 五级签名）。

## 快速开始

```bash
# 检索: 按规模/节点谱过滤
python3 - <<'EOF'
import json
for line in open("metadata/manifest.jsonl"):
    r = json.loads(line)
    if r.get("empty_shell") in (None,) and r["node_count"] > 50:
        print(r["workflow_id"], r["name"], r["node_count"])
EOF

# 解码 64 位 ID（时间戳/版本先后）
python3 tools/coze_id.py 7531683265519517735
```

## 生态

- [nodecoda-compiler](https://github.com/nodecoda/ncoda-compiler) — ncoda 编译器（后端: coze-biz / dify / coze）
- [nodecoda-workflow](https://github.com/nodecoda/nodecoda-workflow) — AI Agent 工作流生成服务

## 免责声明

`desc` 为用户提供的描述,可能包含个人信息;按原作者版权使用。ID 为平台公开
标识;注册/绑定映射（`coze-bindings.yaml`）仅用于编译期工具解析。
