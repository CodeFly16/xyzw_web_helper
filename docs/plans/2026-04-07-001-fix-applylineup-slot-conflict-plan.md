---
title: "fix: 修复 applyLineup 中上阵目标武将和应用鱼灵配置失效问题"
type: fix
status: completed
date: 2026-04-07
---

# fix: 修复 applyLineup 中上阵目标武将和应用鱼灵配置失效问题

## Overview

`applyLineup` 在执行「上阵目标武将」和「应用鱼灵配置」步骤时出现应用不完整的情况。核心原因是 `placeHeroes` 逐一处理英雄时未考虑槽位冲突，导致英雄被下阵后无法重新上阵，进而导致鱼灵配置因英雄不在队伍中而失败。

## Problem Frame

### 上阵目标武将失效

`placeHeroes` 对每个已在队伍但位置错误的目标英雄，采用「先下阵 → 再上阵」的单步处理：

```
处理英雄 A（当前在 slot 0，目标 slot 1）：
  → hero_gobackbattle(slot=0)   // A 被下阵
  → hero_gointobattle(slot=1)   // 失败！slot 1 此时仍被英雄 B 占据
  // 结果：A 被移出队伍，但未能上阵
```

当目标阵容需要英雄之间互换位置时，目标 slot 始终被尚未处理的其他英雄占据，导致 `hero_gointobattle` 必然失败。英雄被下阵后丢失，重试时这个英雄的状态已从「位置错误」变成「不在队伍中」，虽然会被重新上阵，但与其他英雄的位置仍可能再次冲突，3 次重试均可能失败。

### 应用鱼灵配置失效

- **直接依赖上阵步骤**：`artifact_load` 需要英雄在队伍中（或至少可访问）。上阵部分失败时，对应英雄的鱼灵无法装备。
- **成功提示时序错误**：`message.success('阵容已应用')` 在鱼灵配置步骤**执行前**就被调用（第 2169 行），导致用户看到成功提示但鱼灵实际未完成。
- **失败静默跳过**：当 `artifactId` 查找失败时（`if (!artifactId) continue`）既无日志也无错误收集，用户无法感知。

## Requirements Trace

- R1. 目标阵容中所有英雄需在正确位置上阵，无论其初始位置如何分布
- R2. 鱼灵配置在英雄上阵成功后应用，且应用失败情况要反馈给用户
- R3. 阵容应用完成的成功提示应在所有步骤（含鱼灵）均完成后才显示

## Scope Boundaries

- 不修改协议层（`hero_gointobattle`、`hero_gobackbattle`、`artifact_load` 等接口不变）
- 不修改「移除非目标武将」、「应用等级配置」、「同步科技配置」等步骤逻辑
- 不重构进度弹窗 UI 结构

## Context & Research

### 关键代码位置

- `placeHeroes` 函数定义：`src/components/cards/Unlimitedlineup.vue:2056-2098`
- 调用及重试逻辑：`src/components/cards/Unlimitedlineup.vue:2100-2128`
- 鱼灵配置块：`src/components/cards/Unlimitedlineup.vue:2172-2327`
- 成功提示（时序问题）：`src/components/cards/Unlimitedlineup.vue:2166-2170`
- `COMMAND_DELAY = 500ms`，`MAX_PLACE_RETRIES = 3`

### 现有模式

- `fetchLatestData()` 负责拉取最新的 roleInfo 和 presetTeam，在各步骤间复用
- `errors[]` 数组用于收集错误信息，最终通过 `message.warning` 展示
- `getTeamHeroes(teamInfo)` 将服务端格式转换为 `{ heroId, position, artifactId, attachmentUid }` 数组

## Key Technical Decisions

- **两阶段上阵（Two-phase placement）**：先批量下阵所有「位置错误」的目标英雄，再批量上阵所有目标英雄。Phase 1 清空干扰槽位，Phase 2 放置时所有目标槽位均已腾空，不再有冲突。这比拓扑排序简单且对各种位置组合均有效。
- **不新增 API 调用类型**：继续使用现有的 `hero_gobackbattle` / `hero_gointobattle`，只调整调用顺序。
- **成功提示后移**：将全局成功提示移至「刷新队伍信息」步骤前（所有配置步骤都完成后），保持局部成功提示（如 `fishApplied` 条数）不变。

## Open Questions

### Deferred to Implementation

- 服务端对同一 slot 的快速连续操作是否有节流限制：`COMMAND_DELAY = 500ms` 应已覆盖，若仍失败可在 Phase 2 适当增加 delay，实现期观察。
- `hero_gobackbattle` 的 slot 参数是 0-indexed 还是 1-indexed：当前代码已在使用，保持一致即可，无需修改。

## Implementation Units

- [x] **Unit 1: 修复 `placeHeroes` 为两阶段上阵逻辑**

**Goal:** 消除「上阵目标武将」步骤中的槽位冲突，确保所有目标英雄能正确上阵至目标位置

**Requirements:** R1

**Dependencies:** 无

**Files:**

- Modify: `src/components/cards/Unlimitedlineup.vue`（`placeHeroes` 函数，约第 2056-2098 行）

**Approach:**

将 `placeHeroes` 内部拆分为两个阶段：

