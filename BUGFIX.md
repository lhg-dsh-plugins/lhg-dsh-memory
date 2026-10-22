# Bug 修复说明：T3 全量兜底把 score=0.0 的零相关条目随真命中一并返回

对应上游 issue [#637](https://github.com/furongjun-1999/dsh-memory/issues/637)
（open，P2 检索质量；与 lingshu #795 同构、该仓维护者已判定成立）。
本版只修这一处。

## 现象

`MdCG.search` 的前级（桶内 LIKE / 全局 LIKE）无命中时落到 **T3_global_scan**
全量兜底。该出口不设最低分阈值：只要库里有一条真命中（score>0），其余与
查询**零字符/零 bigram 交集**的条目会以 `score=0.0` 随结果按 k 截断返回。
下游（Agent 自动召回注入）拿到 0 分条目时无法区分「有相关记忆」与「库里
有字就回」，零相关上下文以与命中条目同构的形态进入 prompt。

既有守卫 #89②（`MdCG._zero_filler_only`）只处理「全库无真命中」一种情形
（此时上界压到 0、返回空集）；「有真命中 + 零分填充同行」正是它 docstring
里明示不裁的缝隙——本 issue 裁定该缝隙即缺陷。

## 证据链

基线：上游 HEAD `b278362da570b0cc646a1468f6f628334cb67580`（package 0.9.11）。

> 说明：issue 原复现（连字正文 `ﬁnished` × 查询 `finished`）在当前 HEAD 上
> 已被 T2 提前命中（相关归一在基线之后有演进），不再落 T3。故按下述同构
> 构造复现：查询词与正文共享 bigram 但词干不是其子串 ⇒ LIKE 零命中落 T3，
> 且 T3 打分出一正三零，与 issue 的分数断层形态一致。

最小复现（仓根执行，`python3 -X utf8 repro.py`）：

```python
import tempfile
from md_cg.mdcg import MdCG

cg = MdCG(tempfile.mkdtemp(prefix="t3b_"))
cg.add("n_partial", "banana split sundae", layer="knowledge")   # 与 bandana 共享 bigram
cg.add("n_junk1", "北京 发布 窗口 每周三", layer="knowledge")     # 零重叠
cg.add("n_junk2", "giraffe puzzle kiosk", layer="knowledge")    # 零重叠
cg.add("n_junk3", "moss quartz fjord", layer="knowledge")       # 零重叠
res, meta = cg.search("bandana", k=10)
for r in res or []:
    print(r[0].get("id"), r[1])
print("tier:", meta.get("tier"))
```

修复前实测：

```text
tier: T3_global_scan
id=n_partial score=1.0
id=n_junk1 score=0.0
id=n_junk2 score=0.0
id=n_junk3 score=0.0
```

修复后实测：只剩 `n_partial score=1.0`；对照查询 `zzzzqqqq`（纯零相关）
仍走 T3 且返回 0 条（#89② 行为不变）；生产路径 `MdCGOS.search` 同构造
复跑结果一致（两入口共用 `_emit`）。

## 根因

T3 是唯一不设阈值的出口。结果装配单点 `MdCG._emit` 里，主结果取
`scored[:_primary_bound(k, n_neg, tier, n_real)]`；`_primary_bound` 只在
`_zero_filler_only(tier, n_real)` 为真（T3 ∧ 全库无 s>0）时把上界压到 0，
其余情形按 k 预算整段截取——排在真命中之后的 0 分行只要还在 k 内就被
装进结果，且占用 k 配额、被 `record_access` 记为一次访问。

## 改动文件

- `md_cg/mdcg.py`（+10/−1，`MdCG._emit` 装配处，`MdCG` 与 `MdCGOS` 两入口
  共用此单点）：裁切前若 `tier == TIER_GLOBAL_SCAN`，先把 `scored` 过滤为
  `s > 0`。`_n_real` 本就按 s>0 统计，负覆盖尾条预算不受影响；n_real==0
  时过滤后为空集、结果仍为空（#89② 不变）。**仅限 T3**——T0/T1/T2 的 0 分
  带质量信号（S5 降权、reach 补召回邻居），不在此裁。同步更新了 `_emit`
  上方「生效条件」契约注释一句。
- `md_cg/test_retr_s1b.py`、`md_cg/test_retr_s7.py`（各 1 条守卫断言改判）：
  两条守卫的原判据把旧行为当了前提——s1b 要求零相关的异桶节点 b1 出现在
  回退结果里（靠的正是 0 分填充）；s7 的用例有效性判据 `cands < len(r0)`
  靠把 0 分填充计入「全量命中数」才成立。均已按 #637 新口径改判并在注释
  里留痕（真命中等价性、回退审计等其余断言一字未动）。
- `package.json` / `package-lock.json`：version 0.9.11 → 0.9.12；
  repository.url 指向本 fork 仓（lock 文件仅同步其两处 version 字段，
  上游 lock 原滞后为 0.9.8）。

未动：打分函数 `_score`、`_zero_filler_only`、`_primary_bound` 本体、
负覆盖尾条逻辑、其余 tier 的装配路径、Rust 内核（二进制不在本版范围，
与 issue 边界声明一致）。

## 复验结果（修复后）

- 最小复现：`ZERO_SCORE_IDS: []`，只返真命中（见上）。
- md_cg Python 守卫（检索相关 25 个模块逐个实跑）：全绿。含
  `test_issue89_zero_score`（3/3）、`test_issue89a_score_cap_tiebreak`（14/14）、
  `test_emit_negtail_cache`（7/7）、`test_neg_tail_honesty`（46）、
  `test_neg_coverage_budget`（15/15）、`test_retr_s5`（17/17）、
  `test_retr_gates_prodpath`（9/9）、`test_retr_s1b`（34/34，改判后）、
  `test_retr_s7`（43/43，改判后）、`test_retr_s1/s3/s4/s6/s8_time/s9_edges/
  s9_entity_ctx`、`test_search_prefilter`、`test_search_bm25_default`、
  `test_mdstore_search_parity`、`test_recall_face_guards`、
  `test_issue362_neg_condition`、`test_neg_condition_hits`、
  `test_c8_search_rrf_gates`、`test_ccg_perturb`、`test_p0`、
  `test_p33_ccg_wiring`、`test_search_async`、`test_cn_recall_grams`、
  `test_recall_metadata`。
- 与基线对照（git stash 同法复跑，以下均为**改前改后一致**、非本版引入）：
  `test_rank_parity_score_mode` 因缺 Rust release 二进制 fail-closed 退出 1
  （基线同）；`scripts/cogmap_sync.py check` 与 `scripts/check_line_anchors.py`
  基线即各有 1 处未登记项、输出逐字一致（基线同）；`npm test` 248 例中
  242 通过 / 3 失败（#224、#386、hooks 空 catch 源级检查，基线同）/ 3 跳过。
- `scripts/check_unreachable.py`：1026 文件命中 0，通过。
