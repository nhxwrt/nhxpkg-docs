# nhxsta Observability

> **Phase E (PR-S5c-10) wrap** — landed in nhxwrt-src-dev across PR-S5c-1..9
> (#77 / #78 / #79 / #80 / #81 / #83 / #84 / #86 / #87). 本文是 nhxsta 14 个
> `debug.*` ubus 方法 + JSON Schema 契约 + history ring 的统一参考。
>
> Audience: 运维 / AI agent / nhxvue 前端 / future maintainers。

## 1. 心智模型

```
┌──────────────────────────────────────────────────────────────────┐
│ ubus client (CLI / nhxvue / AI agent / lua RPC)                  │
└────────────────────────┬─────────────────────────────────────────┘
                         │ ubus call nhxsta debug.<method> {sta:N}
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│ state_lib (src/state/nhx_sta_state.c) — 14 handler              │
│   ├ debug.identity        (caps[] self-discovery)                │
│   ├ debug.snapshot        (聚合 8 sub-tables)                     │
│   ├ debug.{apdb,rssi,scan_state,roam,btm,rrm,wpa,hal} ─┐        │
│   └ debug.{error,roam,btm,rrm}_history (4 ring buffer)  │        │
└─────────────────────────────────────────────────────────┼────────┘
                                                          │
                          ┌───────────────────────────────┘
                          ▼
┌──────────────────────────────────────────────────────────────────┐
│ L2 库 *_to_reply_local (state.c file-static) 或 库公开 to_reply  │
│   apdb_to_reply (decision/) ─── nhx_sta_sm_roam_to_reply (sm.c) │
│   rssi_to_reply (decision/) ─── btm_to_reply_local              │
│   crest_to_reply (scan/)    ─── rrm_to_reply_local              │
│                                  wpa_to_reply_local              │
│                                  hal_to_reply_local              │
└──────────────────────────────────────────────────────────────────┘
```

**核心设计原则**（贯穿 PR-S5c-1..9 所有 round 修订收敛）：

1. **Schema 是 single source of truth** — `schemas/*.json` 定义契约，C 实现追随。
2. **Drift-free 三处 lockstep** — schema `maximum` ↔ C `#define` cap ↔ wire emit 必须经命名常量同步（如 `STA_CHANNEL_NUM_MAX = 255` 在 utils.h、roam.json `maximum:255`、hal.json `maximum:255`、sm.c `NHXSTA_ROAM_CHANNEL_MAX = STA_CHANNEL_NUM_MAX` 四处一致）。
3. **One canonical degrade sentinel** — 同一 init 状态门控的多字段共享同一 sentinel（如 KVR mode 下 `crest_state="UNKNOWN"` ↔ `crest_profile="--"` ↔ `crest_running=0`）。
4. **Schema_version=1 演化** — 新字段进 `properties` 不进 `required`，避免 "same version different contract" 漂移。
5. **KVR-safe degrade（部分 handler）** — 多数 handler 在 sm/db/listener 未 init 时返回 schema-conformant 退化 reply，不发 partial reply 也不返 INVALID_ARGUMENT。**例外**：`debug.scan_state` / `debug.rssi` standalone 调用仍受 `wireless_is_sta_valid(idx)` 约束在 KVR mode 下 hard-reject `UBUS_STATUS_INVALID_ARGUMENT`（参 `schemas/{scan_state,rssi}.json` description）；这两个方法仅在 `debug.snapshot` 内的 sub-table 路径走 degrade。后续 PR 让 standalone handler 改 KVR-safe degrade 后此例外消失。

---

## 2. 方法清单

| 方法 | Schema | 入参 | 输出形式 | KVR 适配 |
|---|---|---|---|---|
| `debug.identity` | `identity.json` | — | `{schema_version, binary, version, caps[]}` | ✓ caps 缩减反映已注册方法 |
| `debug.snapshot` | `snapshot.json` | `{sta:int}` | identity + status + 8 sub-tables | ✓ 每 sub-table 独立 degrade |
| `debug.apdb` | `apdb.json` | `{sta:int}` | sta_index + anchors[] | KVR db==NULL → degrade `sta_index="-1"` + `anchors=[]` |
| `debug.rssi` | `rssi.json` | `{sta:int}` | state + rssi_dbm + rssi_valid + bssid + cache_age_ms | **standalone 在 KVR 下 hard-reject INVALID_ARGUMENT**（仅 snapshot.rssi sub-table 走 degrade） |
| `debug.scan_state` | `scan_state.json` | `{sta:int}` | state + profile_tag + running + 各 sub-module | **standalone 在 KVR 下 hard-reject INVALID_ARGUMENT**；snapshot.scan_state sub-table 走 `crest_to_reply` degrade `state="UNKNOWN"` |
| `debug.roam` | `roam.json` | `{sta:int}` | in_progress + trigger + target_* + stats | KVR sm_get_ctx==NULL → `in_progress=0` + 空 BSSID |
| `debug.btm` | `btm.json` | `{sta:int}` | listener_initialized + has_last_request + last_dialog_token + ... | listener 是 process-scope，两 mode 通用 |
| `debug.rrm` | `rrm.json` | `{sta:int}` | listener_initialized + complete_count + has_last_request + last_dialog_token | 同 btm |
| `debug.wpa` | `wpa.json` | `{sta:int}` | wpa_state + ssid + bssid + monitor_initialized | wpa_supplicant 未跑 → `wpa_state="UNKNOWN"` + 空字符串 |
| `debug.hal` | `hal.json` | `{sta:int}` | drv_scan_/bss_nl_/vendor_nl_initialized + handler_count + per-STA RF metrics | KVR bss_nl 通常未 init → `bss_nl_initialized=0` |
| `debug.error_history` | `error_history.json` | `{sta:int, n?:int=10}` | errors[] (按 ts_ms 倒序，per-STA 过滤) | 两 mode 通用 |
| `debug.roam_history` | `roam_history.json` | 同上 | roams[] (per-STA) | 两 mode 通用 |
| `debug.btm_history` | `btm_history.json` | 同上 | btms[] (per-STA) | 两 mode 通用 |
| `debug.rrm_history` | `rrm_history.json` | 同上 | rrms[] (per-STA) | 两 mode 通用 |

**入参约定**：
- `sta` 整数（如 `0`）或字符串（如 `"sta0"`），通过 `nhxubus_parse_idx` 双形式 resolver 统一解析。
- 整数路径不做 range 校验，handler 必须显式 `idx >= 0 && idx < NHX_MAX_STA_COUNT` bounds check（Cursor learned rule "STA idx bounds: always validate"）。
- `n` 在 `*_history` 默认 10，最大 32（`HISTORY_RING_CAP`）。

---

## 3. Snapshot 聚合契约

`debug.snapshot` 是 nhxvue / AI agent 主入口 — 一次调用拿全 STA 状态：

```json
{
  "schema_version": "1",
  "binary": "nhxsta",
  "version": "1.2.3",
  "sta": 0,
  "state": "SCAN_NORMAL",
  "bssid": "aa:bb:cc:dd:ee:ff",
  "ssid": "MyAP",
  "freq": 5180,
  "crest_state": "RUNNING",        // LEGACY (优先用 scan_state.state)
  "crest_profile": "default",
  "crest_running": 1,
  "apdb":       { schema_version, sta, sta_index, anchors[] },
  "rssi":       { schema_version, sta, state, rssi_dbm, rssi_valid, bssid, cache_age_ms },
  "scan_state": { schema_version, sta, state, profile_tag, running, ... },
  "roam":       { schema_version, sta, in_progress, trigger, target_bssid, target_channel, target_freq, old_bssid, attempts, successes },
  "btm":        { schema_version, sta, listener_initialized, has_last_request, last_dialog_token, ... },
  "rrm":        { schema_version, sta, listener_initialized, complete_count, has_last_request, last_dialog_token, ... },
  "wpa":        { schema_version, sta, wpa_state, ssid, bssid, monitor_initialized },
  "hal":        { schema_version, sta, drv_scan_initialized, bss_nl_initialized, vendor_nl_initialized, vendor_nl_handler_count, frequency_mhz, channel, noise_dbm, txrate_kbps }
}
```

> 上方仅展示 sub-table 入口字段，详细 properties / required 见各 `schemas/<x>.json`。

**字段 byte-identical 保证**：每个 sub-table 内部布局与对应 `debug.<x>` 单调用输出**逐字节一致**（含 `schema_version` + `sta` wrapper 字段），由共享 `*_to_reply` helper 实现。Consumer 可以 prefer snapshot 路径减少 RTT 而不必担心字段差异。

**Caveat**（PR-S5c-3 round 5 记录）：`scan_state` 与 `rssi` sub-table byte-identical 仅适用 NHXROAM + sta valid 成功路径。KVR mode 下 standalone `debug.scan_state` / `debug.rssi` 仍受 `wireless_is_sta_valid` 约束 hard-reject `UBUS_STATUS_INVALID_ARGUMENT`，而 snapshot 内 sub-table 走 `crest_to_reply` / 退化路径返回 schema-conformant 对象。后续 PR 让 standalone handler 改 KVR-safe degrade 后此 caveat 消失。

**LEGACY 字段**：`crest_state` / `crest_profile` / `crest_running` 顶层字段在 PR-S5c-3 之前已存在，与 `scan_state` sub-table 同源（共享 degrade sentinel）。前端代码迁移完成前保留兼容；新代码优先用 sub-table 路径（drift-free）。

---

## 4. History Ring 契约

4 类 ring buffer（`src/state/nhx_sta_state.c`），每类容量 `HISTORY_RING_CAP = 32`：

| Ring | Recorder API (in `nhx_sta_state.h`) | 字段 |
|---|---|---|
| `error_history` | `nhx_sta_state_record_error(idx, lib_name, nhxubox_result_t r)` | ts_ms, idx, lib_name, kind, code, detail |
| `roam_history` | `nhx_sta_state_record_roam_result(idx, status, from_bssid, to_bssid, from_freq, to_freq)` | ts_ms, idx, status, from_/to_bssid, from_/to_freq |
| `btm_history` | `nhx_sta_state_record_btm_request(idx, dialog_token, neighbor_count)` | ts_ms, idx, dialog_token, neighbor_count |
| `rrm_history` | `nhx_sta_state_record_rrm_request(idx, meas_type, op_class, chnum, dialog_token)` | ts_ms, idx, meas_type, op_class, chnum, dialog_token |

**Per-STA 过滤模式**（PR-S5c-6 btm round 1 → PR-S5c-7 rrm 复用）：

```c
for (int i = 0; i < count && emitted < n; i++) {
    int slot = (head - 1 - i + cap * 2) % cap;   // 反向倒序 (newest-first)
    if (buf[slot].idx != idx) continue;          // STA 过滤
    /* defensive sanitize + emit */
    emitted++;
}
```

Ring 是 process-scope 共享，单纯按 idx 过滤可避免双 STA 场景下 history 串扰。

**Recorder 内置 saturation**（PR-S5c-7 round 2 修复 int overflow wrap）：所有 counter 用 `uint32_t` + `if (count != UINT32_MAX) count++` 防止 32 位溢出回 0。

**`record_error` skip success**：当 `nhxubox_result_t r` 是 `ERR_NONE` (`NHXUBOX_RESULT_OK(r)`) 时直接 return — error_history ring 只关心 error 路径。所有 idx 越界（`idx < 0 || idx >= NHX_MAX_STA_COUNT`）的 record 调用也被静默丢弃保 caller bug 不污染 ring。

**`NHX_LIB_CALL` 宏**（`nhx_sta_state.h:142`）：包装 lib API 调用自动 record_error。任何 `NHX_LIB_CALL(lib_func(args), idx, "lib_name")` 失败时自动写 ring，调用方无需手动 record。

---

## 5. Drift-free 不变量（Cursor learned rules 累积）

PR-S5c-1..9 共 60+ 轮 review 沉淀的核心规则，未来 PR 必须遵守：

| 不变量 | 实例 | 检查方法 |
|---|---|---|
| C clamp + schema maximum lockstep via `#define` | `STA_CHANNEL_NUM_MAX = 255` ↔ `roam.json target_channel.maximum:255` ↔ `hal.json channel.maximum:255` ↔ `sm.c NHXSTA_ROAM_CHANNEL_MAX = STA_CHANNEL_NUM_MAX` | grep `\b255\b` 三处一致 |
| BSSID lowercase normalized before emit | `bssid_to_lower_inplace(buf)` + `bssid_is_zero(buf)` → `""` | 所有 BSSID 输出经此 pattern |
| One canonical degrade sentinel per init flag | KVR mode `crest_state="UNKNOWN"` / `profile="--"` / `running=0` 全由 `crest_is_initialized` 门控 | 同 init 标志的多字段不能各 degrade 各的 |
| schema_version=1 演化只加 properties 不加 required | PR-S5c-4 round 5 fix:snapshot.json 新 sub-tables 在 properties 不在 required | grep `"required"` array 跨版本不变 |
| STA idx bounds check 显式 | handler 入口 `if (idx < 0 \|\| idx >= NHX_MAX_STA_COUNT) return UNKNOWN_ERROR;` | grep `nhxubus_parse_idx` 后必须有 bounds check |
| char[N] string field maxLength=N-1 | `wpa_state` char[32] → schema `maxLength:31` | C struct 定义和 schema 必须配对 |
| uint32_t printf 用 PRIu32 | `#include <inttypes.h>` + `"%" PRIu32` | grep `%u` 后看变量类型；ABI 上 unsigned long 时 `%u` 错位 |
| Forward-declare file-static helper before earlier handler use it | snapshot 在 btm/rrm/wpa/hal_to_reply_local 之前调用，必须在文件顶 forward declare | 编译错误立即暴露 |
| Doc comment ↔ impl 同步 | `rrm_get_complete_count` 注释说"未 init 返 0"，impl 必须真返 0 | review 时 doc 和 impl 一起读 |
| Sub-table 描述里不能列未实现字段 | `snapshot.json hal.description` 不能提 `rxrate` 因 schema/impl 都不暴露 | grep description 关键字 vs `properties` |

---

## 6. Schema 维护

**位置**：`qsdk/nhxpkg/nhxsta/nhxsta/schemas/`

**约束**（参 [schemas/README.md](../nhxsta/nhxsta/schemas/README.md)）：

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "nhxsta/<filename>.json",
  "title": "nhxsta <method-name>",
  "type": "object",
  "properties": {
    "schema_version": { "const": "1" },
    ...
  },
  "required": ["schema_version", ...],
  "additionalProperties": false
}
```

- 字段名 **snake_case**
- 带单位字段加后缀：`rssi_dbm` / `freq_mhz` / `age_ms` / `ts_ms` / `txrate_kbps`
- 必填字段显式列入 `required`
- 演进通过 `schema_version` 管控（当前固定 `"1"`，破坏性变更才 bump）

**Sub-table $ref 模式**（`snapshot.json` 引各 sub schema）：

```json
"apdb": { "$ref": "apdb.json", "description": "..." }
```

`$ref` 委托完整字段验证给 sub schema，drift-free（一处改两处自动同步）。

---

## 7. Consumer 接入

### nhxvue 前端 (TypeScript)

```bash
quicktype --src qsdk/nhxpkg/nhxsta/nhxsta/schemas/*.json \
          --lang typescript -o src/types/nhxsta.ts
```

生成 `Snapshot` / `Apdb` / `Rssi` / ... 类型，与 ubus 输出 schema-conformant 一致。

### AI agent

直接读 `schemas/*.json` 元数据：
- `description` 字段语义、KVR caveat、degrade 行为
- `properties.<field>.description` 单字段语义和单位
- `required` 哪些字段保证存在

```bash
ubus call nhxsta debug.identity                         # 先查 caps[] 看哪些方法可用
ubus call nhxsta debug.snapshot '{"sta":0}'              # 全状态一次取
ubus call nhxsta debug.roam_history '{"sta":0,"n":5}'    # 最近 5 次漫游
```

### CI 校验（OOS but recommended）

```bash
ubus call nhxsta debug.snapshot '{"sta":0}' > /tmp/snap.json
ajv validate -s schemas/snapshot.json -d /tmp/snap.json
```

集成到 R3-device 验证流程后可挡 schema 漂移。

---

## 8. 调试 Cookbook

**全状态快照**：
```bash
ubus call nhxsta debug.snapshot '{"sta":0}'
```

**追踪漫游失败**：
```bash
ubus call nhxsta debug.roam_history '{"sta":0,"n":10}' \
    | jq '.roams[] | select(.status != 1)'
```

**找最近 lib 错误**：
```bash
ubus call nhxsta debug.error_history '{"sta":0,"n":10}' \
    | jq '.errors[] | {ts_ms, lib_name, kind, detail}'
```

**KVR mode 状态确认**：
```bash
ubus call nhxsta debug.identity | jq '.caps'
# nhxsta binary 当前 emit 13 caps:
#   snapshot, apdb, rssi, scan_state, roam, btm, rrm, wpa, hal,
#   error_history, roam_history, btm_history, rrm_history
# (sm_ctx 在 identity.json enum 内是 forward-compat 占位, 当前 handle_identity 不 emit;
#  PR-S7 nhxroam binary 落地 debug.sm_ctx handler 后才进 caps[])
# 未来 nhxkvr binary: caps 缩减（无 apdb / sm_ctx / periodic_scan）
```

**当前 RSSI 快照**：
```bash
ubus call nhxsta debug.rssi '{"sta":0}' | jq '{state, rssi_dbm, rssi_valid, bssid, cache_age_ms}'
# rssi.json 字段: state (READY/NOT_CONNECTED/...), rssi_dbm (string), rssi_valid (0/1),
# bssid (lowercase or empty), cache_age_ms (string -1=miss / 0=now / N=ms ago).
# 无 samples[] 字段; rssi_lib 内部维护 50ms micro-cache 但不通过 wire 暴露历史窗口.
```

---

## 9. 后续 Phase F (PR-S6+) 规划

| PR | 影响 |
|---|---|
| PR-S6 (nhxkvr binary) | `debug.identity.caps` 在 KVR binary 缩减；apdb / sm_ctx 不注册 |
| PR-S7 (nhxroam binary + sm.c 私有化) | `debug.sm_ctx` 真实 handler 在 nhxroam/sm_ctx_handler.c (非 state_lib)；snapshot 加 `sm_ctx` sub-table（仅 nhxroam binary） |
| PR-S8 (SM 死分支清理) | `debug.scan_state.state` enum 减少（删 SCAN_RRM/BTM 后） |

**契约稳定性承诺**：上述 PR 不破坏现有 14 个 method 的 schema_version=1 契约 — 字段只增不减 / 只放宽不收紧 / `required` 集不变。任何破坏性变更必须 bump `schema_version` 到 `"2"`（同步 `NHXSTA_SCHEMA_VERSION` + `identity.json` const + 各 schema const）。
