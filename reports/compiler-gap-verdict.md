# 编译器表达力 gap 裁决（compiler-gap-verdict v1）

> 裁决日期: 2026-08-24 | 范围: corpus/decoded 501 唯一工作流的节点类型谱
> 产出: 编译器 TODO 优先级路线图（batch-b 重写选样的边界依据）

## 结论速览

501 个工作流中 **158 个（31.5%）含疑似 gap 节点类型**。逐类裁决后:
**真 gap 4 类 209 实例**（`variable_aggregator` 为统计别名,已排除）。

| # | 类型 | 实例 | 裁决 | 优先级 |
|---|---|---|---|---|
| 1 | `variable_aggregator` | 43(manifest)/124(json) | **非 gap**: 开源枚举 VariableAggregator = 商业 yaml `variable_merge`（编译器 Aggregate 已支持,27 例 roundtrip 覆盖）; fingerprint.py 口径已统一 | — |
| 2 | `drawing_board` | 145 | **内置节点绑定通道**（P1）: 画布排版,canvasSchema 自由 JSON → 注册表描述 `string canvas_schema` 透传,用户用 json.stringify 构造 | P1 |
| 3 | `image_generate` | 53 | **内置节点绑定通道**（P1）: 参数结构化（modelSetting/prompt/参考图）,输出 image | P1 |
| 4 | `video_frame_extractor` | 2 | **内置节点绑定通道**（P1）: 参数简单（count/interval/method+video）,零障碍 | P1 |
| 5 | `intent` | 9 | **P2 语法设计**: 多出口（intents 每意图一出口）+ llmParam,单输出 tool 语义无法表达;降级 if 链不等价（多次 LLM 调用） | P2 |
| 6 | `llm` 结构化输出 | 大量 | **P2 语法/渲染设计**: coze-biz llm 的 JSON 结构化输出（单 output 对象多字段 或 平铺多端口）ncoda `llm()` 无法表达 → 需 `llm<T>` 结构化输出声明 + responseFormat 渲染（当前 json.parse 降级: 多 1 code 节点 + 平台 JSON 模式丢失） | P2 |
| 7 | capability def 可选参数判定 | 注册表生成 | **P1 数据修复**: def 生成把无默认值的参数全标 required,但平台 plugin 的可选参数（如即梦 image_urls, 文生图不传）在原始工作流中缺省 → 调用端被迫传 `[]`（L4 多 1 literal） | P1 |

## 裁决依据（证据）

### 1. variable_aggregator — 非 gap（统计口径问题）

- `fingerprint.py` 将 legacy json 的 `type=32` 归一为 `variable_aggregator`（沿用 coze-studio 开源枚举名）; `legacy_json_to_ast.py` 归一为 `variable_merge`（商业 yaml 名）
- **同一节点类型两个名字**: decoded yaml 中 0 个 `variable_aggregator`,json 中 124 个 `type=32` —— 全部是 variable_merge
- 编译器 IR `Aggregate`（variable-merge/Group1）在 27 例 roundtrip 中已全面覆盖
- 处置: fingerprint.py 口径统一为 `variable_merge`,manifest/corpus-v1 已重跑

### 2. drawing_board（145 实例,占比最大）— P1 内置节点绑定通道

真实结构（yaml）:
```json
{"type":"drawing_board","parameters":{"canvasSchema":"{\"version\":\"6.0.0-rc2\",\"width\":800,\"height\":500,\"backgroundColor\":\"#ffffffff\",\"customVariableRefs\":[...],\"objects\":[...]}"}}
```
- 画布是**自由 JSON**（canvasSchema 字符串）: 版式/字体/坐标/文本/图片对象 + 对上游变量的引用（customVariableRefs 把 ncoda 变量嵌入画布）
- 通道: 注册表新增 `drawing_board.render({ canvas_schema })`（参数 string 透传）+ 绑定表 `node_type: drawing_board`
- 表达力: 画布编排退化为 JSON 模板（ncoda 侧用 json.stringify + 变量插值构造）—— 可验证但弱;建议先做"单画布 + 文本/图片引用"最小子集验证 roundtrip
- 备选: 为画布设计 DSL（P3,若集散地需要画布类工作流高覆盖再议）

### 3. image_generate（53 实例）— P1 内置节点绑定通道

