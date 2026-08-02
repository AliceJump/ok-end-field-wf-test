# LANG_SYNC_TEST_REPORT

多语言同步工作流完整实测报告（测试仓库 `AliceJump/ok-end-field-wf-test`，分支 `test/lang-sync-validation`，2026-08-03）

## 1. 实际执行顺序

全部使用真实 API（zonai.skport.com / zonai.skland.com / endfield.wiki.gg），无 mock：

```
capture_wiki_catalog.py（7 语 13 模板组合，playwright chromium 真实抓取）
    ↓
capture_wiki_zhcn.py（skland 15 类简中抓取）
    ↓
sync_world_map_langs.py（zonai map/list + catalog API）
    ↓
sync_map_mark_langs.py（zonai map/mark/list API，170 模板）
    ↓
sync_wiki_item_langs.py（读取 tools/wiki_catalog + map_marks.json）
    ↓
sync_character_langs.py（endfield.wiki.gg）
```

脚本入口确认：

| 脚本 | 输入 | 输出 | 修改文件 | 依赖 |
|-|-|-|-|-|
| capture_wiki_catalog.py | zonai.skport.com 真实抓取 | tools/wiki_catalog/by_lang/<ts>_{7语}/*.json + summary | 无 | playwright chromium |
| capture_wiki_zhcn.py | zonai.skland.com 真实抓取 | tools/wiki_catalog/zh_cn/<ts>/*.json + summary | 无 | playwright chromium |
| sync_world_map_langs.py | zonai map/list、map/catalog | tools/official_five_lang.json（快照） | assets/lang/world_map.json、assets/data/world_map.json、i18n/*.po | 代理直连 API |
| sync_map_mark_langs.py | zonai map/mark/list（6 语） | assets/data/map_marks.json | assets/lang/*.json（除 map_marks.json）、assets/items/map/item_names.json、i18n/*.po | 代理直连 API |
| sync_wiki_item_langs.py | tools/wiki_catalog/ 捕获数据 | assets/data/wiki_items.json | assets/lang/*.json（除 map_marks.json/wiki_items.json）、i18n/*.po | 捕获数据 + map_marks.json |
| sync_character_langs.py | endfield.wiki.gg 30 干员 | 无独立产物 | assets/lang/characters.json、assets/data/characters.json | 网络直连 |

## 2. 文件影响（第一轮实际执行结果）

| 脚本 | 输出 | 实际修改 |
|-|-|-|
| sync_world_map_langs.py | tools/official_five_lang.json（新建快照） | world_map.json 0 变更（数据已最新）；po 0 变更 |
| sync_map_mark_langs.py | assets/data/map_marks.json（新建） | templates 170，zh 键 153，added 918，changed 0；po 0 变更 |
| sync_wiki_item_langs.py | assets/data/wiki_items.json（新建） | 1079 zh 键，added 6586，changed 0；po 0 变更 |
| sync_character_langs.py | 30 干员 | characters.json 0 变更（幂等） |

world_map.json before/after SHA256：`E6856FC1384B4714` / `E6856FC1384B4714`（无变化，数据已最新）。

## 3. 数据数量

```
world_map:    nodes 88（5 语全覆盖 87，仅 zh 1）
map_marks:    templates 170 → zh_CN 键 153（6 语全覆盖 153/153）
wiki_items:   zh_CN 键 1079（8 语全覆盖 173，9 语含 es 0）；added 6586
              zh_CN matched 897 / unmatched 857（skland-only 条目）
characters:   operators 30（6 语全覆盖）
```

抽查（真实值）：
- map_marks：储藏箱I→Storage Crate I/資源箱Ⅰ/보물 상자 I/Caja de almacenamiento I；总控中枢→Control Nexus；塔晶→Talosite 等 10 模板全部 6 语完整
- wiki_items：佩丽卡（干员）→Perlica/ペリカ/펠리카/Перлика；爆破单元（武器）→Detonation Unit；重黯石子簇（材料）→Umbronyx Seed；骑士精神（道具）→Chivalric Virtue
- characters：lang_wei→Wulfgard/ウルフガード/울프가드；mi_fu→Mi Fu/ミ・フ/미브 等 5 角色 6 语完整

## 4. 优先级测试（真实破坏验证）

测试名：`酸液源石虫·α`（map_marks 官方 en="Acid Originium Slug α"，wiki_items 官方 en="Acid Originium Slug·α"）

```
步骤 A1：map_marks 先跑 → k_priority en = "Acid Originium Slug α"（marks 值）
步骤 A2：wiki_items 再跑（map_marks.json 存在）→ 值不变 = "Acid Originium Slug α"  PASS（pop 机制生效）
步骤 B ：删除 map_marks.json 后跑 wiki_items → 值被覆盖 = "Acid Originium Slug·α"  FAIL 条件复现
步骤 C ：重跑 map_marks → 值恢复 = "Acid Originium Slug α"
```

结论：**wiki_items 的覆盖阶段确实依赖 map_marks.json 存在**（sync_wiki_item_langs.py:243-247 运行时 pop 其名字）。顺序正确时优先级成立（map_marks > wiki_items）。

## 5. Action 实测

| workflow | run id | 分支 | result | 时间 |
|-|-|-|-|-|
| update-world-map-langs.yml | 30768116648 | test/lang-sync-validation（已含最新数据） | success | 7m46s |
| update-world-map-langs.yml | 30768428428 | test/lang-sync-validation（故意写坏 k_priority en="WRONG_CI_PROBE"） | success | ~8m |

有效 diff 验证：30768428428 的 auto-commit `974d977` 将 `WRONG_CI_PROBE` 修复为 `Acid Originium Slug α`（diff 恰好 1 行），证明 CI 产生有效 diff 并自动提交。

幂等验证：数据最新时（30768116648）CI 无 diff 不产生 commit——与本地第二轮 0 变更一致。

## 6. 发现的问题

1. **【中等】wiki_items 覆盖阶段依赖 map_marks.json 存在**：若 map_marks 步骤失败/未运行（网络、脚本报错），wiki 会把地图点位名按 wiki 译名覆盖到其他 lang JSON（实测 `酸液源石虫·α` 被改为 "Acid Originium Slug·α"）。当前 workflow 顺序正确时无风险，但单点依赖无护栏。
2. **【低】po 文件行尾噪音**：本地 Windows（CRLF checkout）上脚本重写 ok.po 为 LF，git 显示 M 但内容无变化（`--ignore-space-at-eol` 为空）。CI 无此问题（LF checkout）。
3. **【低】zh_CN unmatched 857/1788（48%）**：skland-only 条目（科学兴农、贴纸·…、M.I.警用…壹型等）无 wiki 官方对应，保持 zh_CN-only 不补——预期行为，但占比高，可后续评估是否单独补译。
4. 【无】官方 map_marks 数据 储藏箱 I–IV 英文正确（Storage Crate I–IV），无错位。

## 7. 建议修改

1. sync_wiki_item_langs.py：map_marks.json 不存在时打印 WARNING 并跳过 `sync_other_lang_jsons` 覆盖阶段（而不是静默失去保护）；或改为在工作流中 map_marks 失败即 fail。
2. 脚本写 .po/.mo 时保留原文件行尾（或仓库加 `.gitattributes` 统一 eol），消除本地噪音。
3. 可选：workflow 增加步骤级依赖断言（wiki_items 步骤前检查 map_marks.json 产物存在）。
