# Coze 64 位 ID 生成与解析规范（coze-id-spec v1）

> **状态**: 已实证（源码 + 语料双向验证） | 落盘日期: 2026-08-24
> **适用范围**: plugin_id / api_id / workflow_id / tool_id 等 Coze 系 64 位整数 ID
> **配套工具**: `tools/coze_id.py`（解码/编码/布局校验）

## 1. 结论速览

Coze 的 64 位 ID 是 **带时间戳的雪花变体**，生成公式（来自 coze-studio
开源仓库 `backend/infra/idgen/impl/idgen/idgen.go`）：

```
id = (seconds) << 32 + (millis) << 22 + (counter) << 14 + svrID
```

- `id >> 32` = **Unix 秒**（相对 1970 的绝对时间戳，无私有 epoch）
- **版本先后判定 = 直接比较 `id >> 32`**，不需要任何元数据
- 同一毫秒内同一实例最多分配 255 个 ID（counter 0-255，Redis INCR 自增）

## 2. 位布局

| 位区间 | 宽度 | 字段 | 取值 | 说明 |
|---|---|---|---|---|
| 63..32 | 32 | `seconds` | 0 .. 2^32-1 | Unix 秒（idgen 校验 `seconds & 0xFFFFFFFF == seconds`） |
| 31..22 | 10 | `millis` | 0..999 | 秒内毫秒（`ms % 1000`） |
| 21..14 | 8 | `counter` | 0..255 | 同一毫秒内自增计数器（`maxCounterPosition = 255`） |
| 13..0 | 14 | `svrID` | 0..2^14-1 | 服务器实例 ID（当前实现注释 "A server id is all 0"，线上实测多实例非零） |

## 3. 生成算法（coze-studio 源码）

来源：`backend/infra/idgen/impl/idgen/idgen.go`（`GenMultiIDs`）。

关键逻辑：

```go
const maxCounterPosition = 255

ms := time.Now().UnixNano() / int64(time.Millisecond)   // 当前毫秒
// 同一毫秒内的 Redis 计数器自增：
counterPosition, _ := i.IncrBy(ctx, redisKey, leftNum)  // key: id_generator:<ns>:<svr>:<ms>

// 逐条生成：
for i := start; i < end; i++ {
    seconds := ms / 1000
    millis  := ms % 1000
    id := (seconds)<<32 + (millis)<<22 + i<<14 + svrID
}
```

要点：

1. **时间源**: `UnixNano / 1e6`，即 Unix 毫秒 → 高 32 位是 Unix 秒。
2. **计数器**: 每毫秒每服务器一个 Redis key（`id_generator:<namespace>:<svrID>:<ms>`），
   INCR 自增；counter 超出 255 则推进到下一毫秒继续（最多尝试 8 次）。
3. **批次分配**: `GenMultiIDs(ctx, counts)` 支持一次批量取多个连续 counter
   （如创建插件时一次性为其工具分配 api_id）。
4. **上限**: 单个毫秒、单个实例最多 255 个 ID；同一毫秒不同 svrID 可并行。

## 4. 解析算法

```python
def decode(id):
    seconds = id >> 32
    millis  = (id >> 22) & 0x3FF
    counter = (id >> 14) & 0xFF
    svr     = id & 0x3FFF
    return seconds, millis, counter, svr
```

校验规则：

- `seconds` 必须 ≤ 2^32-1（32 位秒；超限的整数不是本规范 ID）
- `millis` 必须 ≤ 999（超出说明字段错位）
- 合法布局下 `id >> 32` 落在 2024~2027 区间（按当前语料观测）

⚠️ **常见错误**: 不要用 `id >> 22` 当时间戳 —— `(id>>22) & 0x3FF` 是毫秒、
更高位是 `seconds << 10`，`id >> 22` 整体是秒与毫秒的混合，无绝对语义。

## 5. 语料实证（nodecoda-corpus 7829 实例交叉验证）

### 5.1 同一插件多版本 api_id 时间线（剪映小助手 add_audios）