真实结构（yaml）:
```json
{"type":"image_generate","parameters":{"apiParam":null,"modelSetting":{"custom_ratio":{...},"ddim_steps":40,"images_reference":{},"model":8,"ratio":0},"node_inputs":[{"name":"output","input":{"value":{"path":"output","ref_node":"157092"}}}],"node_outputs":{"data":{"type":"image"},"msg":{"type":"string"}}}}
```
- 参数结构化程度高（modelSetting 的 ratio/ddim_steps/model + prompt 输入 + 参考图引用）→ 注册表可完整描述
- 输出 `data` 为 image 类型: ncoda 类型系统已有 `FileTypeWithConstraints({Types:["image"]})`（types/types_test.go）→ 可直接声明 `type ... { File data; string msg; }` 或 any 兜底
- 通道: 注册表 `image_generate.generate({ prompt, ratio, ... })` + 绑定 `node_type: image_generate`

### 4. video_frame_extractor（2 实例）— P1 内置节点绑定通道

```json
{"type":"video_frame_extractor","parameters":{"frameExtractionCount":24,"frameExtractionIntervalMs":1000,"frameExtractionMethod":"EqualTime","node_inputs":[{"name":"video",...}],"node_outputs":{"data":{"type":"object","properties":{"chunks":{...}}}}}}
```
- 参数完全结构化,零障碍;输出 object（chunks 列表）可用 type 图描述

### 5. intent（9 实例）— P2 语法设计

```json
{"type":"intent","parameters":{"intents":[{"name":"比例9:16"},{"name":"比例1:1"},{"name":"比例16:9"}],"llmParam":{"modelName":"豆包·1.8·深度思考","prompt":{...}}}}
```
- 多出口: 每个 intent 一个 branch 输出（类似 switch）,且共享一个 LLM 参数块
- 单输出 tool 语义无法表达多出口;if 链降级需多次 LLM 调用 → **行为不等价**
- 量少（9 实例）: 列 P2,设计方向 = switch-like 语法（`intent(...) { "意图A" -> ...; "意图B" -> ...; }`）或 if-elif 扩展,独立议题

## 编译器 TODO（按优先级）

| 优先级 | 任务 | 状态 |
|---|---|---|
| P0 | 修正 variable_aggregator 统计口径 | ✅ 已完成 |
| P1 | 绑定表 schema 扩展 `node_type` 绑定 | ✅ 已完成（godcc ToolBinding.NodeType,二选一校验 + 白名单） |
| P1 | 渲染器: ToolCall 命中 node_type → 内置节点 | ✅ 已完成（renderBuiltinToolNode × 3,单测 render_builtin_test.go） |
| P1 | 注册表: 3 个 capability 头文件 | ✅ 已完成（capabilities-ncoda,74 defs） |
| P1 | 真实工作流验证 | ✅ S89 觉醒海报画板: canvasSchema 9000B 字节级一致,变量绑定同构,oracle L0 0 error |
| P2 | intent 语法设计（switch-like） | 未开始 |
| P3 | drawing_board 画布 DSL（可选,覆盖度驱动） | 未开始 |
| P2 | llm 结构化输出: `llm<T>` 声明 + responseFormat 渲染（batch-b 驱动,语料大量 llm JSON 模式） | 未开始 |
| P1 | def 可选参数: capability_header Required 判定支持默认值/可选项来源（插件 api 定义驱动） | 未开始 |
| P1 | 本轮已修: registryTypeDescriptor map<string,any> + Any 期望 map 不再聚合误报 + llm message content 显式 string 检查 | ✅ 已完成（godcc 8a860f2, Z20 编译驱动） |

> 2026-08-24 更新: P1 全部落地。IR 层新增 toolValueTree（tool map 参数
> 支持变量 ref 叶子 → {ref_node,path} 编码,数组参数保持旧报错语义）;
> image_generate / video_frame_extractor 渲染已单测覆盖,真实案例留 batch-b。

## 2026-08-24 batch-b smoke 更新（Z20_xhs_zhuti_biji）

- **P1 通道实战验证通过**: 4×drawing_board canvasSchema 字节级一致 + 变量绑定同构;即梦/抠图插件绑定正确;foreign code 透传;text concat;llm 基础调用
- **编译产物**: OP-SET-EQ（操作集完全一致）;差异全部归因于 2 个新 gap（llm 结构化输出 / def 可选参数）+ 良性边形态差异
- **编译器修复**: map<string,any> 参数类型 + Any 期望下 map 不再误报 + llm message content 显式 string 检查（godcc 8a860f2）

## 影响面

- P1 落地后: drawing_board/image_generate/video_frame 覆盖 153+ 工作流 → **可编译面 343 → ~496（99%）**
- intent 9 个暂留 P2;合并后 501 全覆盖
- batch-b 选样: 先选**无 gap 的 343 个**（P1 未落地也能重写）;P1 落地后扩至 496