1. **Phase 1 — 下阵位置错误的英雄**：遍历 `heroesList`，找出已在队伍中但 `position` 不匹配的英雄，对它们全部调用 `hero_gobackbattle`，每次调用后等待 `COMMAND_DELAY`。
2. **Phase 2 — 上阵所有目标英雄**：遍历 `heroesList`，对不在队伍中的英雄调用 `hero_gointobattle`（包含 Phase 1 中被下阵的英雄）。
   - Phase 2 需要一份「Phase 1 下阵后的队伍状态」来判断哪些英雄仍在队伍中。可以从传入的 `latestTeam` 中排除 Phase 1 下阵的英雄来构造这份状态，无需额外网络请求。

调用处（`await placeHeroes(targetHeroes, currentHeroes)` 和重试循环中 `await placeHeroes(mismatches, verifyHeroes)`）不需要修改。

**Test scenarios:**

- Happy path — 目标英雄两两互换位置（A@0↔B@1）：Phase 1 下阵两者，Phase 2 上阵到正确位置，最终验证均到位
- Happy path — 目标英雄全部不在队伍中：Phase 1 无操作，Phase 2 全部上阵
- Happy path — 目标英雄位置完全匹配：Phase 1 和 Phase 2 均无操作
- Edge case — 部分英雄位置正确、部分需移动：仅对位置错误的英雄执行两阶段，正确的跳过
- Edge case — 5 英雄阵容全部需要重新排列：所有英雄先下阵再上阵，验证无英雄丢失
- Integration — 修复后，原来需要 3 次重试的互换场景：第 1 次 `placeHeroes` 调用后即通过校验，无需触发重试

**Verification:**

- 对「目标英雄位置互换」场景，单次 `placeHeroes` 调用后，重试校验循环中 `mismatches.length === 0`，不触发任何重试
- 所有目标英雄均在目标位置上阵（`verifyHeroes` 完全匹配 `targetHeroes`）

---

- [x] **Unit 2: 修复成功提示时序 + 将鱼灵失败纳入 errors 收集**

**Goal:** 成功提示在所有配置步骤完成后才显示；鱼灵无法找到 artifactId 时记录到 `errors`，而非静默跳过

**Requirements:** R2, R3

**Dependencies:** Unit 1（逻辑上独立，但修复上阵才能让鱼灵逻辑被充分验证）

**Files:**

- Modify: `src/components/cards/Unlimitedlineup.vue`（第 2166-2170 行的成功提示块，第 2212 行的 `continue` 处）

**Approach:**

1. **移除 `message.success` 的中间调用**（第 2166-2170 行）：删除这个提前触发的全局成功提示。保留各步骤的局部提示（`fishApplied`、`levelApplied` 等）。
2. **在「刷新队伍信息」步骤前（约第 2396 行前）添加全局提示**：

   ```
   if (errors.length > 0) {
     message.warning(`阵容已应用，但有部分错误:\n${errors.join('\n')}`)
   } else {
     message.success(`阵容 "${lineup.name}" 已应用`)
   }
   ```

   这段逻辑已存在（第 2166-2170 行），只需将其移到正确位置。
3. **鱼灵 `artifactId` 查找失败时记录错误**：在 `if (!artifactId) continue` 处，改为 `errors.push(...)` 后再 `continue`，说明哪个英雄的哪个鱼灵/珍珠无法解析。

**Test scenarios:**

- Happy path — 所有步骤成功：成功提示在刷新队伍信息步骤完成后出现，时序正确
- Happy path — 有鱼灵配置，全部成功：局部 `fishApplied` 提示正常，全局成功提示在刷新后出现
- Error path — 某英雄的 fishId/pearlId 无法映射到 artifactId：`errors` 数组包含该条目，最终提示 warning 而非 success
- Integration — 上阵部分失败导致鱼灵步骤失败：`errors` 中能看到鱼灵失败项（Unit 1 修复后此场景应消除）

**Verification:**

- 调试模式下观察消息弹出顺序：不应在鱼灵配置开始前出现「阵容已应用」提示
- 当存在无法映射的 fishId 时，最终弹出 warning 而非 success，且 warning 文本中包含相关英雄信息

## System-Wide Impact

- **Interaction graph:** 仅影响 `applyLineup` 函数内部逻辑，不影响其他方法
- **Error propagation:** `errors[]` 数组已有统一的错误收集和展示机制，Unit 2 在现有机制内扩展
- **Unchanged invariants:** `fetchLatestData`、`getTeamHeroes`、进度弹窗、重试循环（结构不变）、所有 WS 命令接口均不变
- **API surface parity:** 无新增外部接口

## Risks & Dependencies

| Risk | Mitigation |
|------|------------|
| Phase 1 批量下阵后，Phase 2 上阵前服务端有短暂延迟导致槽位仍显示占用 | 现有 `COMMAND_DELAY` 已有 500ms 缓冲；如仍有问题可在 Phase 1 末尾额外 await delay |
| 修改 `placeHeroes` 逻辑后，重试循环的 `placeHeroes(mismatches, verifyHeroes)` 传入的是子集，Phase 1 可能错误地下阵不在子集中但位置正确的其他英雄 | Phase 1 只处理 `heroesList` 中位置错误的英雄，不会影响未传入的英雄——逻辑是安全的 |

## Sources & References

- 相关代码：`src/components/cards/Unlimitedlineup.vue`（`applyLineup` 函数，约第 1844-2407 行）