| api_id | 解码 | 时间 | counter/svr |
|---|---|---|---|
| 7457837925833834536 | `>>32=1736413204` | 2025-01-09 09:00:04Z | 12 / 40 |
| 7513599192976375817 | `>>32=1749396136` | 2025-06-08 15:22:16Z | 19 / 9 |
| 7522412867740581897 | `>>32=1751448229` | 2025-07-02 09:23:49Z | 5 / 9 |
| 7584926411887018034 | `>>32=1766003298` | 2025-12-17 20:28:18Z | 4 / 50 |

→ **绝对单调递增**，版本先后一目了然。

### 5.2 同批次连续分配（插件创建与其工具）

| ID | 解码 | counter | svr |
|---|---|---|---|
| plugin_id `7522412867740565513` | 2025-07-02 09:23:49.846Z | **4** | 9 |
| api_id `7522412867740581897` (add_audios 新版) | 2025-07-02 09:23:49.846Z | **5** | 9 |

→ 同一毫秒、同一实例、counter 连续：**插件创建与新版工具同批次分配**。

### 5.3 漂移 api_id 归属确认

`7522412878553530395` 解码 = 2025-07-02 09:23:52Z（同秒，svr=27 并发实例）
→ 属于剪映小助手 **2025-07-02 发布批次**的工具 api_id（imgs_infos 系）。
此前 per-workflow 绑定验证"精确命中漂移版本"与此一致。

### 5.4 pluginVersion 字段语义

语料 137 插件仅 1 个填了 `pluginVersion`：`7597045637133254662`（视频增强）
→ 值 `1769086552683` = **Unix 毫秒**（2026-01-22 09:35:52Z）。
该插件 id 解码 `>>32=1768825025`（2026-01-19 12:17:05Z）→ 发布在创建后约 34 小时。
**结论**: `pluginVersion` 是发布时刻的 Unix 毫秒时间戳；导出 YAML 绝大多数为空，
判定版本先后不依赖它（用 `id >> 32` 即可）。

### 5.5 时间戳与"创建时间"字段的关系（注意）

docs.coze.cn 批量查询插件 API（`/v1/plugins/mget`）示例返回的 `created_at`
与 ID 内嵌时间**不一致**（示例数据经脱敏/模拟，不可用于反推）。真实 ID 内嵌
时间以语料（真实下载）为准。

## 6. 应用：插件注册表"同名同绑定、绑最新"

1. **版本先后**: 一律 `api_id >> 32`（绝对 Unix 秒），无需私有 epoch、无需元数据。
2. **同一插件判定**:
   - `plugin_id` 相同 → 同一插件（权威），api 多版本取 `max(api_id >> 32)` 绑最新；
   - `plugin_id` 不同 + pluginName 相同 + api 集合重叠 → 判定"同插件重新发布"
     （实证: "获取音频时长"两组 `get_audio_duration` Jaccard=1.00），合并取最新；
   - pluginName 相同但 api 完全不重叠 → 巧合重名（实证: "豆瓣搜书"两组 api 各异），不合并；
   - 边界（api 名不同但功能同，如"定时器" `time_wait`/`time_waiter`）→ 保守不合并，
     标记"疑似同源"待 `name_for_model`（mget 补全）或人工确认。
3. **plugin_id 稳定、api_id 漂移为主流**: 剪映小助手 4 个 add_audios 版本同挂
   一个 plugin_id；plugin_id 本身也变的情况是"插件重新发布"（如获取音频时长）。
4. **per-workflow 绑定**（行为等价验证）与 **注册表默认绑最新**（新工作流）并存：
   - per-workflow 保留原始精确 api_id → 还原原样；
   - registry 默认 `max(id >> 32)` → 新 ncoda 脚本用最新。

## 7. 局限与后续

- svrID 14 位多实例值（0, 6, 9, 27, 40, 50 …）→ 线上并发实例差异，不影响判定。
- 每毫秒 255 上限 → 高并发下同 ms 顺序由 counter 决定。
- `name_for_model` 是平台语义唯一标识，工作流 YAML 未导出；后续可用
  `GET /v1/plugins/mget` 批量补全进注册表作权威键。
- 如需追溯"某 api_id 对应哪次发布"，可用 ID 内嵌时间戳对齐插件发布历史。

## 附: 复现

```bash
# 解码
python3 tools/coze_id.py decode 7522412867740565513 7522412878553530395
# 校验布局
python3 tools/coze_id.py check 7522412867740565513
# 编码（构造测试 ID）
python3 tools/coze_id.py encode 1751448229 846 5 9
```

## 8. 插件重新发布与 api_id 代际归属（实证补充 v1.1）

**结论**: 插件"重新发布"（新 plugin_id）时，**所有工具 api_id 全部重新分配**，
新旧代际不共享任何 api_id。旧 api_id 仍被旧工作流引用（per-workflow 还原有效），
并会以"跨代际引用"形式出现在新 plugin_id 的扫描结果中。

### 8.1 铁证: 获取音频时长（同名、同 api，两个代际）

| 代际 | plugin_id | plugin 时间 | 该代 get_audio_duration api_id | 关系 |
|---|---|---|---|---|
| 旧 | `7417655779794141219` | 2024-09-23 | `7417655779794157603` | 与 plugin **同秒**（原生） |
| 新 | `7548375061695037503` | 2025-09-10 | `7548375061695053887` | 与 plugin **同秒**（原生） |

新旧 api_id 完全不同；各代际的 api_id 与该代 plugin_id **同批次分配**（同秒，
counter 连续）。契约也有差异：旧 outputs 含 errorBody/isSuccess，新 outputs 无。

### 8.2 同 plugin_id 下 api_id 的三种归属

以剪映小助手（最新代际 `7522412867740565513`, 2025-07-02, 23 api, 3095 实例）为例：

| 类型 | 判定 | 实例 |
|---|---|---|
| 原生 | `|api_id_ts - plugin_id_ts| < 1 天`（同批次） | `add_audios 7522412867740581897`、`str_to_list 7522412878553530395`（漂移 id） |
| 旧引用 | api_id_ts 早于 plugin 数天~数月（前代际残留） | `add_audios 7457837925833834536`（2025-01 代） |
| 新引用 | api_id_ts 晚于 plugin 数天~数月（插件后续升级加工具） | `add_images 7584926411887018034`（2025-12 加） |

**→ "绑最新"必须在 plugin_id 实体内、原生优先、取 max(api_id ts)**，
禁止全局 `max(api_id>>32)`（会跨代际/跨插件绑错）。

### 8.3 多代际插件族（同名判定不够，需 api 集合聚类）

剪映系实证：同一产品线反复重新发布，10+ 个 plugin_id（2025-01 → 2025-12），
显示名各不相同（视频合成_剪映小助手 / 剪映小助手数据生成器 / 视频速创_剪映小助手 /
剪映小助手 / 剪映小助手辅助工具 …），但共享大量 api_name（add_audios /
add_captions / media_tile / text_splitter / timeline_merge / wenan_timeline_range …）。

**同源判定准则（v3）**:
1. `plugin_id` 相同 → 同一插件（权威）；
2. `pluginName` 相同 且 api 集合重叠 → 同一插件重新发布；
3. **`pluginName` 不同但 api 集合高度重叠（Jaccard ≥ 阈值）→ 同源插件族**（剪映系）；
4. 同名但 api 完全不重叠 → 巧合重名，不合并。

### 8.4 合并策略（v3，绑定到最新代际）

```
1. 同源判定: pluginName 相同 OR api 集合 Jaccard ≥ 阈值
2. 最新代际:   max(plugin_id >> 32)
3. api_id:    最新代际 plugin 内, 原生优先(|api_ts - plugin_ts| < 1 天),
              否则 max(api_id >> 32)
4. 契约:      以最新代际 api 记录为准
5. per-workflow 绑定: 保留原始 (plugin_id, api_id), 行为等价验证用
```

"原生"判定完全由 ID 内嵌时间戳自动完成，无需人工维护映射。
